# Edge cases

A catalogue of the real-world data anomalies the pipeline handles. Each one comes from production - they're in the test suite as anonymized fixtures.

The general rule, restated: **the pipeline never silently drops data.** Everything in this document either gets normalized, gets flagged into the anomaly table with a reason code, or gets dropped only when dropping is unambiguously the right answer.

## Time-related

### Sessions crossing the billing-period boundary

**What it looks like:** A session starts at 11:45 PM on the last day of the cycle, ends at 12:30 AM the next day.

**Why it matters:** If you assign the whole session to the earlier period, the client is paying for time worked in the next cycle. If you assign it to the later period, the VA might be unpaid for the cycle they actually worked.

**How it's handled:** Split the session at the period boundary. The 15 minutes before midnight goes on this cycle's invoice; the 30 minutes after goes on next cycle's. Both halves carry the original session ID with a `_pre` / `_post` suffix so the audit trail stays intact.

**Reason code if flagged:** Never flagged - splitting is deterministic and safe.

### Daylight-savings transitions inside a session

**What it looks like:** A session starts at 1:30 AM on the day clocks spring forward. The wall clock jumps from 1:59 to 3:00. The session "ends" at 2:45 AM, which never existed.

**Why it matters:** Naive subtraction gives a negative duration. Most timezone libraries handle the math, but only if you're using timezone-aware datetimes consistently - one naive datetime in the mix corrupts the result.

**How it's handled:** All session timestamps are converted to timezone-aware datetimes at ingestion. Duration is computed in UTC, *then* the start and end are displayed in local time. The DST gap shows up correctly as a gap, not as negative time.

**Reason code if flagged:** `dst_anomaly` - flagged for review even when the math works, because a VA logging through a DST transition is unusual enough to be worth a human glance.

### Clock skew between API and local system

**What it looks like:** A session has an end timestamp slightly in the future relative to the local system clock.

**Why it matters:** Filters that say "exclude sessions ending after now" will drop legitimate current sessions when the API server's clock is even a few seconds ahead.

**How it's handled:** "Now" for filtering purposes is computed as `local_now + 60s` of tolerance. Anything beyond a minute in the future is treated as a real anomaly, not a skew artifact.

**Reason code if flagged:** `future_end_timestamp` - only when beyond the tolerance window.

### Timezone mismatch between API and billing cycle

**What it looks like:** TimeCamp returns UTC. The billing cycle is defined in the *client's* local timezone (e.g. America/Toronto). The VA is in a third timezone (e.g. Asia/Manila).

**Why it matters:** "The month of March" is not the same UTC range for a Toronto client as for a London client. Doing period math in UTC gives different answers than doing it in the client's local time.

**How it's handled:** Period boundaries are computed in the client's timezone *first*, then converted to UTC for comparison against the session timestamps. The VA's timezone never enters period-boundary logic - it only matters for display in the PDF.

**Reason code if flagged:** Never flagged - handled deterministically.

## Data quality

### Zero-duration sessions

**What it looks like:** Start timestamp equals end timestamp.

**Why it matters:** A VA might have accidentally started and stopped a timer. Silently dropping these is wrong - the VA might dispute their hours total. But billing for them is also wrong.

**How it's handled:** Flagged, excluded from invoice, surfaced in the human-review step.

**Reason code if flagged:** `zero_duration`

### Negative-duration sessions

**What it looks like:** End timestamp is *before* start timestamp.

**Why it matters:** Should be impossible. Isn't.

**How it's handled:** Always flagged, never billed, root-caused at the API or client layer.

**Reason code if flagged:** `negative_duration`

### Sessions with missing task or client references

**What it looks like:** A session has a `task_id` that doesn't match any task in the API's task list. Usually means the task was deleted between when the session was logged and when the report was run.

**Why it matters:** Can't generate a line item without knowing what client to bill.

**How it's handled:** Flagged with the orphan task ID preserved so it can be manually reconciled.

**Reason code if flagged:** `orphan_task` or `orphan_client`

### Duplicate session IDs across pagination

**What it looks like:** The same session ID appears on page 3 *and* page 4 of an API response.

**Why it matters:** Double-counting. Direct billing impact.

**How it's handled:** Dedupe by session ID at ingestion, before any other processing. *Not* dedupe by timestamp - two legitimately distinct sessions can have identical start times.

**Reason code if flagged:** Logged for observability but not flagged for review - dedup is automatic and safe.

## Lifecycle

### Tasks logged against archived clients

**What it looks like:** A session references a client whose status is `archived`.

**Why it matters:** Should the time be billed? Depends on *when* the client was archived relative to the billing period.

**How it's handled:**
- Archived before period start → drop the session entirely (client wasn't active when this work was logged, almost certainly a data entry mistake).
- Archived during the period → bill normally for time logged before the archive date, flag for time logged after.
- Archived after period end → bill normally, the archival doesn't affect this cycle.

**Reason code if flagged:** `archived_mid_session`

### Sessions on tasks that changed billing rate mid-cycle

**What it looks like:** A task's billable rate was edited halfway through the billing period.

**Why it matters:** Time logged before the rate change should bill at the old rate, time after at the new rate.

**How it's handled:** Rate changes are looked up by timestamp. Each session is billed at the rate that was in effect when the session *started*. Out of scope for v1 if the rate-history endpoint isn't exposed by the API.

**Reason code if flagged:** `rate_change_during_session` - only when a single session spans a rate change.

## Idempotence

These aren't anomalies, but they're rules the pipeline enforces so that re-running a report produces an identical PDF.

- Sessions are sorted by `(start_utc, session_id)` before rendering - `start_utc` alone isn't unique enough.
- Rounding rules are applied at a single, documented place (line item totals, not at the session level - rounding early causes drift).
- The anomaly table is sorted by reason code, then session ID, for stable diffing across runs.
