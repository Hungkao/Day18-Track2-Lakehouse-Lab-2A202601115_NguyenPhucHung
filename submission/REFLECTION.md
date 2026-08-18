# Reflection: Lakehouse Anti-Patterns in Production

**Student:** Nguyễn Phúc Hưng (ID: 2A202601115)  
**Course:** AICB-P2T2 · Day 18 · Data Lakehouse Architecture  

---

Our team is most at risk of **The Small-File Accumulation & Orphan File Blindspot** (Anti-Pattern #1 & #4).

In our real-time LLM telemetry pipeline, continuous micro-batch ingestion (e.g., 5-second triggers from Kafka) rapidly accumulates tens of thousands of tiny Parquet files. As measured in Lab NB2 and NB6, uncompacted files degrade point-query latency exponentially and bloat object storage API listing costs.

More critically, our team previously assumed running standard VACUUM was sufficient for lifecycle hygiene. Lab NB6 demonstrated a dangerous production trap: VACUUM only reclaims tombstoned files tracked in _delta_log/ or snapshot metadata. Uncommitted orphan files—left behind by crashed worker jobs or aborted transactions before commit—remain completely invisible to VACUUM regardless of retention settings.

Without an explicit scheduled maintenance routine pairing OPTIMIZE / rewrite_data_files with out-of-band orphan set-difference sweeps (comparing physical object storage against active transaction manifests), our lakehouse suffers from silent FinOps cost leaks and severe query degradation. We are now enforcing automated daily compaction, Z-ordering by tenant/model, and weekly orphan sweeping.
