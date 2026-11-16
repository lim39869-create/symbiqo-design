# Operational Metrics

> Numbers from the request-level summary table of a single production instance (one row per
> request, written in the same transaction as the event stream). Window: 2026-10-14 16:24 to
> 2026-11-01 10:34, i.e. 17.8 days. Queries were read-only; no production data was modified.

## Definitions first — otherwise the numbers mean nothing

| Item | Definition |
|---|---|
| Denominator | Every request that entered `request_summaries` in the window (2,426) |
| Success rate | `completed ÷ (completed + failed)`; `cancelled` **excluded** (client-initiated, not a system failure) |
| Latency | `ended_ms − started_ms`, `completed` only, both timestamps valid (1,868 samples) |
| Not counted | 32 records stuck in `running` (see "Known defect"), excluded from the denominator |
| Sample nature | Single deployment, mostly one user, includes the build-out period. **Not a load test.** |

## Overall

| Metric | Value |
|---|---|
| Requests | 2,426 |
| Completed / failed / cancelled / stuck | 1,868 / 425 / 101 / 32 |
| Success rate | **81.5%** |
| Prompt tokens | 36,695,685 |
| Completion tokens | 879,182 |

## By target

The table that matters most: **one engine, six targets of different difficulty.**

| Target | Requests | OK | Failed | Success | Dominant failure |
|---|---:|---:|---:|---:|---|
| deepseek | 1,088 | 970 | 62 | **94.0%** | unrecoverable tool-block format (21) |
| glm | 534 | 417 | 91 | 82.1% | menu item unavailable (32) |
| mimo | 168 | 118 | 34 | 77.6% | no-progress stall (9) |
| qwen | 265 | 172 | 84 | 67.2% | input rewrite missed record (20) |
| minimax | 213 | 123 | 79 | 60.9% | no-progress stall (19) |
| kimi | 158 | 68 | 75 | **47.6%** | risk-control rate limit (23) |

A 94% → 47.6% spread is not code quality fluctuating; it is **target-side resistance
differing**. kimi's dominant failure is rate limiting, not a defect of ours. That distinction
matters when reading the table: one means "we failed", the other means "they blocked us".

## Latency (completed)

| P50 | P90 | P95 | P99 | max | mean |
|---:|---:|---:|---:|---:|---:|
| 7.8s | 26.6s | 39.7s | 92.1s | 306s | 12.8s |

P50 and P99 differ by 12×. The tail comes from three places: waiting for the site to signal
end of generation, risk-control backoff (fixed 60s scale), and bridge rebuilds. **The mean of
12.8s is meaningless**; P95/P99 are the numbers to quote.

## Failure classification

425 failures grouped by **attributable design defect**, not by error string:

| Category | Count | Attribution |
|---|---:|---|
| End-of-generation detection failed (no-progress stall) | 76 | state machine criterion insufficient |
| Input rewrite missed record | 37 | ledger disagreement between daemon and extension |
| Input box not found | 35 | page structure changed, no self-heal |
| Menu item unavailable | 32 | same, interaction path broken |
| Send failed | 29 | timing race |
| Bridge disconnected | 27 | long-connection liveness |
| Unrecoverable tool-block format | 22 | non-standard upstream output; reject rather than repair |
| Extension not connected | 21 | deployment / liveness |
| **Anti-automation block (risk control + captcha + rate limit)** | **63** | external resistance, **14.8% of all failures** |

Two observations:

1. **Anti-automation accounts for only 14.8% of failures.** The overwhelming majority are our
   own state machine misjudging completion. That sets the optimisation order: fix criteria
   first, argue about evasion second.
2. "Input box not found" plus "menu item unavailable" is 67 failures pointing at one thing:
   **detecting and self-healing page-structure change is the first cost of a collection system,
   not anti-automation.** This conclusion is worth more than any success-rate figure.

## Three-window trend

| Window | Requests | Success |
|---|---:|---:|
| 10-14 → 10-16 | 129 | 89.4% |
| 10-17 → 10-19 | 982 | 79.2% |
| 10-20 → 10-22 | 851 | 82.4% |
| 10-23 → 10-25 | 413 | 82.4% |
| 10-26 → 10-28 | 26 | 92.0% |
| 10-30 → 11-01 | 25 | 68.4% |

Volume is concentrated early; late windows are spot checks. **This trend must not be read as
improvement**: window 2 is the heaviest week (nearly a thousand requests) and later samples
are too small. Any "we improved by x%" claim has to control for sample size first, or it is
self-deception.

## Known defect (found while querying)

32 records are stuck in `running`, the earliest at the start of the window. This means a
**finalize-path leak**: a restart or abnormal exit leaves in-flight summaries without a
terminal state. Effect: the denominator is permanently polluted and the "in progress" counter
climbs forever. Fix direction: on startup, converge expired `running` rows to `interrupted`
and count them in the denominator.

## Boundaries of this data

- Single instance, mostly single-user. **It says nothing about concurrency**; that needs a
  separate load test.
- Includes build-out-period requests; the early version's success rate is not the current one.
- Excludes requests retried manually on another platform after a block, so true end-to-end
  completion is higher than shown.
