# P2P Design

> Status: transport layer and connection management are **implemented and in use by the
> client**; cross-network hole-punch success rate is **not yet measured** (needs two hosts
> on different NATs). Items marked *(not measured)* below are design targets, not results.

This document covers the peer-to-peer transport: why it exists, how a connection is
established, what libp2p already gives us, what we had to build ourselves, and what the
measurements say about when P2P is the wrong answer.

---

## 1. Why a P2P layer exists

Not all traffic can share one ingress. The split is decided by a single question:

> **Can the caller install our client?**

| Traffic | Caller | Channel | Why |
|---|---|---|---|
| Payment callbacks (`notify_url`) | provider data centres | public HTTPS tunnel | They will never install anything; needs valid cert + registered domain |
| Admin console | any operator browser | public HTTPS tunnel | Must open from arbitrary devices |
| Third-party SDK access | someone else's machine | public HTTPS tunnel | Only speaks `https://domain` |
| Client ↔ gateway (auth, keys, chat, long-lived browser control) | **our own client** | **P2P** | The client can be installed, and this link carries long-lived streams |

Two rules follow, and they outrank latency:

1. **Reachability beats latency.** A fast path nobody can dial is worthless.
2. **Callbacks must never traverse a public relay.** The relay pool is donated
   infrastructure with no SLA. Dropping one callback is a financial-loss and
   reconciliation problem, so it is not a risk worth taking.

That is why P2P covers exactly one link — the one where both endpoints are ours.

## 2. Layering

```
┌──────────────────────────────────────────────┐
│ App:   base_url → http://127.0.0.1:PORT      │  app never knows P2P exists
├──────────────────────────────────────────────┤
│ Bridge: local TCP ↔ Transport stream         │  one stream per local conn, no reuse
├──────────────────────────────────────────────┤
│ Selector: candidate ordering, dial, breaker  │
├──────────────────────────────────────────────┤
│ Pool: relay list, health, RTT history        │
├──────────────────────────────────────────────┤
│ Transport: libp2p (direct / DCUtR / relay)   │  exposes only net.Conn
└──────────────────────────────────────────────┘
```

The top layer only sees `net.Conn`. Swapping the transport is a deployment parameter, not a
code branch — which is what allowed the TCP transport to stay as a fallback while the P2P
path was being stabilised.

## 3. The ordering decision: relay is the floor, not the fallback

The natural implementation is *dial → wait for DCUtR upgrade → return a stream*. That is
wrong in three ways:

1. **First byte is hostage to the punch.** The punch window is 25s; a failed punch means the
   user waits 25s for the first byte.
2. **Failure is pure sunk cost.** The relay was available the whole time.
3. **A late relay connection prevents punching from starting at all.** DCUtR is triggered by
   *receiving an inbound connection through the relay*. No relay connection, no rendezvous,
   no punch.

So the order is inverted:

```
t=0     ├─ relay connections   (top-N in parallel, staggered 120ms to preserve order)
        ├─ direct dials        (public/LAN addresses from the descriptor)
        └─ background: reservation confirmation (does not block startup)
t≈RTT   └─ tunnel usable → app gets net.Conn immediately
           └─ background punch: DCUtR + direct + simultaneous-open, 4 rounds
                success → subsequent new streams go direct
                failure → nothing happens; we are already on the relay
```

**At no point does anything wait for the punch.** The measured prewarm on the same host is
4–7ms; over the public internet it is one relay RTT.

The "seamless" property, stated precisely:

| Moment | Behaviour |
|---|---|
| Connect | usable stream within one relay RTT |
| Punch fails | **no switch at all** — we were always on the relay, the connection never broke |
| Punch succeeds | **new** streams go direct; in-flight streams stay on the relay |
| All connections lost | watchdog every 1.5s, reconnect (direct first, relay as floor) |

In-stream migration is deliberately **not** implemented. Moving a live byte stream onto a
different underlying connection needs application-level ACKs and a bounded replay window, or
SSE will duplicate or drop bytes. The payoff is turning one long-lived connection from
450ms into 40ms; the risk is data corruption in the wrong direction. Correct order of work:
measure how much long-lived traffic there is and how big the direct-path gain would be,
then design session-level replay — not stream switching.

## 4. Rendezvous mechanics

| Mechanism | Who | Note |
|---|---|---|
| DCUtR (address exchange, RTT/2 alignment, simultaneous dial, 3 retries) | go-libp2p | Main punch path |
| Multi-candidate parallel dial | swarm | Under a simultaneous-connect context, all candidates are dialed at once |
| QUIC single socket | quic-go | Listen and dial share one UDP port, so the NAT mapping is reused — highest punch success rate |
| TCP simultaneous open | tcp transport | Dual-stack listen attempts it; more NAT-dependent, used as a supplement |
| Direct-first (public IP / same subnet) | this layer | Descriptor `addrs` + peerstore private addresses |
| Relay ordering by measured RTT, top-N parallel | this layer | Staggered 120ms bids, first to connect wins |
| Multi-round retry inside the punch window | this layer | 4 rounds at 0 / 300ms / 1.2s / 2.5s, three mechanisms concurrently each round |
| DCUtR single-dial timeout 10s → 4s | this layer | Retry count is worth more than per-attempt patience |
| Both roles dial | this layer | `WithSimultaneousConnect(isClient=true/false)` each tried once |
| UPnP / NAT-PMP | libp2p | Where the gateway cooperates, skip punching entirely |

**Why relay choice affects punch success:** DCUtR computes when both sides should fire from
the coordinating connection's RTT (`synTime = rtt/2`). Jittery relays misalign the two sides
and the hole closes before either side fires. "It connects" is not the criterion — "low,
stable RTT" is.

**Why both roles dial:** `WithSimultaneousConnect` only flips the direction flag on a
connection. The actual simultaneity depends on both peers emitting SYN inside the same
window, and neither can know the other's state. Trying both roles costs one extra failed
dial; missing the window costs the whole round.

## 5. libp2p capability audit (v0.51.0)

Read from source, not from documentation.

**Present:**

| Capability | Where |
|---|---|
| DCUtR (`/libp2p/dcutr`) | `p2p/protocol/holepunch` |
| Simultaneous connect | `p2p/net/swarm` (`network.WithSimultaneousConnect`) |
| AutoNAT v2 | `p2p/protocol/autonatv2`, assembled by default in `config` |
| circuit relay v2 (client + server) | signalling channel and forwarding fallback |
| NAT port mapping (UPnP / NAT-PMP / PCP) | `libp2p.NATPortMap()` |
| QUIC / TCP / WebSocket | default transports |
| observed-address reporting | `p2p/protocol/identify` |

**Absent (and not enable-by-config):**

| Missing | Impact | Note |
|---|---|---|
| Multi-STUN candidate collection | medium | Uses "the address the peer told me about"; no STUN query, one less candidate source |
| Port prediction / birthday-paradox scanning | **large** | The main tool against **symmetric NAT**; libp2p has none |
| Forced TCP simultaneous open | medium | Depends on OS and NAT; the library can initiate, not guarantee |
| Application-level TURN | small | circuit relay substitutes, but without credentials or quota |
| Multipath aggregation | small | No bandwidth striping across connections |

**Conclusion: use the library, but do not expect it to solve NAT types.**

libp2p covers the mainstream approach (DCUtR ≈ a condensed ICE: relay as signalling plus
simultaneous dial) with high engineering completeness. What it lacks — port prediction, STUN
candidates, TURN — is exactly the heavy machinery for **symmetric NAT / CGNAT**. That is
research-grade work, not a flag.

**The ceiling on punch success is set by the two NAT types, not by the library.**

## 6. What we added on top

### 6.1 Address filter (implemented)

`EnableHolePunching()` does not enable `WithAddrFilter` by default, so the host advertises
`127.0.0.1`, `::1`, `169.254.x` and docker bridges as punch candidates. Those always fail,
but they occupy slots in the simultaneous-dial window — and simultaneity is the whole
mechanism. `punchFilter` drops loopback and link-local addresses.

### 6.2 Punch measurement harness (implemented)

`-mode punch` connects through the relay, then polls for 25s for any connection that is not
`p2p-circuit`, and reports whether an upgrade happened and how long it took. Without this
number, tuning is guesswork.

First run (two hosts behind the same NAT, inbound port firewalled) reported
`no direct address within 25s` — which validates the measurement path, not a cross-network
rate.

### 6.3 Birthday-paradox candidate exchange (implemented)

This is the one mechanism libp2p genuinely cannot express, because **there is no channel to
tell the peer about a predicted port** — `HolePunchAddrs()` only accepts addresses the host
considers public. Since both ends are our code, we built the channel:

```
client                                    gateway
  │  PLAN{addrs, birthday_ip, ports}         open N UDP sockets (open first, advertise after)
  │ ─────────────────────────────────▶
  │ ◀──────── ACK{token, ports, ip}          reservation recorded
  │  GO
  │ ─────────────────────────────────▶        start responder-side punching immediately
  │ start initiator-side punching immediately
  │ ◀────── probe packets from both sides ──▶
  │      hit → run QUIC on the punched socket
  │ ─────── QUIC (five-tuple already allowed by NAT) ──▶ bridge to local HTTP
```

Design parameters, all measured trade-offs rather than round numbers:

| Parameter | Value | Reasoning |
|---|---|---|
| Sockets per side | 20 | 20 was the empirical compromise; more hits UDP session limits on consumer routers |
| Reservation TTL | 20s | PLAN→GO is one ACK round trip (≈1s tunnel RTT); 20s is slack |
| Punch budget | 12s | Beyond this the relay path is already serving the request |
| Concurrent bridges | 16 | Bounds file descriptors and NAT table entries |
| Bridge TTL | 6h | QUIC `Accept` never returns on client exit; without a cap each client leaks one UDP port |

Port **reservation** (rather than a fixed port range) exists because multiple clients punch
concurrently: a fixed range serves one. The token lets the GO phase recover exactly the
sockets opened at PLAN, and the janitor reclaims them when a client disappears between
PLAN and GO.

This is only useful against NATs that allocate ports predictably. On symmetric NAT with
randomised ports the birthday probability collapses — see §7.

## 7. Measured reality: NAT classes decide everything

Same socket, four STUN targets, mapping observed each time:

| Environment | Observed | Class |
|---|---|---|
| Home host | `171.107.134.129:51136` for every target | **EIM** (endpoint-independent mapping) |
| Cloud A (container) | one IP, **4 different ports** | **symmetric** |
| Cloud B (container) | **3 different public IPs**, 4 different ports | **multi-egress + symmetric** |

Cross-cloud punch attempt, both sides registering all four candidates and probing every
200ms for 60s:

```
== result ==   sent 1200 packets, received 0
== result ==   sent 1200 packets, received 0
```

Not "high latency, some loss" — nothing came back in either direction. Under symmetric NAT
the peer cannot know which port to aim at, so classic UDP hole punching is dead at the
theory level, not the implementation level.

Three conclusions that changed the architecture:

1. **A node's NAT class determines its topology role.** A host that can only dial out
   (HTTP/WS inbound allowed, no UDP hole punching, no global IPv6, no `/dev/net/tun`, no
   `CAP_NET_ADMIN`) is a **worker**, not a peer: dial the anchor, take work, return results,
   accept inbound only over HTTP/WS plus relay.
2. **Where punching is impossible, stop budgeting for it.** Relay throughput measured
   1.99 MB/s and 6.02 MB/s between the two clouds — enough for control and streaming text,
   not for large payloads.
3. **IPv6 is not a universal escape hatch.** Both environments had no global IPv6 address and
   no v6 route, so v6 punching was excluded at the kernel level.

The same investigation reversed its own conclusion three times in one day before landing on
"symmetric NAT" as the root cause. The reversals are kept in the record because the sequence
— hypothesis, measurement, retraction — is the actual output.

## 8. Failure budget and fallback rules

```
candidates = descriptor.relays (21) ∪ direct addresses learned via peerstore
order      = (reachable ? 1:0, RTT_p50, -historical success rate)

dial:
  1. direct address first, 800ms timeout
  2. top-3 relays in parallel, first to connect wins (never serially)
  3. max 2 attempts per candidate; total budget 3s
  4. all failed → fall back to public tunnel, record p2p_fallback, keep probing in background
  5. any relay failing 3 times consecutively → 5 min cooldown, removed from candidate set
```

Why the top 3 in parallel: relay RTTs cluster in 457–880ms, so serial waiting compounds the
worst case into ~2.7s while parallel costs one RTT.

Why an explicit budget: "it felt broken so we switched" is not a criterion. Every fallback
emits an event with a reason, so the ratio is a number, not an opinion.

## 9. Long-lived connection hygiene

| Mechanism | Implementation | Failure it prevents |
|---|---|---|
| Stable identity | Ed25519 key on disk (`-identity`), peerID derived from public key | Otherwise every restart must ask a central service "where am I", and the autonomy is nominal |
| Static descriptor | `gw.json` = `{peer_id, relays[]}`, atomic write | Client needs no online discovery |
| Parallel reservation at startup | Reserve with all relays concurrently | Serial: 21 × 15s timeout = 5 minutes worst case |
| Heartbeat | 20s application-level ping, libp2p ping as backstop | Relays and public edges both reap idle sessions |
| Half-open detection | 45s with no inbound byte → declare half-open → rebuild | A half-open TCP accepts writes but never returns reads; write-only health checks lie |
| Reconnect backoff | 1s / 2s / 4s / 8s / 15s cap, ±20% jitter | Fixed intervals synchronise into a thundering herd when a relay dies |
| Stream isolation | One stream per local connection; a broken stream closes only itself | HTTP keep-alive and WebSocket need independent connection semantics |
| Idempotency | Dial may retry; gateway requests carry idempotency keys | A retry must not become a duplicate charge |

Reservation is deliberately **off the startup critical path**: it determines whether others
can dial *in*, not whether we can dial *out*. Startup only waits for the relay connection.

## 10. Security

- **Transport:** libp2p Noise/TLS end-to-end; the relay sees ciphertext only.
- **Stream level:** first line is `AUTH <token>`, obtained at login and never written to
  `gw.json` — the descriptor is designed to be copyable.
- **Descriptor integrity:** Ed25519 signature over `gw.json` *(not implemented)*. Without it,
  a copied descriptor can be edited to point at a malicious relay.
- **Authorization boundary:** the gateway still authenticates API keys. P2P is a channel, not
  an authentication mechanism.

## 11. Observability

| Metric | Purpose |
|---|---|
| `p2p_dial_total{result}` | connect success/failure counts |
| `p2p_relay_rtt_ms{relay}` | p50/p95 per relay |
| `p2p_holepunch_total{result}` | punch success rate — the number that decides whether P2P keeps getting budget |
| `p2p_fallback_total{reason}` | fallbacks by reason |
| `p2p_session_age_seconds` | long-lived connection lifetime distribution |
| `p2p_stream_errors_total{kind}` | stream-level error classes |

The punch success rate is the budget gate: below 50%, the correct response is to stop
tuning and move spend to self-hosted ingress.

## 12. Roadmap

| Stage | Content | Done criterion |
|---|---|---|
| P0 | Heartbeat + tunnel wired into the client | implemented; ≥1h without reconnect *(cross-network verification pending)* |
| P1 | Parallel selection + breaker + metrics | kill any single relay, business unaffected |
| P2 | Punch rate measurement and tuning (TCP/QUIC dual stack) | ≥70%, direct RTT p50 <40ms |
| P3 | Descriptor signature + hot relay updates | change relays without restarting clients |

## 13. Anti-patterns

- Routing payment callbacks through the public relay — no SLA.
- Carrying attachments or large responses over the relay — measured 200 KB/s, 5 MB takes 25s.
- Probing 21 relays serially — worst case is 21 stacked timeouts.
- Using "the connection still exists" as a health signal — half-open writes succeed.
- Putting tokens in `gw.json` — descriptors get copied and pasted into chat windows.
- Shipping long-lived connections without a heartbeat — every channel reaps idle sessions.

---

## 14. Why this transfers to node operations

The primitives here are the same ones a libp2p-based chain node spends its life on: peer
identity derived from a key, rendezvous through a relay, NAT classification deciding whether
a node can be dialled or only dials out, reservation and liveness, and a metric that tells
you when to stop optimising one path and change topology instead.

Three transferable judgements, in decreasing order of how much they matter:

1. **Determine the NAT class before designing the topology.** The classification (EIM vs
   symmetric vs multi-egress) predicts whether a host can ever be an inbound peer. Discovering
   this after building a punch pipeline wastes the whole pipeline.
2. **Keep the guaranteed path as the floor, not the fallback.** Anything that puts first-byte
   latency behind a probabilistic mechanism will be felt as an outage, not as a slow path.
3. **Keep a record of retracted conclusions.** The three reversals in §7 are worth more in an
   interview than any success rate, because they show the measurement loop closing.

Differences worth stating plainly rather than glossing over: this system uses libp2p's
circuit relay v2 and DCUtR, not Ethereum's discv5/devp2p, and it has no chain state, no
consensus participation and no peer scoring based on protocol compliance. The overlap is
transport and peer management, not consensus.
