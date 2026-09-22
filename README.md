Crypto Live Market Pipeline — Real-Time Cryptocurrency Analytics

An automated, scheduled data pipeline that ingests live cryptocurrency prices from a public REST API daily, processes them through a Bronze/Silver/Gold architecture using Databricks Auto Loader for true incremental ingestion, and powers a natural-language analytics dashboard that emails a fresh snapshot every day.

Show Image

Overview

Unlike a typical portfolio project built on a static historical dataset, this pipeline works against genuinely live data. Every day, it calls the CoinGecko public API to capture a real, current price snapshot for 10 major cryptocurrencies, and lands it incrementally into a growing Delta Lake table — the data is different every single run, not a fixed file being reprocessed.

The pipeline uses Databricks Auto Loader to handle incremental file ingestion properly, tracking exactly which files have already been processed rather than reloading everything each time, then transforms the data through a Bronze/Silver/Gold medallion architecture into a time-series fact table. The entire pipeline is orchestrated as a scheduled Databricks Workflow with task dependencies, retry logic tested against a real external service, and email notifications on failure.

On top of this, a Genie-powered natural language dashboard provides live market analytics, automatically refreshing daily and emailing a PDF snapshot to keep stakeholders updated without anyone needing to log in and check manually.

Tech Stack
Category	Tools
Platform	Databricks (Serverless Compute, Unity Catalog)
Languages	Python (PySpark), SQL
Storage	Delta Lake
Ingestion	Databricks Auto Loader (incremental file processing)
Orchestration	Databricks Workflows (scheduled, retries, email alerts)
Analytics	Databricks Genie, Databricks SQL Dashboards
External API	CoinGecko (live, public, no key required)
Architecture
CoinGecko API (live, public)
   │
   ▼
Ingestion — fetch_and_land_prices (with retry logic on network failure)
   │  lands timestamped JSON snapshot into a Volume
   ▼
Bronze Layer — Auto Loader (incremental, tracks processed files via checkpoint)
   │
   ▼
Silver Layer — cleaned, flattened, typed (append-only, grows every run)
   │
   ▼
Gold Layer — Dim_Coin + Fact_Crypto_Prices (time-series, append-only)
   │
   ├──► Genie — natural language dashboard queries
   └──► Databricks Dashboard — daily refresh + emailed PDF snapshot

Orchestrated by a scheduled Databricks Workflow (daily, 9 AM):
fetch → transform (bronze/silver) → build gold → data quality check
with retries and failure email notifications on every task
Data Model

Fact table: fact_crypto_prices — append-only, time-series. Every pipeline run adds a new price snapshot per coin, rather than overwriting the previous one, so historical trends are preserved.

Dimension: dim_coin — coin ID to display name mapping.

Coins tracked: Bitcoin, Ethereum, Tether, BNB, Solana, XRP, Cardano, Dogecoin, Polkadot, Litecoin.

Key Highlights
Built ingestion against a genuinely live, external public API rather than a static dataset — the data is real and different on every run
Used Databricks Auto Loader for true incremental ingestion, correctly tracking which files have already been processed instead of naive full reloads
Fact table proven to grow correctly across multiple runs — verified doubling from 10 to 20 rows across two consecutive pipeline executions, with two distinct price snapshots per coin
Orchestrated with a scheduled daily Databricks Workflow, including retry logic tested against a real external service's actual failure modes (rate limits, timeouts), not simulated ones
Configured email notifications on job failure, and a daily automated dashboard PDF subscription
Identified and fixed a data quality bug in the dashboard where market cap was summed across all historical snapshots instead of the latest one per coin — inflating "Total Market Cap" by over 30x before correction
Challenges & Key Decisions

Metric View Aggregation Bug The "Total Market Cap" tile initially showed $63 trillion, then $114 trillion after a first attempted fix — both far off from the realistic ~$2-3.5 trillion for these 10 coins combined. Traced to the dashboard's underlying metric view summing market_cap_usd across every historical snapshot in the append-only fact table, rather than filtering to the most recent snapshot per coin. Diagnosed the actual query being executed, identified that the first fix attempt targeted the wrong layer (the raw table instead of the metric view the dashboard actually queries), and corrected it by filtering to the latest snapshot_hour before aggregating.

Serverless Compute Limitation Databricks' %run magic for sharing configuration across notebooks doesn't work the same way on serverless compute. Worked around this by inlining shared configuration values directly into each notebook — a known trade-off that would be refactored using Job parameters in a production setting.

Real Network Reliability Unlike a static dataset, calling a live public API means genuine network failures are possible. Implemented retry logic with exponential backoff specifically on the ingestion step, since it's the component making the actual external network call.

Repository Structure
├── 01_ingestion/
│   └── fetch_and_land_prices.py       # Calls CoinGecko API, lands JSON snapshot, retry logic
├── 02_transformation/
│   ├── bronze_to_silver.py            # Auto Loader ingestion + Bronze-to-Silver cleaning
│   └── silver_to_gold.py              # Dim_Coin + Fact_Crypto_Prices construction
├── 03_quality/
│   └── data_quality_check.py          # Automated validation checks
├── 04_utils/
│   └── shared_config.py               # Catalog/schema/volume paths, coin list
├── images/
│   ├── architecture-diagram.png
│   └── dashboard-screenshot.png
└── README.md
Key Insights
The fact table correctly grew from 10 to 20 rows across two consecutive pipeline runs, proving genuine incremental data capture rather than static reprocessing
Aggregating "current state" metrics (like total market cap) on an append-only, growing table requires explicitly filtering to the latest snapshot per entity — a subtle but critical distinction from one-time historical aggregation
Retry logic against a real external API needs to account for genuine, unpredictable failure modes rather than only simulated errors
Data Source

CoinGecko API — free, public, no API key required. Live cryptocurrency market data (price, market cap, 24h volume, 24h change) for major coins, updated in real time.
