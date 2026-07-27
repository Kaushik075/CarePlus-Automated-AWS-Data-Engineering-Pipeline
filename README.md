# CarePlus — Automated AWS Data Engineering Pipeline

Centralizes fragmented MySQL ticket data and raw system logs from a simulated BPO support operation into a single automated AWS pipeline — turning scattered, manually-queried data into live Power BI dashboards with zero manual reporting.

## The Problem

CarePlus is a simulated BPO company running customer support across multiple channels. Support ticket data lived in MySQL. Separately, every backend interaction — API calls, session activity, CPU load, errors — was written to a raw `.log` file with no schema and no easy link back to the ticket system except a shared ticket ID buried in unstructured text.

Answering a basic question — *"which agent has the slowest resolution time, and were their sessions hitting backend errors?"* — meant manually opening two systems and cross-referencing by hand.

## What It Does

**Ingestion** — Daily incremental pulls, tracked by a date-checkpoint file so each run only ingests new data. Tickets pulled from MySQL, logs picked up as raw files, both landed in S3.

**Transform** — Raw log files are parsed with a purpose-built regex engine handling a non-standard pipe-delimited format, then cleaned: typo-correction on severity levels, negative-response-time filtering, dedup, type casting. Ticket CSVs are transformed via a managed Spark job (AWS Glue). Both output typed, compressed Parquet.

**Warehouse + Analytics** — Parquet loads into Redshift Serverless via IAM-role-based COPY (no embedded credentials). Athena enables ad-hoc SQL directly on the S3 layer without a warehouse load. Power BI connects to Redshift for two live, auto-refreshing dashboards.

## Architecture

<img width="5214" height="2224" alt="Image" src="https://github.com/user-attachments/assets/f0c40dde-7843-42e3-81fd-f3e3489a3fbc" />

```
MySQL (tickets) ──┐
                   ├──> S3 Raw ──> Transform ──> S3 Processed (Parquet) ──┐
Raw log files ─────┘   (Lambda / Glue)                                    │
                                                                            ├──> Redshift Serverless ──> Power BI
                                                                            └──> Athena (ad-hoc SQL)
```

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

## Key Engineering Decisions

- **Date-checkpoint incremental loading** instead of full reloads — every run only pulls new data, keeping ingestion idempotent and cheap.
- **Split transform compute by workload shape** — Lambda for lightweight, event-driven log parsing; Glue for heavier, Spark-benefiting CSV-to-Parquet ticket transforms.
- **Regex-based log parsing** over a semi-structured, pipe-delimited format — a common real-world pattern for legacy log systems.
- **Data quality handled in-pipeline** — severity-label typo correction, negative-response-time filtering, and dedup happen before data lands in Parquet.
- **IAM role-based Redshift COPY** instead of embedded credentials — no secrets in query text, ever.
- **Parquet as the intermediate format** — columnar, typed, and compressed before warehouse load.


---




## Data Model

Built as a Power BI star schema: `dim_date` centralizes date logic across both fact tables, `dim_tickets` bridges `support_tickets` and `support_logs` through a single `ticket_id`, and a dedicated `key_measures` table holds all DAX calculations.


<img width="1536" height="1024" alt="Image" src="https://github.com/user-attachments/assets/f93335f2-a990-4346-b9ae-b233b62aa57e" />

---

## Dashboards

**CarePlus Ticket Insights** — total/resolved/open/escalated tickets, avg interactions, avg resolution time, tickets by agent, ticket status by channel, resolution time by category and priority.

![Ticket Insights]

<img width="1126" height="541" alt="Image" src="https://github.com/user-attachments/assets/b7eaa952-7ae9-4540-9878-a37f31fc331d" />

**CarePlus Support Logs** — total logs, avg CPU, avg response time, logs by user agent and severity, CPU trend over time, response-time distribution vs. ticket volume.

![Support Logs]

<img width="958" height="540" alt="Image" src="https://github.com/user-attachments/assets/a867fa34-1b69-41c5-9dc5-c4ccbd3ad98e" />

---



## Skills Applied

Serverless ETL design · event-driven vs. batch compute tradeoffs · schema design for OLAP · incremental data loading patterns · SQL aggregation and analytical query writing · Redshift warehouse modeling · Power BI dashboard design for operational reporting.

---

## What I actually built:**

- The full ingestion → transform → storage → warehouse → analytics → BI pipeline described above, end to end
- Incremental, checkpoint-based daily loading for both data sources
- Custom regex log parser with real data-quality cleaning logic
- IAM role-based secure warehouse loading
- A star-schema Power BI model with two live dashboards

