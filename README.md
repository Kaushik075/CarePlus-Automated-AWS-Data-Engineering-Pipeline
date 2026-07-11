# CarePlus — Automated AWS Data Engineering Pipeline


## The Problem

CarePlus is a simulated BPO company running customer support across multiple channels. Two critical data sources had no connection to each other:

A support ticket is created and tracked in MySQL. Separately, every backend interaction on that same ticket — API calls, session activity, CPU load, errors — gets written to a raw `.log` file with no schema and no link back to the ticket system except a shared ID buried in unstructured text.

**Answering a basic question** — *"which agent has the slowest resolution time, and were their sessions hitting backend errors?"* — meant manually opening a database, manually opening log files, and manually cross-referencing by hand. There was no single source of truth, no automation, and no way to trust the numbers without a human doing the joining every time.

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


<img width="5214" height="2224" alt="Image" src="https://github.com/user-attachments/assets/f0c40dde-7843-42e3-81fd-f3e3489a3fbc" />

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

The full model is built out in Power BI as a proper star schema — not just the two raw source tables. dim_date centralizes date logic so both support_tickets and support_logs can be filtered and trended consistently without duplicating calendar fields. dim_tickets acts as a bridge dimension, cleanly relating the two fact tables through a single ticket_id rather than a direct many-to-many join. A dedicated key_measures table holds all DAX measures (Avg CPU %, Error Rate %, avg resolution minutes, Total Logs, etc.), keeping calculations separate from raw columns. This structure is what powers both dashboards below.

<img width="1536" height="1024" alt="Image" src="https://github.com/user-attachments/assets/f93335f2-a990-4346-b9ae-b233b62aa57e" />

---

## Dashboards

**CarePlus Ticket Insights** — total/resolved/open/escalated tickets, avg interactions, avg resolution time, tickets by agent, ticket status by channel, resolution time by category and priority.

![Ticket Insights]

<img width="1126" height="541" alt="Image" src="https://github.com/user-attachments/assets/b7eaa952-7ae9-4540-9878-a37f31fc331d" />

**CarePlus Support Logs** — total logs, avg CPU, avg response time, logs by user agent and severity, CPU trend over time, response-time distribution vs. ticket volume, with a day-by-day selector across the full month.

![Support Logs]

<img width="958" height="540" alt="Image" src="https://github.com/user-attachments/assets/a867fa34-1b69-41c5-9dc5-c4ccbd3ad98e" />

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
