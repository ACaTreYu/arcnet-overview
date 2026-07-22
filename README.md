# ARCnet

**A transport-agnostic reliability protocol for real-time multiplayer combat games.**

ARCnet is a compact TypeScript protocol layer that adds sequencing, selective-acknowledgement retransmit, forward error correction (FEC), and adaptive congestion control on top of *any* byte transport — WebRTC DataChannels, WebSockets, raw UDP, or a Discord Activity's proxied socket. It was built for and is battle-tested in **Arcbound**, a fast-paced arena combat game by [Arcbound Interactive](https://arcboundinteractive.com), where 60 Hz inputs and state snapshots have to survive lossy mobile networks.

> **Status:** v3 — in production. This repository is private; this document is a public overview.
>
> **Stats:** 16 commits · TypeScript (100%) · ~2,300 lines of strict-mode TS (~1,100 in the core protocol) · 7 test suites, 44 tests · zero runtime dependencies

---

## Novel Systems & Methods

The interesting engineering in ARCnet is not "a reliability layer exists" — it's a handful of original mechanisms designed around the realities of browser-based real-time games on unreliable links. Implementations are proprietary; what follows is the concept and the result.

### 1. Sequence-Tagged, Length-Prefixed XOR FEC

Classic XOR parity schemes assume fixed group boundaries and equal-length packets — both assumptions break in a real game stream. ARCnet's FEC packets explicitly carry the identities of the packets they protect, and each protected packet is folded into the parity together with its own length. The results:

- **Wrap-safe recovery.** Because group membership is carried explicitly rather than derived from sequence arithmetic, recovery works correctly across 16-bit sequence wraparound with no modular edge cases.
- **True-sequence delivery.** A recovered packet is re-injected into the channel state machine under its *original* sequence number, so ordered delivery, staleness filtering, and ACK state all behave exactly as if the packet had arrived normally.
- **Variable-length groups.** Length-prefixing inside the parity means a group can mix a 12-byte input packet with a 300-byte snapshot and still recover either one exactly.
- **Bidirectional recovery.** Parity may arrive before or after the data packets it covers; recovery triggers the moment the "exactly one missing" condition is met from either direction.

Net effect: single-packet losses on protected channels are healed in **zero round trips**, without waiting for a retransmission timeout — the difference between a visible hitch and nothing at all on a mobile connection.

### 2. Four-Channel Reliability Model with Piggybacked Selective ACKs

Instead of one reliability policy for everything, ARCnet runs four independent sequence spaces, each with delivery semantics matched to a class of game traffic:

| Channel | Semantics | Typical traffic |
|---|---|---|
| **CRITICAL** | Ordered, acked, retransmitted | Deaths, scores, match events |
| **RELIABLE** | Acked and retransmitted, unordered | Fire events, hits |
| **SEQUENCED** | Latest-wins, stale packets dropped | State snapshots, inputs |
| **VOLATILE** | Fire-and-forget | Cosmetics, particles |

Acknowledgement state (highest-received sequence plus a selective-ACK bitfield covering a window of recent packets) rides piggyback on every outgoing packet — there are no dedicated ACK packets in the steady state, so acknowledgement costs zero extra sends during normal play. Retransmit timing is driven by smoothed per-channel RTT and RTT-variance estimators, with a bounded retry budget so a dead peer can't pin memory.

### 3. Continuously Adaptive Snapshot Pacing

Most game netcode throttles in coarse steps ("good / bad connection"). ARCnet's congestion controller blends windowed loss rate, RTT *trend* (recent samples vs. older samples, catching congestion before loss appears), and a smoothed bandwidth estimate into a single congestion score — and maps that score onto a **continuous** recommended snapshot interval (30–90 ms, i.e. 33 Hz down to 11 Hz), with a curve shaped to ramp gently under light congestion and back off decisively under heavy congestion. A discrete GOOD / FAIR / POOR quality tier is still exposed for callers that want simple branching, but the continuous signal eliminates the visible "gear shifts" that tier-stepped throttling causes.

### 4. RTT Percentile Telemetry from Real ACKs

Every acknowledged packet yields a genuine application-level RTT sample. ARCnet keeps a rolling sample window and exposes **p50 / p95 / p99 RTT** directly from the session stats. On mobile links, average RTT looks fine while the p95 tells the real story — the percentile stats give the game an honest jitter and tail-latency signal to drive interpolation buffers and quality decisions, with no external probing.

### 5. Packet Coalescence

Small real-time packets are dominated by per-send transport overhead (DTLS/SCTP framing on WebRTC costs tens of bytes per send). ARCnet defines a batching envelope that fuses multiple protocol packets into one transport send, with a receive-side splitter that transparently unpacks a bundle back into individual packets — the rest of the protocol never knows batching happened. Truncated bundles degrade safely.

### 6. Compact Binary Input Codec

A companion codec replaces JSON input and fire-event messages with packed binary encodings, achieving an **80–85% size reduction** (e.g. ~100-byte JSON inputs down to under 20 bytes). Tagged first bytes let a receiver cheaply branch between binary packet types and a JSON fallback, so the codec was adoptable incrementally without a flag day.

### 7. Genuinely Transport-Agnostic Core

ARCnet never opens a socket. The session consumes and produces `ArrayBuffer`s, so the identical protocol code runs over a WebRTC DataChannel (the UDP-like path), a WebSocket (TCP fallback with uniform stats), Node's raw UDP, or inside a **Discord Activity**, where all traffic is forced through Discord's proxy — the transport is re-pointed once and the protocol layer is untouched. The repo ships a worked Discord Activity integration (proxy URL mapping plus the embedded OAuth handshake and its non-obvious pitfalls).

---

## Public API at a Glance

The entire protocol is driven through one small session object:

```ts
import { ARCnetSession, Channel, QualityTier } from 'arcnet';

const session = new ARCnetSession();
transport.onMessage((data) => {
  const delivered = session.receive(data);
  for (const { channel, payload } of delivered ?? []) handleMessage(channel, payload);
});
for (const buf of session.send(Channel.SEQUENCED, payload)) transport.send(buf);
setInterval(() => {
  for (const buf of session.tick(Date.now())) transport.send(buf); // retransmits + congestion eval
}, 50);
if (session.getQuality() === QualityTier.POOR) throttleNonEssentialTraffic();
```

`getStats()` exposes the full telemetry surface: smoothed RTT, `rttP50` / `rttP95` / `rttP99`, loss rate, quality tier, estimated bandwidth, goodput, the adaptive snapshot interval, retransmit and FEC-recovery counters, and byte/packet totals.

## Quality & Testing

- **Strict TypeScript** throughout; zero runtime dependencies.
- **44 tests across 7 Vitest suites** covering round-trip framing, header validation and version rejection, FEC recovery (including out-of-order parity arrival and variable-length groups), send/retransmit behavior, loss-rate accounting, modular sequence ordering across the 16-bit wrap boundary, and RTT percentile math.
- Ships as a CommonJS build consumable from both ESM `import` and CJS `require()` code.

## Documents

- [ARCHITECTURE.md](ARCHITECTURE.md) — layer stack, packet flow, transport abstraction, FEC/ACK paths, and the congestion control loop, with diagrams.
- [SETUP.md](SETUP.md) — the toolchain and environment needed to build a comparable stack.

## License

MIT © Arcbound Interactive.
