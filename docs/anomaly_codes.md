# Anomaly reason codes

Every session that can't be processed cleanly gets written to the anomaly table with a structured reason code. The human running the report sees these before the PDF is generated.

This is the full set.

| Code | Trigger | Default action |
|------|---------|----------------|
| `zero_duration` | start == end | Exclude from invoice, flag for review |
| `negative_duration` | end < start | Exclude from invoice, flag for review |
| `future_end_timestamp` | end > now + 60s tolerance | Exclude from invoice, flag for review |
| `dst_anomaly` | session spans a DST transition | Include in invoice, flag for review |
| `archived_before_period` | client archived before period start | Drop silently (audit log only) |
| `archived_mid_session` | client archived between start and end | Bill prorated portion, flag remainder |
| `orphan_task` | task_id not found in current task list | Exclude, flag with original ID preserved |
| `orphan_client` | client_id not found in current client list | Exclude, flag with original ID preserved |
| `rate_change_during_session` | billing rate changed between start and end | Bill at start-time rate, flag for review |
| `missing_billable_flag` | session lacks billable/non-billable designation | Default to non-billable, flag for review |

## Design principles for new codes

When a new anomaly type comes up:

1. **One code per root cause, not per symptom.** A session with both a negative duration and an orphan task gets two codes, not a combined one. Codes are independent.
2. **The code names what's wrong, not what to do about it.** `zero_duration` describes the data; the action ("exclude from invoice") is policy and might change.
3. **Default actions err on the side of *not billing*.** When in doubt, exclude. The cost of under-billing is an internal review; the cost of over-billing is a client dispute.
4. **Every code has a fixture in the test suite.** New code without a fixture is a regression waiting to happen.
