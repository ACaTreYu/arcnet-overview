# SETUP — Toolchain & Environment

This describes the toolchain and environment used to build ARCnet, and what you would need to build a comparable protocol stack. It is intentionally generic — it is not a dump of the private repository's configuration.

## System Dependencies

| Dependency | Version | Purpose |
|---|---|---|
| Node.js | 20 LTS or newer (22 recommended) | Build tooling, test runner, server-side/raw-UDP usage |
| npm | Bundled with Node | Dependency and script management |
| TypeScript | 5.7+ | Compiler — the only build tool; no bundler is required |
| Vitest | 4.x | Test runner (dev-only) |
| @types/node | Matching your Node major | Type definitions for Node-side usage (dev-only) |

That's the entire toolchain. The published library has **zero runtime dependencies** — the protocol core uses only standard `ArrayBuffer` / `DataView` / `Uint8Array` APIs available in every modern browser and Node.

## Compiler Configuration (in spirit)

The project compiles with settings along these lines; any equivalent setup works:

- **Target:** ES2020 — modern enough for the platform APIs used, old enough for wide browser coverage.
- **Module format:** CommonJS output. A CJS build is consumable from both ESM `import` and CJS `require()` callers, which matters when the same package feeds a bundled browser client and a plain Node game server.
- **`strict: true`** — non-negotiable for protocol code; sequence math and buffer offsets are exactly where implicit `any` hides bugs.
- **Declarations + source maps** emitted for consumers.
- Compile `src/` only; keep `tests/` and `examples/` out of the published build.

## Runtime Environments

The core must run unmodified in three environments, which constrains the code style:

1. **Browser** — no Node built-ins in the protocol core (no `Buffer` dependence in code paths; type guards accept `Buffer | ArrayBuffer | Uint8Array` where inputs may vary).
2. **Node server** — the CJS build is `require()`-able directly; raw UDP via `dgram` works as a transport.
3. **Discord Activity iframe** — same browser build, but all network traffic must traverse Discord's proxy. This requires no protocol changes, only transport-level URL mapping done before the first network call, plus the Discord embedded-app SDK (`@discord/embedded-app-sdk`) — a dependency of the *example integration only*, never of the library.

## External Accounts / Services (only for the Discord path)

Building the Discord Activity integration additionally requires:

- A **Discord application** in the Discord Developer Portal with:
  - the Activity enabled,
  - **URL Mappings** configured (a proxy prefix pointing at the host that serves your game WebSocket and HTTP API),
  - OAuth2 configured; the application ID is public, the **client secret must live server-side only** (supplied via environment variable to the token-exchange endpoint — never committed, never shipped to the client).
- A game server reachable over **HTTPS/WSS** (the proxy will not forward to plain HTTP), with an endpoint that exchanges the SDK's one-time OAuth code for an access token.

No other external services, accounts, or keys are involved anywhere in the stack.

## Project Layout Pattern

A comparable stack fits comfortably in a small, flat layout:

```
src/            protocol core + companion codec (the only published code)
tests/          Vitest suites, one per protocol concern
examples/       transport integrations (not built, not published)
```

Scripts to expect: `build` (tsc), `typecheck` (tsc --noEmit), `test` (vitest run), `clean`.

## Testing Approach

Reliability protocols live or die on edge cases, so the suites are organized by concern rather than by file:

- **Framing round-trip** — encode/decode symmetry, magic and version rejection, short-buffer handling.
- **FEC** — recovery with parity arriving before and after data, variable-length groups, sequence-wrap groups, corrupt-parity rejection, cache eviction under sustained loss.
- **Retransmit** — pending tracking, ACK clearing (head ACK and bitfield), timeout behavior, retry budget.
- **Loss accounting** — the loss-rate denominator counts *cleared pending packets*, not raw receives; tests pin this down because it is easy to regress.
- **Sequence math** — modular comparison and ordering across the 16-bit wrap boundary, including the ordered-queue flush.
- **Percentiles** — nearest-rank behavior at small sample counts and boundary p-values.

Simulating a lossy link in tests requires no network at all: because the session's inputs and outputs are plain buffers, a "network" is just two sessions and an array you selectively drop, reorder, or delay packets from. This is the single biggest payoff of the transport-agnostic design — the entire protocol is unit-testable at full speed with deterministic loss patterns.

## Building Something Similar — Checklist

1. Pick a fixed, versioned header early; reject unknown versions from day one.
2. Keep the core synchronous and socket-free — buffers in, buffers out. Test harnesses and new transports become trivial.
3. Use modular (wrap-aware) sequence comparison everywhere; never compare sequence numbers with `<`.
4. Piggyback ACK state on every packet before considering dedicated ACK traffic.
5. Bound every cache and retry loop — pending maps, FEC stores, ordered-delivery queues — so a hostile or dead peer cannot grow memory.
6. Measure RTT from real acknowledgements and keep percentiles, not just averages.
7. Make congestion response continuous, not stepped, and derive it from trends as well as levels.
