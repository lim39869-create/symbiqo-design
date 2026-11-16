# Symbiqo — Design Notes

> A **design-only** repository. The product implementation is private and is not distributed here.

Symbiqo turns a capability that exists **only behind a web UI** into a standards-compliant
API (OpenAI `/v1/chat/completions`, Anthropic `/v1/messages`).

It is not page scraping. It drives a real browser session, intercepts incremental rendering at
the network layer, and reassembles it into a canonical stream. The adversary is a constantly
changing front-end state machine, not a documented endpoint.

There is also a peer-to-peer transport under the client, because the client ↔ gateway link
must not depend on a public edge.

And underneath all of it there is a host that has been running the whole stack for two months
without a single container restart — documented in [`OPERATIONS.md`](OPERATIONS.md), because a
design that has never been operated is a hypothesis.

## Why this repository is public

Two roles are relevant to the material here, and neither is proven by line counts.

**Collection engineering.** What matters is not how many sites were crawled. It is whether the
system can measure itself (a denominator, a definition, a result that could be falsified),
whether it can attribute a failure to our own criterion versus target-side resistance, and
whether it knows which findings must not be written down.

**Node operations.** Some of the same primitives decide whether a node can be dialled at all:
peer identity derived from a key, relay rendezvous, NAT classification, reservation and
liveness, and a metric that says when to stop optimising a path and change topology instead.

## Contents

| File | Content | State |
|---|---|---|
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Three-tier topology, event stream contract, extension points, failure modes and what they cost | implemented |
| [`P2P.md`](P2P.md) | P2P transport: relay-as-floor ordering, libp2p capability audit, custom birthday-paradox rendezvous, measured NAT classes, fallback rules | implemented |
| [`METRICS.md`](METRICS.md) | 17.8 days of production data: success rate, per-target breakdown, latency percentiles, failure taxonomy | measured |
| [`OPERATIONS.md`](OPERATIONS.md) | A real host: 16 containers, 66 days uptime, zero restarts, deployment timeline, on-chain agent history, and the one thing that will break first | measured |
| [`BOXPROXY.md`](BOXPROXY.md) | Runtime substrate for disposable containers: three-layer control plane, single-port addressing, keepalive across three platforms, six measured capability ceilings | implemented, **P2P bridging not built** |
| [`CAPTCHA.md`](CAPTCHA.md) | Verifying an answer that cannot be read off the image: tolerance bound, measurement baseline that was polluted, differential localization, and the criteria a small model must meet | research, **model not trained** |
| [`INGRESS-AUTH.md`](INGRESS-AUTH.md) | Admission control for bridged container ingress: three-layer design, per-environment trust boundaries, refusal list, implementation order | **design only, not implemented** |
| [`DISCLOSURE.md`](DISCLOSURE.md) | The boundary: what is published, what is redacted, what is never published | — |
| [`README.zh-CN.md`](README.zh-CN.md) | Chinese versions exist for `README`, `ARCHITECTURE`, `METRICS`, `DISCLOSURE` (`*.zh-CN.md`); the newer files are English only | — |

The `State` column is not decoration. Two of these files describe work that does not exist yet,
and the difference between a measured result and an intended one is the most useful thing a
design note can communicate.

## Four conclusions worth more than the code

**1. Anti-automation is not the main cost.** Over 17.8 days the dominant failures were
end-of-generation detection (76) and page-structure self-healing (67). Anti-automation blocks
accounted for 14.8% of all failures. Spend on criteria and self-healing before spending on
evasion.

**2. Success rate must be split by target.** One engine, six targets, 94.0% down to 47.6%.
Averaging them merges "our defect" with "their block" into one number, after which nothing can
be prioritised.

**3. A guaranteed path must be the floor, not the fallback.** Putting first-byte latency behind
a probabilistic mechanism (hole punching) is experienced as an outage, not as a slow path. In
the P2P layer the relay is established first and punching runs in parallel only to make things
faster.

**4. Explicit failure beats silent degradation.** A declared missing capability with a rejected
request is cheaper than a silently dropped payload, because silent loss is invisible until the
data is already wrong.

## Status

Design notes are a first draft. Implementation is private and running in production. Some
items are explicitly marked as design intent rather than measurement — `P2P.md` states which,
because a design document that hides the difference between "measured" and "planned" is worse
than no document.
