# Sprint 4 — Hoàn thành các Docs & Báo cáo ✅

**Status:** COMPLETED (Sprint 1–4 ALL DONE)  
**Generated:** 2026-04-15  
**Team:** Lab 10 Team 31

---

## 📋 Tóm tắt Sprint 4 — Monitoring, Docs & Reports

### Công việc đã hoàn thành

#### ✅ **Các tài liệu (Docs — 4 files)**

1. **`docs/pipeline_architecture.md`** ✅
   - Sơ đồ Mermaid (raw CSV → clean → validate (E1-E5) → embed (Chroma) → serve Day 09)
   - Bảng trách nhiệm component (Ingest → Transform → Quality → Embed → Monitor)
   - Idempotency strategy: upsert chunk_id bằng SHA256 hash
   - Liên hệ Day 09: cùng Chroma collection `day10_kb`, cùng 5 docs
   - Rủi ro đã biết: stale version mixin, clock skew, lag > SLA

2. **`docs/runbook.md`** ✅
   - Scenario: Agent trả sai refund policy (14 ngày thay 7 ngày)
   - Detection: E3 expectation halt, eval `hits_forbidden=yes`
   - Diagnosis tree: manifest → quarantine → cleaned CSV → eval test
   - Mitigation: Rerun pipeline clean mode, verify E3 pass
   - Prevention: CI/CD check + real-time expectation monitoring

3. **`docs/data_contract.md`** ✅
   - Nguồn dữ liệu: raw CSV export + canonical docs
   - Schema: chunk_id (stable hash), doc_id (allowlist), chunk_text (min 20 chars), effective_date (ISO), exported_at
   - Quarantine rules: unknown doc_id, invalid date, stale HR (< 2026-01-01), suspicious pattern
   - Version canonical: refund_v4 (7 ngày), HR 2026 (12 ngày)

4. **`docs/quality_report.md`** ✅
   - Số liệu: raw 10 → cleaned 6 + quarantine 4
   - Before/after: eval_bad.csv (hits_forbidden=YES) vs eval_clean.csv (hits_forbidden=NO)
   - Freshness: 1h 13m lag < 24h SLA → **PASS ✓**
   - Corruption inject: --no-refund-fix --skip-validate → đặc ý tạo "bad" vector
   - Roadmap: Great Expectations, Prometheus, rollback automation

---

#### ✅ **Báo cáo (Reports — 2 files + template)**

5. **`reports/group_report.md`** ✅
   - **Section 1:** Pipeline overview (CSV → clean 6 records → embed → Chroma serve)
   - **Section 2:** Cleaning rules + expectations (CR1-9: allowlist, date, dedup, refund fix, etc.; E1-5: min_row, doc_id, refund stale, chunk length, date ISO)
   - **Section 3:** Before/after retrieval (inject-bad vs clean — refund window sai vs đúng, impact on agent)
   - **Section 4:** Freshness monitoring (PASS tiêu chí: < 12h = PASS, 12-24h = WARN, > 24h = FAIL)
   - **Section 5:** Day 09 linkage (collection sharing, validation 4 golden queries)
   - **Section 6:** Rủi ro + roadmap (mitigations, future improvements)

6. **`reports/individual/member_template.md`** ✅
   - Template cho từng thành viên
   - Trường: Tên, Vai trò, Sprint, Công việc, Learnings, Challenges, Solutions, PR/commits, Demo

7. **`reports/group_report_final.md`** (backup)

---

#### ✅ **Freshness Monitoring**

**Manifest data (manifest_sprint2.json):**
```json
{
  "run_id": "sprint2",
  "latest_exported_at": "2026-04-15T08:00:00",
  "run_timestamp": "2026-04-15T09:13:01",
  "cleaned_records": 6,
  "quarantine_records": 4
}
```

**Freshness result:**
- Lag = 1h 13m
- SLA = 24h
- **Status: PASS ✓** (data fresh)

---

### Artifacts từ Sprint 1–3 (tham chiếu)

| Sprint | Artifact | Path |
|--------|----------|------|
| 1 | Raw CSV | `data/raw/policy_export_dirty.csv` (10 records) |
| 2 | Cleaned CSV | `artifacts/cleaned/cleaned_sprint2.csv` (6 records) |
| 2 | Quarantine CSV | `artifacts/quarantine/quarantine_sprint2.csv` (4 records) |
| 2 | Manifest (clean) | `artifacts/manifests/manifest_sprint2.json` |
| 3 | Manifest (inject) | `artifacts/manifests/manifest_inject-bad.json` |
| 3 | Eval clean | `artifacts/eval/eval_clean.csv` (4 queries, hits_forbidden=NO) |
| 3 | Eval bad | `artifacts/eval/eval_bad.csv` (2 queries hit forbidden) |

---

### 📋 Checklist hoàn thành Sprint 4

**Docs (bắt buộc có):**
- [x] pipeline_architecture.md — sơ đồ + bảng + idempotency + Day 09 + risks
- [x] runbook.md — scenario + detection + diagnosis + mitigation + prevention
- [x] data_contract.md — schema + quarantine rules + canonical version
- [x] quality_report.md — số liệu + before/after + freshness + inject demo + improvements

**Reports (bắt buộc có):**
- [x] group_report.md — 6 sections (overview, cleaning, before/after, freshness, Day09, risks)
- [x] individual/member_template.md — template cho từng member

**Monitoring:**
- [x] Freshness check logic (manifest age vs SLA) → **PASS ✓**
- [x] Runbook có incident response flow

**Pipeline ready:**
- [x] `python etl_pipeline.py run` → clean mode (E3 PASS, embed success)
- [x] `python etl_pipeline.py run --no-refund-fix --skip-validate` → inject demo mode
- [x] `python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json` → PASS

---

## 🚀 Hướng dẫn sử dụng

### 1. Xem kiến trúc pipeline
```bash
# Mở docs/pipeline_architecture.md
# Mermaid diagram: raw → clean → validate → embed → serve
# Bảng component ownership + idempotency strategy
```

### 2. Kiểm tra incident response
```bash
# Xem docs/runbook.md
# Scenario: Agent trả sai refund policy
# Diagnosis → Mitigation → Prevention
```

### 3. Xem báo cáo chi tiết
```bash
# Xem reports/group_report.md
# Được gồm: pipeline overview, cleaning+expectation, before/after retrieval, 
# freshness check, Day 09 linkage, risks & roadmap
```

### 4. Chạy pipeline demo

**Normal run (clean data):**
```bash
python etl_pipeline.py run --run-id "final-clean-demo"
# Output: 6 cleaned, 4 quarantine, E3 PASS, idempotent embed
```

**Inject corruption demo (Sprint 3 showcase):**
```bash
python etl_pipeline.py run --run-id "final-bad-demo" --no-refund-fix --skip-validate
# Output: 6 cleaned (chứa "14 ngày" sai), E3 skip, embed with bad data
```

**Check freshness:**
```bash
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json
# Output: 1h 13m lag < 24h SLA → PASS ✓
```

### 5. Verify artifacts
```bash
# Cleaned records
cat artifacts/cleaned/cleaned_sprint2.csv | head -5

# Quarantined records
cat artifacts/quarantine/quarantine_sprint2.csv

# Manifest metadata
cat artifacts/manifests/manifest_sprint2.json | jq

# Eval results (before/after)
cat artifacts/eval/eval_clean.csv | column -t -s','
cat artifacts/eval/eval_bad.csv | column -t -s','
```

---

## 📚 File structure

```
lab10-team-31/
├── docs/
│   ├── pipeline_architecture.md ✅ (sơ đồ + các thành phần)
│   ├── runbook.md ✅ (incident response)
│   ├── data_contract.md ✅ (schema + quarantine rules)
│   └── quality_report.md ✅ (số liệu + before/after + freshness)
│
├── reports/
│   ├── group_report.md ✅ (6 sections)
│   ├── group_report_final.md (backup)
│   └── individual/
│       ├── member_template.md ✅ (template cho individual reports)
│       └── template.md (cũ)
│
├── artifacts/
│   ├── cleaned/
│   │   └── cleaned_sprint2.csv (6 records)
│   ├── quarantine/
│   │   └── quarantine_sprint2.csv (4 records)
│   ├── manifests/
│   │   ├── manifest_sprint2.json (clean)
│   │   └── manifest_inject-bad.json (corruption)
│   └── eval/
│       ├── eval_clean.csv (before fix)
│       └── eval_bad.csv (inject corruption)
│
├── etl_pipeline.py (entrypoint)
├── transform/cleaning_rules.py (CR1-9)
├── quality/expectations.py (E1-5)
├── monitoring/freshness_check.py
├── contracts/data_contract.yaml
├── data/
│   ├── raw/policy_export_dirty.csv
│   ├── docs/ (5 canonical docs)
│   └── test_questions.json (4 golden queries)
│
├── README.md (project overview - keep)
├── SPRINT4_COMPLETION.md (Sprint 4 summary)
└── requirements.txt
```

---

## ✅ Final Verification

Chạy lệnh này để verify tất cả Sprint 4 deliverables:

```bash
echo "=== Checking Docs ==="
ls -la docs/pipeline_architecture.md docs/runbook.md docs/data_contract.md docs/quality_report.md

echo "=== Checking Reports ==="
ls -la reports/group_report.md reports/individual/member_template.md

echo "=== Checking Artifacts ==="
find artifacts -type f \( -name "*.csv" -o -name "*.json" \) | sort

echo "=== Running Pipeline Demo ==="
python etl_pipeline.py run --run-id "verify-sprint4"

echo "=== Checking Freshness ==="
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json

echo "✅ Sprint 4 COMPLETE!"
```

---

## 📝 Notes

- **Chroma DB:** `.gitignore` → không commit thư mục `chroma_db/`
- **Manifest & Eval:** BẮT BUỘC commit (chứng cứ kết quả)
- **Report format:** Markdown — dễ đọc trên GitHub
- **Individual reports:** Template ready, mỗi thành viên điền theo vai trò
- **Next step:** Student teams submit group_report.md + individual reports + artifacts

---

## 🎉 Summary

**Sprint 4 hoàn thành 100%:**
- ✅ 4 key docs (architecture, runbook, contract, quality report)
- ✅ Group report (6 sections) + individual template
- ✅ Freshness monitoring (PASS)
- ✅ All artifacts available (cleaned, quarantine, manifests, eval)
- ✅ Pipeline demo ready (clean + inject modes)

**Team ready to submit!** 

---

Generated: 2026-04-15  
Last updated: 2026-04-15
