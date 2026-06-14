# iron-bridge — Design

A small, transport-agnostic library for connecting a web application with the
host that embeds it. Hosts can be a VS Code extension, another web application
(iframe), or an IntelliJ IDEA plugin (JCEF). It is inspired by Apache KIE
Tools' `envelope-bus`, but deliberately reduced to a single core idea.

## Goals

- **Extremely simplified.** One communication primitive, a tiny wire protocol,
  no incidental complexity.
- **Native first.** The core knows nothing about React or any UI framework. It
  only knows how to move messages over a pluggable transport. Framework
  niceties are shipped as separate, optional packages.
- **Clear direction from the start.** Whether a call goes host → app or
  app → host is encoded in the type system, not discovered at runtime.
- **Per-host packages** under a single scope, in a Yarn workspaces monorepo.

## Background: what we keep from KIE Tools and what we drop

KIE Tools' `envelope-bus` centers on one wire type (`EnvelopeBusMessage`) and a
message manager that builds typed proxies so callers write
`api.requests.foo(...)` (a Promise) or `api.notifications.bar.send(...)`
(fire-and-forget). It supports **seven** message purposes.

We keep the typed-proxy ergonomics and drop the rest:

| KIE Tools concept | Decision |
| --- | --- |
| `REQUEST` / `RESPONSE` | Kept, generalized into a subscription (see below). |
| `NOTIFICATION` + `*_SUBSCRIPTION` / `*_UNSUBSCRIPTION` | Folded into the same subscription primitive. |
| `SHARED_VALUE_*` (reactive state sync) | **Dropped for now.** May return later, built on top, once we have concrete use cases. |
| `targetEnvelopeId` / `targetEnvelopeServerId` / `directSender` (multi-envelope routing) | **Dropped.** One bridge is point-to-point. Optional `bridgeId` namespacing is the only multiplexing under consideration. |
| 60s init polling handshake | **Dropped.** Replaced by a `ready` signal + outbound buffering. |

## Core idea: everything is a subscription

Instead of separate "request" and "notification" mechanisms, every method on
every contract returns an **Observable**. This unifies the cases:

- A request/response is an Observable that **emits once and completes**.
- An event stream (theme changes, file watching, content changes) is an
  Observable that **emits many times**.
- Fire-and-forget is an Observable that is simply never subscribed to, or one
  that completes immediately.

The single primitive is *subscribe to a method on the other side*. A key
benefit falls out for free: **teardown propagates**. When one side unsubscribes
from a long-lived stream, the other side's source is torn down too (e.g. the
host stops watching files). The Promise model cannot express this, and KIE
Tools only achieves it through manual subscription/unsubscription purposes.

## Wire protocol (the common channel)

One message type; the five purposes are the Observer contract serialized:

```ts
type BridgePurpose = "subscribe" | "next" | "error" | "complete" | "unsubscribe";

interface BridgeMessage {
  bridgeId?: string;          // namespace, if multiple bridges share one transport
  purpose: BridgePurpose;
  id: string;                 // subscription correlation id
  type?: string;              // method name — only present on "subscribe"
  data?: unknown;             // args on "subscribe"; emitted value on "next"
  error?: SerializedError;    // present on "error"
}
```

| Purpose | Direction | Meaning |
| --- | --- | --- |
| `subscribe` | caller → implementer | Invoke `type` with `data` as args; begin the subscription `id`. |
| `next` | implementer → caller | An emitted value for subscription `id`. |
| `error` | implementer → caller | Terminal error for subscription `id`. |
| `complete` | implementer → caller | Terminal completion for subscription `id`. |
| `unsubscribe` | caller → implementer | Tear down subscription `id`. |

`id` generalizes KIE Tools' `requestId` to any subscription.

## Observable contract (zero-dep, RxJS-interop-compatible)

The core ships a tiny (~50 line) implementation of the Observable contract and
tags it with `Symbol.observable`, so it is **structurally compatible with
RxJS** without depending on it. Consumers who use RxJS get the full operator
pipeline via `from(...)`; consumers who do not get plain `.subscribe()`.

```ts
interface Observer<T> { next(v: T): void; error(e: unknown): void; complete(): void; }
interface Unsubscribable { unsubscribe(): void; }

interface Observable<T> {
  subscribe(observer: Partial<Observer<T>> | ((v: T) => void)): Unsubscribable;
  [Symbol.observable](): Observable<T>;   // RxJS `from()` accepts this natively
}
```

```ts
import { from, firstValueFrom } from "rxjs";

firstValueFrom(bridge.requests.readFile("a.yaml"));            // Promise ergonomics
from(bridge.requests.watchFiles("**/*.yaml")).pipe(/* ops */); // full RxJS power
```

## Contracts and direction

Direction is encoded as two typed contracts. Both sides extend a built-in base
contract that guarantees minimal functionality (handshake + liveness) and where
the lifecycle lives, so consumers never wire it themselves.

```ts
// Lifecycle guaranteed by core.
interface BridgeBase {
  ready(): Observable<void>;   // completes once the far side is wired
  ping(): Observable<void>;
}

// Implemented by the APP; the host subscribes to these (host → app).
interface CommandApi extends BridgeBase {
  setTheme(theme: Theme): Observable<void>;
  contentChanges(): Observable<string>;
}

// Implemented by the HOST; the app subscribes to these (app → host).
interface RequestApi extends BridgeBase {
  readFile(path: string): Observable<Uint8Array>;
  watchFiles(glob: string): Observable<FileEvent>;
}
```

**Every contract method must return an `Observable`** — maximally uniform and
predictable. (We may add a lenient `T | Promise<T> | Observable<T>` input
normalization later if the always-Observable form proves verbose.)

From the perspective of the web app exposed through the bridge:

- **Commands** are what the host sends to the app (the app implements
  `CommandApi`).
- **Requests** are what the app sends to the host (the host implements
  `RequestApi`).

## The factory

`createBridge` takes a transport and a local implementation, and returns a
typed proxy of the *remote* side.

```ts
function createBridge<Local extends ApiDefinition<Local>,
                      Remote extends ApiDefinition<Remote>>(
  transport: Transport,
  impl: Local,
): Bridge<Remote>;
```

`Bridge` exposes a generic `.remote` proxy; the host packages alias it to
`.commands` / `.requests` so each side reads naturally:

```ts
// App side — implements CommandApi, calls the host's RequestApi.
const bridge = createBridge<CommandApi, RequestApi>(transport, appImpl);
bridge.requests.readFile("a.yaml").subscribe(bytes => { /* ... */ });
bridge.requests.watchFiles("**/*.yaml").subscribe(ev => { /* ... */ });
// unsubscribing stops the host-side watcher

// Host side — implements RequestApi, calls the app's CommandApi.
const bridge = createBridge<RequestApi, CommandApi>(transport, hostImpl);
bridge.commands.contentChanges().subscribe(yaml => { /* ... */ });
```

Same engine on both sides; only the generics are swapped.

## The transport seam

The only thing every host has in common is a duplex pipe. That is the entire
abstraction each host package implements:

```ts
interface Transport {
  send(message: BridgeMessage): void;
  receive(handler: (message: BridgeMessage) => void): () => void;  // returns dispose
}
```

It reduces cleanly onto each target:

| Host | App side (in the webview) | Host side |
| --- | --- | --- |
| **VS Code** | `acquireVsCodeApi().postMessage` + `window.addEventListener("message")` | `webview.postMessage` + `webview.onDidReceiveMessage` |
| **Web (iframe)** | `parent.postMessage(m, origin)` + `addEventListener("message")` | `iframe.contentWindow.postMessage` + `addEventListener("message")` (origin-checked) |
| **IntelliJ (JCEF)** | `window.cefQuery({ ... })` + injected `window.__ironBridgeReceive` | Java handler + `executeJavaScript(...)` |

Core handles everything above the pipe: correlation, the base-contract
handshake, and teardown propagation.

## Handshake

No polling. The app emits `ready` once its transport is wired; the host
**buffers outbound `subscribe` messages until `ready` arrives**, then flushes
them. This replaces KIE Tools' 60-second init polling with a few lines and no
timeouts.

## Packages

A Yarn workspaces monorepo (Yarn ≥ 4.14, `workspaces: ["packages/*"]`,
lerna-lite for versioning/publishing — the same shape Kaoto uses).

```
packages/
  core      → @iron-bridge/core       (wire types, Observable, createBridge, Transport — zero deps)
  vscode    → @iron-bridge/vscode     (host + webview transports, .commands/.requests aliases)
  web       → @iron-bridge/web        (iframe/window transport)
  intellij  → @iron-bridge/intellij   (JCEF transport)
  react     → @iron-bridge/react      (useBridge / useObservable; optional, depends on core)
```

## Open questions

- **`ApiDefinition` typing.** How strictly to constrain "every method returns an
  `Observable`" at the type level while keeping inference ergonomic.
- **`bridgeId` multiplexing.** Whether multiple bridges sharing one transport is
  worth supporting in v1, or cut until needed.
- **Error serialization.** Shape of `SerializedError` and how much of a thrown
  error survives the wire.
