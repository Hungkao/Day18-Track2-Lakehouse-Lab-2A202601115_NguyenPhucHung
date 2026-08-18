# Báo Cáo Kết Quả Thực Nghiệm — Day 18 Lakehouse Lab

**Học viên:** Nguyễn Phúc Hưng  
**Track:** Track 2 — Data Lakehouse Architecture  
**Kết quả tổng thể:** 100 / 100 điểm (All Green)

---

## I. Tổng Hợp Chấm Điểm Theo Rubric (100/100 Pts)

### Part A — Foundations (44 / 44 pts)
1. **NB1 — Delta Lake Basics (8/8 pts):**
   * Tạo bảng Delta thành công, _delta_log/*.json ghi nhận đầy đủ transaction log metadata.
   * **Schema enforcement** chặn thành công bản ghi lỗi type (ge='thirty').
   * **Schema evolution** với schema_mode='merge' tự động mở rộng cột 	ier mà không làm hỏng schema cũ.
   * DuckDB zero-copy query trực tiếp qua Apache Arrow.
2. **NB2 — Small-File & OPTIMIZE + Z-Order (12/12 pts):**
   * Tái hiện thành công sự cố 200 file nhỏ (streaming micro-batches).
   * Compaction giảm số lượng file từ 200 xuống còn ~10-50 files.
   * Z-ORDER theo user_id đem lại tỉ lệ loại trừ file (**files-pruned ratio $\ge 10\times$**).
3. **NB3 — Time Travel & MERGE Upsert (12/12 pts):**
   * MERGE 100K rows (50K updates, 50K inserts) hoàn tất trong $< 1$ giây.
   * dt.restore(2) rollback dữ liệu lỗi thành công (số dòng score < 0 về chính xác 0).
   * history() ghi nhận $\ge 5$ versions bao gồm transaction RESTORE.
4. **NB4 — Medallion Pipeline LLM Observability (12/12 pts):**
   * Bronze: 200,000 dòng raw logs.
   * Silver: Deduplication loại bỏ 9,948 dòng trùng lặp (Silver < Bronze), phân vùng theo date.
   * Gold: Tổng hợp đúng chỉ số {50}, p_{95}$ latency, cost_usd, error_rate cho 7 ngày $\times$ 3 models (claude-haiku-4-5, claude-sonnet-4-6, claude-opus-4-7).

### Part B — Lakehouse 2026 (50 / 50 pts)
5. **NB5 — Apache Iceberg & Catalog as Control Plane (13/13 pts):**
   * Bảng tạo hoàn toàn qua Catalog (SQLite SqlCatalog); partition spec sử dụng DayTransform(ts).
   * **Hidden Partitioning:** Lọc trên cột gốc 	s tự động prune metadata từ 10 files xuống 1 file (Pruning ratio **\times \ge 5\times$**).
   * Schema evolution: Rename cột latency_ms $\rightarrow$ latency_millis giữ nguyên ield_id (metadata-only update, zero data rewrite).
6. **NB6 — Table Maintenance: 4 Mandatory Jobs (13/13 pts):**
   * **Job 1 (Compaction):** Giảm từ 200 files xuống 10 files (\times$ reduction).
   * **Job 2 (Clustering):** Z-order stats pruning cho phép skip $\ge 50\%$ số files cho point-query.
   * **Job 3 (Snapshot Expiry):** Thu hồi commit history, metadata Iceberg giảm về 3 snapshots.
   * **Job 4 (Orphan Removal):** Quét và xoá sạch 3 uncommitted orphan files mà lệnh VACUUM thông thường bỏ sót.
   * **Job 5 (Checkpointing):** Tạo Parquet checkpoint và cập nhật _last_checkpoint.
7. **NB7 — Multimodal & Vectors in Table (13/13 pts):**
   * Đo lường **Random-access amplification** ($\ge 5	imes$) do kích thước row-group khi đọc inline blob.
   * Lượng tử hoá int8 vector giảm dung lượng lưu trữ trên đĩa **.8\times$** mà vẫn bảo toàn recall và topic clustering fidelity ($>90\%$).
   * **Tái hiện Lifecycle Desync Bug:** Bản ghi bị xoá trong bảng Delta trả về 0 kết quả, trong khi external vector DB độc lập vẫn trả về dữ liệu rác (stale index).
8. **NB8 — Agents as Consumers & Provenance (11/11 pts):**
   * Trajectory Medallion: Silver phân vùng theo gent_version (policy-v2, policy-v3).
   * Time-travel pin version đảm bảo tính tái lập 100% cho training run.
   * Phân vùng 4 rổ bản quyền tuân thủ **EU AI Act Article 10** (licensed, public_domain, scraped_optout_checked, synthetic), loại trừ hoàn toàn dữ liệu UNCLASSIFIED.

### Part C — Reproducibility (6 / 6 pts)
* make test xanh: 24/24 pytest tests pass.
* make run-all xanh: Chạy thông suốt 8/8 notebooks trong ~53s.

---

## II. Danh Mục Deliverables Trong Repo
1. 
otebooks/*.ipynb: Toàn bộ 8 notebook đã thực thi kèm output cell.
2. submission/screenshots/:
   * README.md & lakehouse_tree_and_delta_log.txt: Chi tiết cây lưu trữ _lakehouse/ và commit log JSON của _delta_log/.
3. submission/REFLECTION.md: Phân tích $\le 200$ từ về bẫy production *Small-File Accumulation & Uncommitted Orphan Files*.
4. submission/bonus/ARCHITECTURE.md: Thiết kế kiến trúc Lakehouse cho bài toán *LLM Observability 1B requests/day*.
