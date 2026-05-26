# TimeCamp Reporting Automation

A Python automation I built at [Pixel Buddy](https://pixelbuddy.com) to
turn raw TimeCamp API data into client-ready PDF reports - while catching
the billing edge cases that would otherwise quietly cost the business
money.

> **Note:** The production code lives in a private repo. This writeup
> covers the architecture, the problems it solves, and the design
> decisions behind it.

---

## The problem

Pixel Buddy runs a virtual assistant service. VAs log their time in
TimeCamp across multiple clients, multiple timezones, and multiple
billing arrangements. Every billing cycle, someone had to:

1. Pull each VA's hours from TimeCamp
2. Reconcile time entries against client agreements
3. Catch anomalies before they became billing disputes
4. Generate a clean PDF report per client

Done manually, this was slow and - more importantly - **error-prone in
ways that mattered financially**. A missed overage, a misattributed
entry, or a timezone slip could mean over-billing a client (relationship
damage) or under-billing (direct revenue loss).

I built the automation to handle all four steps and, critically, to
**flag the things humans were missing**.

---

## The headline story: billing-risk detection

This is the part I'd lead with in an interview.

The naive version of this automation just sums hours and renders a PDF.
The version that actually matters identifies time entries that *look
fine* but represent billing risk. A few of the patterns it catches:

- **Hour-cap overages** - a VA logs 22 hours in a week against a
  20-hour retainer. Without detection, the client sees a 22-hour
  invoice and disputes it, or we eat the 2 hours.

- **Misattributed entries** - a VA accidentally logs time to Client A
  while doing work for Client B. Detected by cross-referencing task
  metadata against the client's known task set.

- **Suspiciously round entries** - multiple back-to-back 1.00-hour
  entries on the same day from the same VA. Not always wrong, but
  worth flagging for review.

- **Timezone-induced double-counts** - an entry that crosses midnight
  in the VA's local time but gets split awkwardly by TimeCamp's UTC
  storage, creating what looks like two separate sessions.

- **Stale or open timers** - a timer left running for 14 hours
  overnight. Almost always a forgotten stop, not real work.

Each pattern produces a flag with a severity level. Low-severity flags
get noted in the report appendix; high-severity flags block the report
from being sent until reviewed.

The interesting design decision here was **where to draw the line
between automation and human review**. Auto-correcting billing data is
dangerous - wrong correction is worse than no correction. So the system
detects and surfaces; humans decide.

---

## Architecture

See [`docs/architecture.md`](docs/architecture.md) for the full
diagram and component breakdown.

High level:
TimeCamp API  
│
▼  
[ Ingestion ] --> raw entries (Bronze)
│  
▼  
[ Normalization ] --> timezone-corrected, deduped entries (Silver)
│  
▼  
[ Risk detection ] --> flagged entries + severity
│  
▼  
[ Aggregation ] --> per-client, per-VA summaries (Gold)
│  
▼  
[ PDF generation ] --> client-ready reports

The Bronze/Silver/Gold pattern wasn't accidental - it's the same
medallion architecture I'm now applying to my
[job-industry-trend-tracker](https://github.com/yucompiled/job-industry-trend-tracker)
project. Raw data is always preserved; transformations are layered and
auditable; the final report is reproducible from any historical run.

---

## Pseudocode: the risk detection layer

See [`docs/risk-detection-pseudocode.md`](docs/risk-detection-pseudocode.md)
for a sanitized walkthrough of how flagging works.

---

## What I'd do differently

A few things I'd change with hindsight:

- **Move detection rules to config, not code.** Right now adding a new
  flag means a code change. A YAML- or DB-backed rules engine would
  let ops staff add patterns without me in the loop.

- **Add a feedback loop.** When a human reviews a flagged entry, the
  system should learn whether the flag was useful. Right now it's
  fire-and-forget.

- **Switch the scheduler.** It runs on cron today. Airflow would give
  me proper DAG visibility, retry logic, and a UI for ops - and it's
  what I'm now learning for the trend tracker project.

---

## Stack

- **Python** - core automation, API client, business logic
- **TimeCamp API** - data source
- **pandas** - entry normalization and aggregation
- **ReportLab** - PDF generation
- **cron** - scheduling (Airflow planned)
- **pytz / zoneinfo** - timezone handling

---

## Why this writeup exists

The code is private because it's production tooling for a real business
with real client data. But the *thinking* behind it isn't proprietary,
and the problems it solves are the same problems every operations-heavy
business runs into.

If you're a hiring manager evaluating me for a data engineering role:
this is the work I can defend in the most depth. Ask me about the
timezone bugs - I have stories.

---

📫 **Karl** - [yukarlandrew@icloud.com](mailto:yukarlandrew@icloud.com) · [LinkedIn](https://www.linkedin.com/in/yu-karl/)
