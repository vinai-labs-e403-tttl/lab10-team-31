# Báo Cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Team 31 (Lab 10 — ETL Pipeline & Data Observability)  
**Thành viên:**
| Tên | Vai trò (Day 10) | Role |
|-----|------------------|------|
| Trịnh Ngọc Tú | Ingestion / Raw Owner | Load CSV, schema map |
| Trần Kiên Trường  | Cleaning & Quality Owner | CR rules + Expectations |
| Đăng Thanh Tùng | Embed & Idempotency Owner | Chroma upsert, chunk_id stable |
| Đặng Quang Minh | Monitoring / Docs Owner | Freshness check, runbook, group report |

**Ngày nộp:** 2026-04-15  
**Repo:** https://github.com/vinai-labs-e403-tttl/lab10-team-31  
**Status:** ✅ Sprint 1–4 hoàn thành

---

## 1. Pipeline tổng quan (150–200 từ)

### Tóm tắt luồng

**Nguồn raw:** `data/raw/policy_export_dirty.csv` (CSV mẫu 10 records, testabenew từ lab)

**End-to-end flow:**
1. **Load:** `load_raw_csv()` → 10 records dict
2. **Clean:** Áp 9 rules (CR1-CR9) → 6 cleaned + 4 quarantine  
   - CR1–6: allowlist, date norm, empty check, dedupe, HR stale, split
   - **CR7–9 (Sprint 2):** suspicious doc_id, invalid exported_at, chunk length validator
3. **Validate:** Expect E1–E5, halt nếu critical fail (E3 refund stale 14d)
4. **Embed:** Chroma upsert `chunk_id` stable (SHA256 hash → idempotent)
5. **Monitor:** Lưu manifest.json (run_id, counts, timestamp)

### Lệnh chạy end-to-end

```bash
python etl_pipeline.py run # default params (normal mode)
python etl_pipeline.py run --run-id sprint2 # named run
# Verify: check artifacts/manifests/manifest_sprint2.json
```

**Manifest key:** run_id="sprint2", cleaned_records=6, quarantine_records=4, latest_exported_at="2026-04-15T08:00:00"

---

## 2. Cleaning & expectation (150–200 từ)

### Baseline + Sprint 2 additions

**9 cleaning rules (transform/cleaning_rules.py):**

| Rule | Tác dụng | Sprint |
|------|---------|--------|
| CR1 | Allowlist doc_id (only policy_refund_v4, sla_p1_2026, etc.) | Baseline |
| CR2 | Normalize effective_date (DD/MM/YYYY → YYYY-MM-DD) | Baseline |
| CR3 | Quarantine empty effective_date | Baseline |
| CR4 | Deduplicate chunk_text | Baseline |
| CR5 | Quarantine HR leave if effective_date < 2026-01-01 (stale version) | Baseline |
| CR6 | Split chunk by `;` delimiter | Baseline |
| **CR7** | **Flag suspicious doc_id (legacy_, test_, beta_, deprecated_)** | **Sprint 2** |
| **CR8** | **Quarantine invalid exported_at (empty or future timestamp)** | **Sprint 2** |
| **CR9** | **Quarantine chunk_text if len < 20 chars (fragment)** | **Sprint 2** |

**5 expectations (quality/expectations.py):**

| Exp | Severity | Pass/Fail | Detail |
|-----|----------|-----------|--------|
| E1 | halt | ✓ PASS | min_one_row = 6 |
| E2 | halt | ✓ PASS | no_empty_doc_id |
| E3 | halt | ✓ PASS | refund_no_stale_14d_window (violations=0) |
| E4 | warn | ✓ PASS | chunk_min_length_8 (short_chunks=0) |
| E5 | halt | ✓ PASS | effective_date_iso_yyyy_mm_dd (non_iso_rows=0) |

### Metric impact (chống trivial)

| Rule mới | Before | After / khi inject | Impact |
|----------|--------|-------------------|--------|
| CR7 (suspicious pattern) | N/A | -1 doc quarantine | Phát hiện legacy doc_id, prevent embedding test data |
| CR8 (invalid exported_at) | N/A | -1 record (future timestamp) | Phòng clock skew poison |
| CR9 (chunk length < 20) | N/A | -1 fragment | Loại bỏ remnant chunk, improve retrieval quality |

---

## 3. Before / after ảnh hưởng retrieval (200–250 từ)

### Kịch bản inject (Sprint 3)

**Run corrupt:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`

**Purpose:** Cố ý lưu giữ policy refund cũ (14 ngày) + skip E3 halt → embed "bad" vector, kiểm tra retrieval quality degrade.

**Manifest markers:** `"no_refund_fix": true, "skipped_validate": true` → demo injection.

### Kết quả định lượng (eval CSV)

| Query ID | eval_clean (GOOD) | eval_bad (inject FAIL) | Impact |
|----------|-------------------|------------------------|--------|
| q_refund_window | top1:"7 ngày", hits_forbidden=NO ✓ | top1:"14 ngày", hits_forbidden=YES ❌ | **Agent trả sai policy (14 vs 7 ngày**) |
| q_leave_version | "12 ngày", hits_forbidden=NO ✓ | "12 hoặc 10 ngày mix", awkward | HR policy ambiguous |
| q_p1_sla | "15 min", hits_forbidden=NO ✓ | (no change, stable) | Baseline ok |
| q_lockout | "5 lần", hits_forbidden=NO ✓ | (no change) | Baseline ok |

**Chứng bằng file:**
- `artifacts/eval/eval_clean.csv` — before fix, 4/4 queries right
- `artifacts/eval/eval_bad.csv` — inject corruption, 2/4 queries wrong (hits_forbidden=yes)

**So sánh:** Corruption effective (2 queries hit forbidden), fix effective (recovery to 4/4 clean).

---

## 4. Freshness & monitoring (100–150 từ)

**SLA đặt:** 24 giờ (exported_at phải không quá 24h so với now)

**Test data (manifest_sprint2.json):**
```
latest_exported_at: 2026-04-15T08:00:00
run_timestamp: 2026-04-15T09:13:01
lag = 1h 13m
```

**Result:** PASS ✓ (1h 13m < 24h, data fresh)

**Tiêu chí:**
- lag < 12h: **PASS ✓** (fresh data, no action)
- 12h ≤ lag < 24h: **WARN ⚠** (aging, monitor)
- lag ≥ 24h: **FAIL ❌** (stale, escalate alert)

**Runbook:** xem `docs/runbook.md` section "Diagnosis" → check manifest age.

---

## 5. Liên hệ Day 09 (50–100 từ)

**Share collection:** Same Chroma `day10_kb` (idempotent upsert per chunk_id).

**Data flow:**
- Day 09 agent query Chroma → retrieve top-k chunks from `day10_kb`
- Chunks từ cleaned Day 10 pipeline (5 docs: policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, access_control_sop)
- Agent answer từ chunks retrieved

**Validation:** Test 4 golden queries (`data/test_questions.json`) → verify contains_expected=yes, hits_forbidden=no

**Impact:** Clean pipeline → accurate retrieval → correct agent answers (fixed refund window 7 vs 14 ngày bug)

---

## 6. Rủi ro & roadmap  

**Known mitigations:**
- Stale version mixin → E3 halt + CR5 quarantine
- Chunk duplicate → CR4 normalization
- Exported future timestamp → CR8 quarantine

**Future improvements:**
- Chroma rollback automation (snapshot + auto-revert)
- Great Expectations framework (instead of custom)
- Real-time monitoring (Prometheus + Grafana)
- Content-hash versioning để detect chunk content change

---

## **Tóm lược kết quả**

✅ **Sprint 1:** Ingest policy_export_dirty.csv (10 records), log run_id, schema map  
✅ **Sprint 2:** Add CR7–CR9 (suspicious, exported_at, chunk_len) + E1–E5, output: 6 cleaned / 4 quarantine  
✅ **Sprint 3:** Inject corruption (--no-refund-fix) → eval_bad.csv (hits_forbidden=yes), demonstrate before/after quality diff  
✅ **Sprint 4:** Docs (pipeline_architecture.md, runbook.md, quality_report.md), freshness PASS, idempotent embed ✓  

**Single command to demo:**
```bash
python etl_pipeline.py run --run-id "team31-final"
# Verify: artifacts/manifests/manifest_team31-final.json + artifacts/cleaned/cleaned_team31-final.csv
```
