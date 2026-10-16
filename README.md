# ARCnet

**A transport-agnostic reliability protocol for real-time multiplayer combat games.**

ARCnet is a compact TypeScript protocol layer that adds sequencing, selective-acknowledgement retransmit, forward error correction (FEC), and adaptive congestion control on top of *any* byte transport — WebRTC DataChannels, WebSockets, raw UDP, or a Discord Activity's proxied socket. It was built for and is battle-tested in **Arcbound**, a fast-paced arena combat game by [Arcbound Interactive](https://arcboundinteractive.com), where ~33 Hz state snapshots and inputs have to survive lossy mobile networks.

> **Status:** v3.2 — in production in Arcbound. This repository is private; this document is a public overview.
>
> **Stats:** 21 commits · TypeScript (100%) · ~2,800 lines of strict-mode TS (~1,100 in the core protocol, ~200 in the input codec) · 10 test suites, 66 tests · zero runtime dependencies

---

## Novel Systems & Methods

The interesting engineering in ARCnet is not "a reliability layer exists" — it's a handful of original mechanisms designed around the realities of browser-based real-time games on unreliable links. The source is private; what follows is the concept and the result.

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

Acknowledgement state (highest-received sequence plus a selective-ACK bitfield covering a window of recent packets) rides piggyback on every outgoing packet, so when traffic flows both ways acknowledgement costs zero extra sends. A peer that only *receives* acked traffic — a game client listening to server events — has nothing to piggyback on, so it answers with **ACK-only packets**: the header alone, with no sequence of its own, never tracked or retransmitted. Older receivers read one as an empty packet and still apply its ACKs, so either side could deploy first. Retransmit timing is driven by smoothed per-channel RTT and RTT-variance estimators, with a bounded retry budget so a dead peer can't pin memory.

Ordered delivery is hardened at both ends of a stream's life: a fresh stream waits for its true first packet even when a later one arrives first, and a hole the sender has stopped retransmitting is given up after a few seconds, so the events queued behind it are released instead of stalling.

In Arcbound, the server sends match events on CRITICAL and snapshots on SEQUENCED. Inputs and fire events travel beside ARCnet as compact codec packets (section 6) rather than on RELIABLE: an early trial of fire on an acked channel gave way to plain unreliable sends.

### 3. Continuously Adaptive Snapshot Pacing

Most game netcode throttles in coarse steps ("good / bad connection"). ARCnet's congestion controller blends windowed loss rate, RTT *trend* (recent samples vs. older samples, catching congestion before loss appears), and a smoothed bandwidth estimate into a single congestion score — and maps that score onto a **continuous** recommended snapshot interval (30–90 ms, i.e. 33 Hz down to 11 Hz), with a curve shaped to ramp gently under light congestion and back off decisively under heavy congestion. A discrete GOOD / FAIR / POOR quality tier is still exposed for callers that want simple branching, but the continuous signal eliminates the visible "gear shifts" that tier-stepped throttling causes.

This is a library feature Arcbound does not switch on today: the game sends a snapshot every ~33 Hz physics tick at a fixed rate, after an early adaptive-rate trial starved clients of updates.

### 4. Honest Telemetry: RTT Percentiles and Receive-Side Loss

Every acknowledged packet yields a genuine application-level RTT sample. ARCnet keeps a rolling sample window and exposes **p50 / p95 / p99 RTT** directly from the session stats. On mobile links, average RTT looks fine while the p95 tells the real story — the percentile stats offer an honest jitter and tail-latency signal for interpolation buffers and quality decisions, with no external probing. RTT comes from whichever acked channel the peer is actually acknowledging, so a session whose acked traffic is all match events still measures it.

Loss is measured only where it can be measured honestly. The sender's loss rate counts acked channels only — a latest-wins snapshot stream is never acknowledged and would otherwise read as near-total loss. The receiver counts the sequence numbers it never saw on the sequenced channel, updated on every receive, so a receive-only peer gets a real loss figure with no ticking at all. Arcbound's client loss readout runs on these receive-side counters; its interpolation delay is sized from the game's own per-snapshot lateness samples rather than from the RTT percentiles.

### 5. Packet Coalescence

Small real-time packets are dominated by per-send transport overhead (DTLS/SCTP framing on WebRTC costs tens of bytes per send). ARCnet defines a batching envelope that fuses multiple protocol packets into one transport send, with a receive-side splitter that transparently unpacks a bundle back into individual packets — the rest of the protocol never knows batching happened. Truncated bundles degrade safely. Arcbound's server sends each snapshot together with its FEC parity as one bundle.

### 6. Compact Binary Input Codec

A companion codec replaces JSON input and fire-event messages with packed binary encodings, achieving an **80–85% size reduction** (e.g. ~100-byte JSON inputs down to under 20 bytes). Tagged first bytes let a receiver cheaply branch between binary packet types and a JSON fallback, so the codec was adoptable incrementally without a flag day.

Since v3.2 an input can also carry its **sample time**: when the reported position was true on the sender's clock. Dated by arrival alone, positions clump with network jitter and a remote ship stands still, then catches up. With the sample time, the receiver maps the sender's clock onto its own using the *fastest* recent delivery rather than an average (which wanders with every late clump), so remote ships keep their real rhythm through jitter and bursts. The field is optional and compatible both ways: old decoders read new packets, and an input without a time is the old packet byte for byte.

In Arcbound these packets ride an unreliable DataChannel beside ARCnet's framed traffic, each input resent with the previous few reports for redundancy. The server relays every reported position to the other players with its age, and the client maps the server's clock with the same fastest-delivery rule.

### 7. Genuinely Transport-Agnostic Core

ARCnet never opens a socket. The session consumes and produces `ArrayBuffer`s, so the identical protocol code runs over a WebRTC DataChannel (the UDP-like path), a WebSocket (TCP fallback with uniform stats), Node's raw UDP, or inside a **Discord Activity**, where all traffic is forced through Discord's proxy — the transport is re-pointed once and the protocol layer is untouched. The repo ships a worked Discord Activity integration (proxy URL mapping plus the embedded OAuth handshake and its non-obvious pitfalls).

In Arcbound, ARCnet carries the WebRTC DataChannel path. Every client tries that upgrade; where it cannot connect, as inside Discord's Activity sandbox, the game stays on its WebSocket fallback, which carries the same snapshots without ARCnet framing.

### 8. Snapshot Formats That Evolve per Connection (Arcbound)

The payload ARCnet carries most is Arcbound's state snapshot. Its codec belongs to the game, not the library, but it is built to change safely on a live service:

- **Capability handshake.** Each client announces, first thing on every connection, which snapshot format it decodes, and the server encodes for that client from the next broadcast. No announcement means the original format, so client and server deploy in either order and a browser tab loaded before the change keeps working.
- **v2** sends projectile ids as numbers instead of strings and positions as fixed-point integers, at a precision picked by measurement: about **36% smaller snapshots** and **25–30% less server send time** on projectile-heavy rooms.
- **v3** stops resending long-lived bouncing projectiles that every client already simulates from their spawn: a full record only when new, after a bounce, and on a slow resync, otherwise just the id. On a room with ~1,000 in the air, a snapshot fell from **~23 KB to ~2 KB**.
- **Skippable extensions.** New data rides tagged, length-prefixed sections that older decoders skip, so the server can add one before every client knows it.

Because FEC parity is as long as the longest packet it protects, every byte saved in the snapshot is saved again in its parity.

---

## Public API at a Glance

The entire protocol is driven through one small session object:

```ts
import { ARCnetSession, Channel, QualityTier } from 'arcnet';

const session = new ARCnetSession();
transport.onMessage((data) => {
  const delivered = session.receive(data);
  for (const { channel, payload } of delivered ?? []) handleMessage(channel, payload);
  for (const ack of session.takeAcks()) transport.send(ack); // ACK-only replies for receive-only peers
});
for (const buf of session.send(Channel.SEQUENCED, payload)) transport.send(buf);
setInterval(() => {
  for (const buf of session.tick(Date.now())) transport.send(buf); // retransmits + congestion eval
}, 50);
if (session.getQuality() === QualityTier.POOR) throttleNonEssentialTraffic();
```

`getStats()` exposes the full telemetry surface: smoothed RTT, `rttP50` / `rttP95` / `rttP99`, loss rate, quality tier, estimated bandwidth, goodput, the adaptive snapshot interval, retransmit and FEC-recovery counters, receive-side sequenced counters, ACK counters, and byte/packet totals.

## Quality & Testing

- **Strict TypeScript** throughout; zero runtime dependencies.
- **66 tests across 10 Vitest suites** covering round-trip framing, header validation and version rejection, FEC recovery (including out-of-order parity arrival and variable-length groups), send/retransmit behavior, loss-rate accounting, modular sequence ordering across the 16-bit wrap boundary, RTT percentile math, ACK-only packets and the retransmits they prevent, ordered-stream start and hole give-up, and the input codec's sample time (including old-decoder compatibility).
- Ships as a CommonJS build consumable from both ESM `import` and CJS `require()` code.

## Documents

- [ARCHITECTURE.md](ARCHITECTURE.md) — layer stack, packet flow, transport abstraction, FEC/ACK paths, and the congestion control loop, with diagrams.
- [SETUP.md](SETUP.md) — the toolchain and environment needed to build a comparable stack.

## License

MIT © Arcbound Interactive.
