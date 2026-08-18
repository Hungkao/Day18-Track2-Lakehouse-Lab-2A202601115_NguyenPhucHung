# Architecture Brief: Foundation Model LLM Observability at Scale (1B Requests/Day)

**Author:** Nguyễn Phúc Hưng (ID: 2A202601115)  
**Role:** Senior Data & Lakehouse Architect on-call  
**Track:** Track 2 — Data Lakehouse Architecture (Day 18 Bonus Challenge)  

---

## 1. Executive Summary & Constraints
* **Volume:** 1 Billion requests/day (~5 KB/request payload) = **5.0 TB/day** raw ingestion (~150 TB/month).
* **Budget Cap:** Total storage cost $\le$ **,000 / month**.
* **Key SLAs:**
  1. Real-time cost & latency dashboards per tenant refreshed every **5 minutes**.
  2. Full prompt/response payloads retained for **7 days** (incident analysis & quality review).
  3. Long-term aggregates retained for **1 year** (compliance, billing, capacity planning).
  4. Mandatory **PII tokenization** at the Bronze boundary prior to query access.

---

## 2. Medallion Storage Architecture

`
                    Kafka Streaming (5s micro-batch)
                                 │
                                 ▼
        ┌──────────────────────────────────────────────────┐
        │  BRONZE: raw_llm_events (Delta Lake / Append-Only)│
        │  - Inline SHA-256 HMAC Tokenization for PII      │
        │  - Partition: None (high ingestion throughput)   │
        │  - Retention: 7 Days (Hard TTL via Expiry)       │
        └────────────────────────┬─────────────────────────┘
                                 │
                        Structured Streaming
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  SILVER: parsed_llm_calls (Delta / Iceberg)       │
        │  - Structured schema (tokens, latency, status)   │
        │  - Dedup via 
equest_id (Watermark 2 hours)    │
        │  - Partition: date + Z-ORDER: (tenant_id, ts)│
        │  - Retention: 7 Days (Hot S3 Standard)           │
        └────────────────────────┬─────────────────────────┘
                                 │
                       5-min Continuous Rollup
                                 │
        ┌────────────────────────▼─────────────────────────┐
        │  GOLD: tenant_daily_metrics / tenant_5m_metrics   │
        │  - p50/p95/p99 latency, cost_usd, error rates    │
        │  - Partition: date + Z-ORDER: 	enant_id      │
        │  - Retention: 365 Days (Warm S3 Standard-IA)     │
        └──────────────────────────────────────────────────┘
`

---

## 3. Storage Optimization & FinOps Tiering

### 3.1 FinOps Cost Breakdown ($\le$ ,000/Month Target)
* **Bronze & Silver Hot Tier (Days 0–7):**
  * 7 days $	imes$ 5 TB/day = 35 TB Bronze raw (ZSTD compressed $pprox$ 14 TB).
  * 7 days $	imes$ Silver parsed $pprox$ 5 TB.
  * Total active storage: ~19 TB on AWS S3 Standard (.023/GB) = **~ / month**.
* **Gold Aggregate Tier (Days 0–365):**
  * 365 days $	imes$ 100 MB/day aggregate metrics = ~36.5 GB on S3 Standard-IA = **~.50 / month**.
* **Cold Storage Archive (Optional 30-day compliance deep archive on Glacier Flexible):**
  * 150 TB raw payload $	imes$ .0036/GB = **~ / month**.
* **Total Storage & Maintenance Compute:** $pprox$ **,500 – ,200 / month**, comfortably within the **,000/month** ceiling.

---

## 4. Operational Maintenance Schedule (The 5 Mandatory Jobs)

1. **Hourly Micro-Compaction (Job 1):**
   * Compact 5-second streaming files into 128 MB – 256 MB Parquet files.
   * Eliminates the small-file read penalty ($>10\times$ file skipping efficiency).
2. **Daily Z-Order Clustering (Job 2):**
   * Run OPTIMIZE silver.llm_calls ZORDER BY (tenant_id, model) on the previous day's partition to maximize metadata-driven min/max stats pruning.
3. **Automated Snapshot Expiry & Vacuum (Job 3):**
   * Retain 7 days of Delta commit history (VACUUM RETAIN 168 HOURS).
4. **Out-of-band Orphan Sweeper (Job 4):**
   * Perform weekly set-difference sweeps between object store prefixes and active Delta log manifests to eliminate uncommitted parquet files from aborted task executors.
5. **Deterministic Checkpointing (Job 5):**
   * Trigger Parquet-based checkpoint generation every 10 commits to bound log-replay latency.

---

## 5. Architectural Trade-offs & Rejected Alternatives

| Candidate Alternative | Evaluation & Why Rejected |
|---|---|
| **Raw JSON on Object Store + Hive Partitions** | **Rejected.** Hive-style partitioning requires explicit query filters and lacks ACID isolation. Concurrent compaction corrupts in-flight reads. |
| **Separate Vector DB for Embedding Telemetry** | **Rejected for Audit.** Storing embeddings in external vector databases introduces the *lifecycle desync bug* (entity deleted in primary table remains queryable in vector index). In-table embeddings with DuckDB/Hudi native vectors guarantee strict ACID coherence. |
| **Unpartitioned Single-Table Log** | **Rejected.** Full-table scans across 1B requests/day cost thousands of dollars in query compute and breach the p95 $<$ 1s dashboard SLA. |
