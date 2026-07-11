# CarePlus — Automated AWS Data Engineering Pipeline


## The Problem

CarePlus is a simulated BPO company running customer support across multiple channels. Two critical data sources had no connection to each other:

A support ticket is created and tracked in MySQL. Separately, every backend interaction on that same ticket — API calls, session activity, CPU load, errors — gets written to a raw `.log` file with no schema and no link back to the ticket system except a shared ID buried in unstructured text.

Answering a basic question — *"which agent has the slowest resolution time, and were their sessions hitting backend errors?"* — meant manually opening a database, manually opening log files, and manually cross-referencing by hand. There was no single source of truth, no automation, and no way to trust the numbers without a human doing the joining every time.

## What CarePlus Does

CarePlus runs two parallel, fully automated ingestion-to-dashboard pipelines — one for structured ticket data, one for semi-structured log data — that converge into a single analytics layer.

**Ingestion** — Daily incremental pulls, tracked by a date-checkpoint file so each run only ingests new data (no reprocessing, no duplicates). Tickets pulled from MySQL, logs picked up as raw files, both landed in S3.

**Transform** — Raw log files are parsed with a purpose-built regex engine handling a non-standard pipe-delimited format, then cleaned: typo-correction on severity levels, negative-response-time filtering, dedup, type casting. Ticket CSVs are transformed via a managed Spark job. Both output typed, compressed Parquet.

**Warehouse + Analytics** — Parquet loads into Redshift Serverless via IAM-role-based COPY (no embedded credentials). Athena enables ad-hoc SQL directly on the S3 layer without a warehouse load. Power BI connects to Redshift for two live, auto-refreshing dashboards.

**Result:** ticket volume, resolution time, agent performance, and backend system health are now one query away — refreshed daily, with zero manual joining.

---

## Results

| Metric | Before | After |
|---|---|---|
| Cross-referencing ticket + log data | Manual, per-request | Automated, joined on `ticket_id` |
| Reporting refresh | Ad-hoc, manual pull | Daily, automated |
| Source of truth | 2 disconnected systems | 1 unified analytics layer |
| Data quality issues (typo severities, bad response times, dupes) | Unhandled | Caught and corrected in transform |

Sample dashboard output from a single day of data (July 1, 2025):

| | |
|---|---|
| Total tickets | 48 |
| Resolved / Open / Escalated | 39 / 8 / 1 |
| Avg resolution time | 1,098 min |
| Total backend logs | 89 |
| Avg CPU at event time | 55.1% |
| Avg response time | 983 ms |

---

## Architecture

![Pipeline Architecture](architecture/pipeline_architecture.png)

```
MySQL (tickets) ──┐
                   ├──> S3 Raw ──> Transform ──> S3 Processed (Parquet) ──┐
Raw log files ─────┘   (Lambda / Glue)                                    │
                                                                            ├──> Redshift Serverless ──> Power BI
                                                                            └──> Athena (ad-hoc SQL)
```

Two sources, two transform paths, one warehouse. Tickets go through AWS Glue (heavier CSV transform, benefits from managed Spark). Logs go through AWS Lambda, triggered directly on S3 upload — lightweight, event-driven, no idle compute.

---

## Data Model

**support_tickets** (MySQL → Redshift)
`ticket_id` (PK) · `created_at` · `resolved_at` · `agent` · `priority` · `issue_category` · `num_interactions` · `status` · `channel`

**support_logs** (raw files → Redshift)
`timestamp` · `log_level` · `component` · `ticket_id` (FK) · `session_id` · `ip` · `response_time` · `cpu` · `event_type` · `error` · `user_agent` · `message` · `debug`

**Relationship:** one ticket → many logs, joined on `ticket_id`.

---

## Dashboards

**CarePlus Ticket Insights** — total/resolved/open/escalated tickets, avg interactions, avg resolution time, tickets by agent, ticket status by channel, resolution time by category and priority.

![Ticket Insights](dashboards/ticket_insights.png)

**CarePlus Support Logs** — total logs, avg CPU, avg response time, logs by user agent and severity, CPU trend over time, response-time distribution vs. ticket volume, with a day-by-day selector across the full month.

![Support Logs](dashboards/support_logs.png)

---

## Tech Stack

| Layer | Tool |
|---|---|
| Source DB | MySQL |
| Ingestion | Python (boto3, SQLAlchemy, python-dotenv) |
| Transform (logs) | AWS Lambda + regex parsing |
| Transform (tickets) | AWS Glue |
| Storage | Amazon S3 (raw + processed, Parquet) |
| Warehouse | Amazon Redshift Serverless |
| Ad-hoc query | Amazon Athena |
| Visualization | Power BI |

---

## Key Engineering Decisions

- **Date-checkpoint incremental loading** instead of full reloads — every run only pulls new data, keeping ingestion idempotent and cheap without needing a full CDC tool.
- **Split transform compute by workload shape** — Lambda for lightweight, event-driven log parsing; Glue for heavier, Spark-benefiting CSV-to-Parquet ticket transforms.
- **Regex-based log parsing** over a semi-structured, pipe-delimited format — a common real-world pattern for legacy log systems that don't emit JSON.
- **Data quality handled in-pipeline**, not left to the warehouse — severity-label typo correction, negative-response-time filtering, and dedup happen before data ever lands in Parquet.
- **IAM role-based Redshift COPY** instead of embedded credentials in SQL — no secrets in query text, ever.
- **Parquet as the intermediate format** — columnar, typed, and compressed before warehouse load, keeping both storage and query cost down.

## Skills Applied

Serverless ETL design · event-driven vs. batch compute tradeoffs · schema design for OLAP · incremental data loading patterns · SQL aggregation and analytical query writing · Redshift warehouse modeling · Power BI dashboard design for operational reporting.

---
