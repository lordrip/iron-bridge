# iron-bridge — Design

A small, transport-agnostic library for connecting a **guest** web application
with the **host** that embeds it. Hosts can be a VS Code extension, another web
application (iframe), or an IntelliJ IDEA plugin (JCEF). It is inspired by
Apache KIE Tools' `envelope-bus`, but deliberately reduced to a small,
predictable core.

> **Status.** POV 1 (the host) and POV 2 (the guest) are settled and described
> below. The **bus** (wire protocol, multiplexing, sync) and a few cross-cutting
> concerns are deliberately deferred to the end and still under design.

## Goals

- **Extremely simplified for consumers.** The surface a developer touches stays
  tiny and predictable; incidental complexity lives in the core, not the API.
- **Native first.** The core knows nothing about React or any UI framework. It
  only knows how to move messages over a pluggable transport. Framework
  niceties ship as separate, optional packages.
- **Clear direction from the start.** Whether a call goes host → guest or
  guest → host is encoded in the type system, not discovered at runtime.
- **Per-host packages** under a single scope, in a Yarn workspaces monorepo.

## Vocabulary

So both sides talk the same language. Two **independent** axes describe every
method: its **direction** and its **kind**.

| Term | Meaning |
| --- | --- |
| **Host** | The embedder: VS Code extension, IntelliJ plugin, or parent web page. |
| **Guest** | The embedded web app — the webview / iframe / JCEF content. |
| **Contract** | A typed interface listing the methods one side implements. |
| **Handler** | A concrete implementation of a contract method. |
| **Bridge** | The per-side object from `createBridge`: exposes the remote side as a typed proxy, and routes incoming calls to local handlers. |
| **Command** | *Direction:* host → guest. The host sends it; the guest implements the handler. |
| **Request** | *Direction:* guest → host. The guest sends it; the host implements the handler. |
| **One-way message** | *Kind:* a method that returns `void`. Fire-and-forget, sent eagerly, no response. Exists in **both** directions. |
| **Data-returning call** | *Kind:* a method that returns a value or a stream. Surface **deferred** (see the bus section). |

**The two axes are orthogonal.** "Command" / "request" name only *direction*,
not whether a value comes back:

- `setTheme(theme): void` — a one-way **command** (host → guest).
- `sendUpdate(text): void` — a one-way **request** (guest → host).
- `readFile(path): <data>` — a data-returning **request** (guest → host).

This is the key correction over earlier drafts, which conflated "command" with
"one-way" and "request" with "has a response."

## POV 1 — The Host

The host has two jobs.

**1. Send commands to the guest.** One-way commands return `void`; calling one
sends immediately (eager) and returns nothing:

```ts
bridge.commands.setTheme("dark");   // sent now, returns void
bridge.commands.setContent(text);   // sent now, returns void
```

No ACK and no `.subscribe()` ceremony — fire-and-forget. (Delivery
confirmation is intentionally omitted for now; it can be added later if a real
need appears.)

**2. Handle requests from the guest.** The host registers **one exhaustive
object or class instance** implementing the request contract:

```ts
class HostHandlers implements RequestApi {
  saveContent(text: string): void { writeToDisk(text); }   // one-way request
  readFile(path: string) { /* data-returning — surface TBD */ }
}

const bridge = createBridge<RequestApi, CommandApi>(transport, new HostHandlers());
```

- The compiler forces **every** contract method to be present — a missing
  handler is a compile error, never a silent runtime gap.
- A plain object literal works identically (it is structurally the same to
  TypeScript); a class is useful when handlers need shared state or
  constructor-injected dependencies.
- Liveness (`ping`) is **injected by core** both ways — the host never writes
  it. Guest **readiness** arrives as a core-managed guest → host signal: the
  host's bridge flushes its outbound buffer and exposes readiness automatically,
  with nothing to implement (see Handshake).

## POV 2 — The Guest

The mirror image of the host.

**1. Handle commands from the host.** Register one exhaustive object/class
implementing the command contract:

```ts
class GuestHandlers implements CommandApi {
  setTheme(theme: Theme): void { document.body.dataset.theme = theme; }
  setContent(text: string): void { editor.setValue(text); }
}

const bridge = createBridge<CommandApi, RequestApi>(transport, new GuestHandlers());
```

**2. Send requests to the host.** One-way requests are eager and return `void`,
exactly like commands in the other direction:

```ts
bridge.requests.saveContent(editor.getValue());   // sent now, returns void
// data-returning requests (e.g. readFile) — surface TBD
```

Same engine on both sides; only the generics swap.

## Contracts and direction

Direction is encoded as two typed contracts. Both sides extend a built-in base
contract that core implements and injects, so consumers never wire lifecycle
themselves.

```ts
// Liveness — bidirectional, implemented and injected by core.
// Either side may ping the other.
interface BridgeBase {
  ping(): /* liveness — final shape with the bus */;
}

// Implemented by the GUEST; the host sends these (host → guest).
interface CommandApi extends BridgeBase {
  setTheme(theme: Theme): void;
  setContent(text: string): void;
}

// Implemented by the HOST; the guest sends these (guest → host).
interface RequestApi extends BridgeBase {
  saveContent(text: string): void;        // one-way request
  readFile(path: string): /* data — TBD */;
}
```

**Readiness is not in the shared base** — it is directional. Only the guest
knows when its transport/DOM is wired, and the host has no other way to tell, so
`ready` is a **core-managed guest → host signal** (conceptually a one-way
request). The guest's bridge sends it automatically; the host's bridge consumes
it. It is excluded from the exhaustive-registration check — consumers never
implement or call it.

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

`Bridge` exposes a generic `.remote` proxy; the per-host packages alias it to
`.commands` (host side) / `.requests` (guest side) so each side reads naturally.
To keep consumers from ever getting the generic order wrong, the per-host
packages should also expose direction-specific factories
(`createHostBridge` / `createGuestBridge`) with the generics baked in.

## Behavior settled so far

- **One-way messages** (return `void`) are **eager, fire-and-forget**, in both
  directions. No ACK (can be added later).
- **Registration** is an **exhaustive object or class instance**; the compiler
  enforces completeness. Core-managed lifecycle — `ping` (bidirectional) and
  `ready` (guest → host) — is injected and excluded from that completeness check.
- **Direction** lives in the type system via two contracts.

## Background: what we keep from KIE Tools and what we drop

KIE Tools' `envelope-bus` centers on one wire type (`EnvelopeBusMessage`) and a
message manager that builds typed proxies so callers write `api.requests.foo(...)`
or `api.notifications.bar.send(...)`. It supports **seven** message purposes.

We keep the typed-proxy ergonomics and reduce the rest:

| KIE Tools concept | Decision |
| --- | --- |
| `NOTIFICATION` (fire-and-forget) | Kept as the **one-way message** (`void` return), in both directions. |
| `REQUEST` / `RESPONSE` | Kept as **data-returning calls**; consumer surface deferred (see bus). |
| `*_SUBSCRIPTION` / `*_UNSUBSCRIPTION` | Folded into the data-returning/stream surface — deferred. |
| `SHARED_VALUE_*` (reactive state sync) | **Dropped for now.** May return later, built on top, once we have concrete use cases. |
| multi-envelope routing (`targetEnvelopeId`, …) | Reduced. One bridge is point-to-point; multiplexing for multiple guests is a **deferred** bus concern. |
| 60s init polling handshake | **Dropped.** Replaced by a `ready` signal + outbound buffering (see Handshake). |

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

| Host | Guest side (in the webview) | Host side |
| --- | --- | --- |
| **VS Code** | `acquireVsCodeApi().postMessage` + `window.addEventListener("message")` | `webview.postMessage` + `webview.onDidReceiveMessage` |
| **Web (iframe)** | `parent.postMessage(m, origin)` + `addEventListener("message")` | `iframe.contentWindow.postMessage` + `addEventListener("message")` (origin-checked) |
| **IntelliJ (JCEF)** | `window.cefQuery({ ... })` + injected `window.__ironBridgeReceive` | Java handler + `executeJavaScript(...)` |

Core handles everything above the pipe: correlation, the base-contract
handshake, and (for data-returning calls) teardown propagation.

## Handshake

No polling. Readiness is modeled as a **guest → host** signal (`ready`), because
only the guest knows when its transport/DOM is wired and the host has no other
way to tell. The guest's bridge sends `ready` automatically once wired; the host
**buffers outbound messages until it arrives**, then flushes them. This replaces
KIE Tools' 60-second init polling with a few lines and no timeouts. `ready` is
core-managed at both ends — consumers neither implement nor call it.

## Packages

A Yarn workspaces monorepo (Yarn ≥ 4.14, `workspaces: ["packages/*"]`,
lerna-lite for versioning/publishing — the same shape Kaoto uses).

```
packages/
  core           → @iron-bridge/core       (wire types, createBridge, Transport — zero deps)
  vscode         → @iron-bridge/vscode     (host + webview transports, direction-specific factories)
  web            → @iron-bridge/web        (iframe/window transport)
  podman desktop → @iron-bridge/web        (iframe/window transport)
  intellij       → @iron-bridge/intellij   (JCEF transport)
  react          → @iron-bridge/react      (useBridge / useRequest; optional, depends on core)
```

## Deferred — the bus (POV 3) and cross-cutting concerns

Everything below is **not yet decided** and is the focus of the next round.

- **Consumer surface for data-returning calls.** `Promise<T>` for one-shots vs
  `Observable<T>` for streams vs a single uniform `Observable<T>` everywhere.
  (Current lean: polymorphic by return type, while the *wire* stays a uniform
  subscription — not decided.) A zero-dep, RxJS-interop Observable
  (`Symbol.observable`) is a candidate for the stream case.
- **Wire protocol.** A single `BridgeMessage` with subscription-style purposes
  (`subscribe` / `next` / `error` / `complete` / `unsubscribe`) for
  data/streams, plus eager send for one-way messages. The final shape depends on
  the surface decision above.
- **Multiplexing.** A single host (e.g. VS Code) embeds **multiple guests at
  once** — one per open file/tab. How bridges are namespaced and addressed
  (`bridgeId`) over a shared transport.
- **Throttle / debounce.** Coalescing guest edit bursts (typing) before they
  cross the bus.
- **Sync / conflict.** The host edits the underlying buffer (its text model)
  while the guest edits too — who wins, and how echo loops are avoided.
- **Error serialization.** The shape of `SerializedError` and how much of a
  thrown error survives the wire.
- **`ApiDefinition` typing.** How strictly to constrain method shapes
  (`void` vs data-returning) at the type level while keeping inference ergonomic.
