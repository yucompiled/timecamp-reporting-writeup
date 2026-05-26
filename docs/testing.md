# Testing approach

The pipeline is tested at three levels. The most valuable layer - by a wide margin - is the third one.

## Unit tests

Standard pytest coverage of the pure functions: timezone conversion, period-boundary splitting, dedupe, rate lookup. These are fast, deterministic, and catch regressions in the math.

Roughly 60 of these. They run in under a second.

## Integration tests

The ingestion -> normalization -> render path tested against a mocked TimeCamp API response. Verifies that the stages compose correctly and that the PDF output is byte-identical across runs (idempotence).

Roughly 15 of these. They run in a few seconds.

## Real-world anomaly fixtures

This is the layer that catches the bugs the other two miss.

Every time a real billing-period run surfaced a session that the pipeline handled wrong (or *almost* handled wrong), I:

1. Pulled the raw session data out of the API response
2. Anonymized the client name, task name, and any identifying timestamps
3. Saved it as a fixture file in `tests/fixtures/anomalies/`
4. Wrote a test asserting how it should be processed
5. Fixed the code until the test passed

The fixture library currently has about 40 cases. Each one represents a real situation the pipeline failed at least once. Synthetic test cases never found bugs at the rate these did.

### Why this matters more than coverage percentages

A unit test verifies that `split_session_at_boundary()` does what I think it does. A fixture verifies that the pipeline handles *the actual shape of real data the API returns*, which is not always what the API docs say it returns.

Examples of fixtures that caught regressions a unit test wouldn't have:

- A session where the API returned `null` for the description field - the renderer crashed because it assumed the field was always a string.
- A session where the task name contained a forward slash, which broke the PDF filename generation.
- A session where two consecutive API pages returned the same session, but with a *different* end timestamp on each page (the session was still in progress when page 1 was fetched, completed by the time page 2 was fetched). Dedupe by ID alone took the older copy; the fix was to dedupe by ID and prefer the most recent version.

None of these were predictable from the API documentation. All of them broke production at least once.

## What's not tested well

Being honest about limitations:

- **Concurrency.** The pipeline runs serially. If two report runs happened simultaneously against the same anomaly table, the second would clobber the first. Hasn't been a problem because the schedule is sequential, but it's a known limitation.
- **API failure modes.** I've handled the common ones (rate limits, 5xx retries) but haven't simulated a partial outage in the middle of pagination. The retry logic *should* handle it; I haven't proven it does.
- **PDF rendering edge cases.** Very long client names, unusual character sets, and right-to-left text all have known visual issues that I haven't prioritized fixing.
