# ✅ SPRINT 4 — HOÀN THÀNH TOÀN BỘ

## 🎉 Tóm tắt công việc

Đã hoàn thành **Sprint 4** (Monitoring, Docs & Reports) cho Lab Day 10 — etl_pipeline + data_observability.

---

## 📄 Các tài liệu đã tạo

### **4 Tài liệu chính (Docs)**

1. ✅ **`docs/pipeline_architecture.md`**
   - Mermaid diagram (raw CSV → clean → validate → embed → Chroma serve)
   - Bảng trách nhiệm component (Ingest, Transform, Quality, Embed, Monitor)
   - Idempotency strategy (chunk_id upsert, stable SHA256 hash)
   - Liên hệ Day 09 (collection sharing, validation 4 golden queries)
   - Rủi ro đã biết + mitigation

2. ✅ **`docs/runbook.md`**
   - Scenario incident: Agent trả sai refund policy (14 ngày vs 7 ngày)
   - Detection: E3 expectation halt, eval `hits_forbidden=yes`
   - Diagnosis tree: 4-step flow (manifest → quarantine → cleaned → eval)
   - Mitigation: Rerun clean, verify E3 pass
   - Prevention: CI/CD check + expectation monitoring

3. ✅ **`docs/data_contract.md`**
   - Schema: chunk_id (stable hash), doc_id (allowlist), chunk_text, effective_date, exported_at
   - Nguồn dữ liệu: raw CSV export + canonical docs
   - Quarantine rules: unknown doc_id, invalid date, stale HR (< 2026-01-01), suspicious pattern
   - Canonical version: refund_v4 (7 ngày), HR 2026 (12 ngày)

4. ✅ **`docs/quality_report.md`**
   - Số liệu: raw 10 records → cleaned 6 + quarantine 4
   - Before/after retrieval: eval_bad.csv (hits_forbidden=YES) vs eval_clean.csv (hits_forbidden=NO)
   - Freshness check: 1h 13m lag < 24h SLA → **PASS ✓**
   - Corruption inject demo: `--no-refund-fix --skip-validate` cố ý tạo bad vector
   - Roadmap: Great Expectations, Prometheus, rollback automation

### **2 Báo cáo (Reports)**

5. ✅ **`reports/group_report.md`**
   - Section 1: Pipeline overview (flow, command, run_id)
   - Section 2: Cleaning + expectation (CR1-9 rules, E1-5 expectations, metric_impact table)
   - Section 3: Before/after retrieval (inject-bad vs clean, refund policy demo)
   - Section 4: Freshness & monitoring (SLA 24h, PASS/WARN/FAIL criteria)
   - Section 5: Day 09 linkage (collection sharing, validation rules)
   - Section 6: Rủi ro & roadmap (mitigations, improvements)

6. ✅ **`reports/individual/member_template.md`**
   - Template cho từng thành viên báo cáo cá nhân
   - Trường: Tên, Vai trò, Sprint, Công việc, Learnings, Challenges, Solutions, PR/commits

### **3 Hướng dẫn bổ sung (Support)**

7. ✅ **`SPRINT4_README.md`** — Hướng dẫn sử dụng toàn bộ Sprint 4
8. ✅ **`SPRINT4_SUMMARY.md`** — Tóm tắt cuối cùng  
9. ✅ **`SPRINT4_COMPLETION.md`** — Checklist hoàn thành

---

## 📊 Dữ liệu chính (Data Summary)

| Metric | Giá trị | Status |
|--------|--------|--------|
| **Raw records** | 10 | ✓ (CSV mẫu: policy_export_dirty.csv) |
| **Cleaned records** | 6 | ✓ (Pass 9 cleaning rules CR1-9) |
| **Quarantine records** | 4 | ✓ (unknown doc_id, stale HR, etc.) |
| **Expectation E3** | PASS ✓ | ✓ (refund chỉ "7 ngày", không "14 ngày") |
| **Freshness SLA** | PASS ✓ | ✓ (1h 13m lag < 24h) |
| **Retrieval quality** | Before/After | ✓ (eval_clean 4/4, eval_bad 2/4) |

---

## 🚀 Demo Commands

```bash
# ✅ Normal run (clean data)
python etl_pipeline.py run --run-id "final-clean"
# Expected: 6 cleaned, 4 quarantine, E3 PASS, embed success

# ✅ Inject corruption demo (showcase problem)
python etl_pipeline.py run --run-id "final-bad" --no-refund-fix --skip-validate
# Expected: 6 cleaned (với "14 ngày" sai), E3 skip, bad embed

# ✅ Freshness check
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json
# Expected: PASS (1h 13m < 24h SLA)
```

---

## 📋 Artifacts Available

```
artifacts/
├── cleaned/
│   └── cleaned_sprint2.csv ...................... 6 records (clean data)
├── quarantine/
│   └── quarantine_sprint2.csv ................... 4 records (rejected)
├── manifests/
│   ├── manifest_sprint2.json .................... Run metadata (clean)
│   └── manifest_inject-bad.json ................. Run metadata (injection)
└── eval/
    ├── eval_clean.csv ........................... 4 queries, 4/4 PASS
    └── eval_bad.csv ............................. 4 queries, 2/4 with hits_forbidden
```

---

## ✅ Complete Checklist

**Docs (bắt buộc):**
- [x] pipeline_architecture.md — diagrams + components + idempotency
- [x] runbook.md — incident response workflow
- [x] data_contract.md — schema + quarantine + version canonical
- [x] quality_report.md — metrics + before/after + freshness

**Reports (bắt buộc):**
- [x] group_report.md — 6 sections (overview → risks)
- [x] individual/member_template.md — template for members

**Monitoring:**
- [x] Freshness check — PASS (1h 13m < 24h)
- [x] Runbook incident response — documented

**Pipeline:**
- [x] Clean run (E3 PASS, embed idempotent)
- [x] Inject demo (corruption showcase)
- [x] Artifacts committed (cleaned, quarantine, manifest, eval)

---

## 📂 File Overview

```
lab10-team-31/
│
├── docs/ ................................................. Documentation  
│   ├── pipeline_architecture.md ..................... ✅ Diagram + components
│   ├── runbook.md .................................... ✅ Incident response
│   ├── data_contract.md .............................. Schema + rules
│   └── quality_report.md ............................. ✅ Metrics + before/after
│
├── reports/ ............................................. Báo cáo
│   ├── group_report.md ................................ ✅ 6 sections
│   └── individual/
│       └── member_template.md ........................ ✅ Individual template
│
├── SPRINT4_README.md ................................... ✅ Hướng dẫn sử dụng
├── SPRINT4_SUMMARY.md .................................. ✅ Tóm tắt
├── SPRINT4_COMPLETION.md ............................... ✅ Checklist
│
├── etl_pipeline.py ..................................... Entrypoint
├── transform/cleaning_rules.py ........................ CR1-9 logic
├── quality/expectations.py ............................ E1-5 logic
├── monitoring/freshness_check.py ..................... SLA monitoring
│
└── artifacts/ ........................................... Data artifacts
    ├── cleaned/*.csv .................................... Cleaned data
    ├── quarantine/*.csv .................................. Rejected records
    ├── manifests/*.json .................................. Run metadata
    └── eval/*.csv ....................................... Retrieval quality
```

---

## 🎓 Key Results

### Before/After comparison:

| Aspect | Before (inject-bad) | After (sprint2) | Impact |
|--------|-------------------|-----------------|--------|
| Refund window | "14 ngày" ❌ | "7 ngày" ✓ | Agent trả đúng |
| HR leave version | Mix 2025/2026 | Only 2026 ✓ | Consistent policy |
| Retrieval hits_forbidden | Some queries YES | All queries NO | Quality improved |
| Freshness lag | 1h 13m | PASS ✓ | Data fresh |
| E3 expectation | Halt skip | PASS | Validation strict |

---

## 🚀 Next Steps (for student teams)

1. **Fill individual reports** — each member fills `reports/individual/[name].md`
2. **Verify group_report.md** — check team name + member names + all 6 sections
3. **Run final demo** — `python etl_pipeline.py run` (exit 0 expected)
4. **Commit & push** — all docs + artifacts to git
5. **Submit** — group report + individual reports + evidence

---

## 📞 Documentation Map

| Question | File |
|----------|------|
| "Pipeline hoạt động như thế nào?" | `docs/pipeline_architecture.md` (Mermaid diagram) |
| "Có lỗi gì thì phải làm sao?" | `docs/runbook.md` (incident response) |
| "Data schema là gì?" | `docs/data_contract.md` (schema + quarantine) |
| "Chất lượng dữ liệu tốt không?" | `docs/quality_report.md` (metrics + before/after) |
| "Kết quả cuối cùng là gì?" | `reports/group_report.md` (6 sections overview) |
| "Thiết kế tổng quan?" | `SPRINT4_README.md` (usage guide) |

---

## ✨ Final Status

```
╔════════════════════════════════════════╗
║  SPRINT 1-4: ALL COMPLETE ✅           ║
╠════════════════════════════════════════╣
║ ✅ Sprint 1: Ingest & schema           ║
║ ✅ Sprint 2: Clean + validate + embed  ║
║ ✅ Sprint 3: Inject & before/after     ║
║ ✅ Sprint 4: Monitoring + docs + repo  ║
╚════════════════════════════════════════╝

Ready for submission! 🚀
```

---

## 📝 Notes

- **Chroma DB:** Không commit (`.gitignore`)
- **Artifacts:** BẮT BUỘC commit (cleaned, quarantine, manifests, eval) — chứng cứ kết quả
- **Format:** Markdown — dễ đọc + reviewable on GitHub
- **Pipeline:** Working `python etl_pipeline.py run` command ready
- **Freshness:** PASS (1h 13m lag < 24h SLA)

---

Generated: **2026-04-15** ✅  
Lab: **Day 10 — Data Pipeline & Data Observability**  
Team: **Lab 10 Team 31**

**Status: COMPLETE & READY FOR SUBMISSION** 🎉
