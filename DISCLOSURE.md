# Disclosure Boundary

This file defines **how far this design repository goes**. Three zones, each mapped to real
directories in the private implementation. The boundary is published so it can be reviewed,
rather than explained after the fact.

---

## Zone 1 — publishable as-is

Nothing here is reusable to rebuild the product or to reach a third party. This is the direct
evidence of design judgement.

| Content | Why it is safe |
|---|---|
| Three-tier topology (caller → daemon → extension) | A generic architectural pattern; cannot be used to reproduce the product |
| Event stream semantics: unidirectional, cancellable, retryable, idempotent | It is a **contract**, not an implementation; publishing contracts improves review |
| Same stream feeding both JSON and SSE exits | Same |
| Extension-point location: policy registry + adapter, engine untouched | The core design decision; must be written |
| Failure taxonomy (unrecoverable format / no-progress stall / bridge loss) | The taxonomy is a design artifact and does not expose trigger details |
| P2P architecture: layering, ordering, fallback rules, capability audit | Transport design; the site-facing surface is untouched |
| NAT classification results and the conclusion drawn from them | Measurement methodology, transferable, no site knowledge |
| Quantitative metrics (magnitude and ratio only) | Scale and ratio, not absolute figures that reveal capacity |
| Verification methodology for an unreadable answer: tolerance bound, differential localization, discrimination curve | Measurement method and negative results; names no target and describes no access path |
| Ingress admission-control design (`INGRESS-AUTH.md`) | Design of **our own** admission control; the platform routing facts it rests on are already public |
| Runtime substrate architecture (`BOXPROXY.md`) | Control-plane design over documented platform APIs; no keys, no account identifiers |

## Zone 2 — publish after redaction

Valuable, but the raw form leaks production environment or third-party detail.

| Content | Redaction required |
|---|---|
| Production inspection records | Remove host paths, service names, ports, log filenames; keep only "problem → evidence type → action" |
| Error distribution | Keep category and share; drop combinations that identify a specific target or model |
| Latency and throughput | Keep method and magnitude; drop values that let someone back out capacity |
| Capacity and retention policy | Discuss the trade-off (why a retention window exists, what it costs); no thresholds |

## Zone 3 — never published

| Content | Reason |
|---|---|
| Site reverse-engineering notes (7 files under `docs/reverse-engineering/`) | Describes private interfaces of third parties; publishing hands over the method |
| Specific input-injection and detection-avoidance technique | Publishing voids it, and invites targeted blocking |
| Extension-side adapters (selectors, injection timing, wait conditions) | The product's actual moat and the carrier of the above |
| Production configuration, keys, session state, relay inventory | Security |
| Raw inspection logs | Contain host paths and log bodies; allow deployment shape to be inferred |
| Concrete model/target mapping table | Operational information, not design information |
| Which target the verification methodology applies to, and how a session is reached | Questions 1 and 3: it would hand over both the target and the entry method |
| Grant keys, allowlists, per-environment secrets, bridge identities | Security; and question 3 |

---

## Self-check before adding a document

Ask three questions in order; any hit demotes the document a zone.

1. Could a reader **reproduce** integration with a specific target from this? → Zone 3
2. Could a reader **locate** a specific production instance or account? → Zone 2, redact
3. Could a reader **run** this directly? → Zone 3

Only when all three miss does it belong in Zone 1.

### Applied to the three newest files

| File | Q1 reproduce | Q2 locate | Q3 run | Zone |
|---|---|---|---|---|
| `CAPTCHA.md` | no — the target is never named and no access path is described | no | no — the scripts are not included, only their results | 1 |
| `BOXPROXY.md` | no — addresses are shown as templates, never real | no | no | 1 |
| `INGRESS-AUTH.md` | no — it is admission control for our own system | no | **no, because none of it exists** | 1 |

The third column of `INGRESS-AUTH.md` is the honest answer and also the reason it belongs here:
a design that has not been implemented cannot be run. This is worth stating because the usual
failure mode of publishing a security design is not leaking a secret — it is implying that the
protection exists.
