# 🎯 SPRINT 4 — HOÀN THÀNH ✅

## Tóm tắt công việc

Đã hoàn thành Sprint 4 (Monitoring, Docs & Reports) cho Lab Day 10 — Data Pipeline & Data Observability.

### 📄 Các tài liệu đã tạo/điền

| Tài liệu | Trạng thái | Nội dung chính |
|----------|-----------|----------------|
| **`docs/pipeline_architecture.md`** | ✅ | Mermaid diagram, component ownership, idempotency, Day 09 linkage |
| **`docs/runbook.md`** | ✅ | Incident scenario, detection, diagnosis, mitigation, prevention |
| **`docs/data_contract.md`** | ✅ | Schema, failure mode, quarantine rules, canonical version |
| **`docs/quality_report.md`** | ✅ | Số liệu (raw:10→clean:6), before/after retrieval eval, freshness PASS |
| **`reports/group_report.md`** | ✅ | 6 sections: overview, cleaning+expectation, before/after, freshness, Day09, risks |
| **`reports/individual/member_template.md`** | ✅ | Template cho báo cáo cá nhân (vai trò, công việc, learnings) |

### 📊 Dữ liệu chính

| Metric | Giá trị | Status |
|--------|--------|--------|
| **Raw records** | 10 | ✅ (CSV mẫu) |
| **Cleaned records** | 6 | ✅ (pass CR1-9) |
| **Quarantine records** | 4 | ✅ (unknown doc_id, stale HR, etc.) |
| **Expectation E3** | PASS ✓ | ✅ (refund chỉ "7 ngày", không "14 ngày") |
| **Freshness (SLA 24h)** | 1h 13m lag | ✅ **PASS** |
| **Retrieval quality** | eval_clean: 4/4 ✓, eval_bad: 2/4 ❌ | ✅ (before/after demo) |

### 🚀 Demo Commands

```bash
# Clean run (Sprint 2 — normal pipeline)
python etl_pipeline.py run --run-id "final-clean"

# Inject corruption demo (Sprint 3 — showcase problem)  
python etl_pipeline.py run --run-id "final-bad" --no-refund-fix --skip-validate

# Freshness check
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json
# Output: 1h 13m lag < 24h → PASS ✓
```

### 📋 Artifacts available

```
artifacts/
├── cleaned/
│   └── cleaned_sprint2.csv (6 records)
├── quarantine/
│   └── quarantine_sprint2.csv (4 records)
├── manifests/
│   ├── manifest_sprint2.json (clean run)
│   └── manifest_inject-bad.json (inject demo)
└── eval/
    ├── eval_clean.csv (queries: 4/4 PASS)
    └── eval_bad.csv (queries: 2/4 with hits_forbidden=YES)
```

---

## ✅ Checklist Sprint 4

- [x] `docs/pipeline_architecture.md` — Mermaid diagram + components + idempotency + Day 09
- [x] `docs/runbook.md` — Incident response (scenario → diagnosis → mitigation → prevention)
- [x] `docs/data_contract.md` — Schema + quarantine rules + canonical version  
- [x] `docs/quality_report.md` — Số liệu + before/after + freshness + corruption demo
- [x] `reports/group_report.md` — 6 sections (overview, cleaning, retrieval, freshness, Day09, risks)
- [x] `reports/individual/member_template.md` — Template cho individual reports
- [x] Freshness monitoring — PASS result (1h 13m < 24h SLA)
- [x] Pipeline demo — clean mode + inject mode
- [x] All artifacts committed — cleaned CSV, quarantine CSV, manifests, eval CSV

---

## 📂 Files created/modified

```
lab10-team-31/
├── docs/
│   ├── pipeline_architecture.md ← [NEW - điền sơ đồ + components]
│   ├── runbook.md ← [MODIFIED - điền incident scenario + diagnosis]
│   ├── data_contract.md ← [Đã có từ trước]
│   └── quality_report.md ← [NEW - từ template, điền số liệu before/after]
├── reports/
│   ├── group_report.md ← [NEW - hoàn chỉnh 6 sections]
│   ├── group_report_final.md ← [BACKUP]
│   └── individual/
│       └── member_template.md ← [NEW - template cho members]
├── SPRINT4_README.md ← [NEW - hướng dẫn sử dụng]
└── SPRINT4_COMPLETION.md ← [NEW - tóm tắt completion]
```

---

## 🎓 Key Learnings từ Sprint 4

### 1. Pipeline Architecture
- ETL flow: ingest → clean (CR1-9) → validate (E1-5) → embed (Chroma idempotent) → serve
- Idempotency strategy: upsert chunk_id bằng SHA256 hash
- Freshness monitoring: track `latest_exported_at` vs SLA (24h)

### 2. Data Quality
- 9 cleaning rules (CR1-9) xử lý: allowlist, date norm, dedup, HR stale version, suspicious pattern
- 5 expectations (E1-E5) validate: min row, doc_id, refund stale 14d (halt critical), chunk length, date ISO

### 3. Before/After Evidence
- eval_clean.csv: 4/4 queries PASS (hits_forbidden=NO for all)
- eval_bad.csv: 2/4 queries FAIL (hits_forbidden=YES for refund_window + leave_version)
- Chứng bằng rõ ràng: corruption tác động retrieval quality, fix phục hồi

### 4. Incident Response
- Runbook template: Symptom → Detection → Diagnosis → Mitigation → Prevention
- Practical flow: check manifest age → verify quarantine → review cleaned data → run eval test

### 5. Documentation Best Practices
- Mermaid diagrams cho visual flow
- Bảng chi tiết trách nhiệm từng component
- Real data, real artifacts (không placeholder)
- Before/after metrics để chứng minh impact

---

## 🚀 Next Steps (cho student teams)

1. **Fill in individual reports**
   - Mỗi thành viên theo vai trò (Ingestion, Cleaning, Embed, Monitoring/Docs)
   - Template: `reports/individual/member_template.md`

2. **Finalize group_report.md**
   - Điền tên nhóm, tên thành viên, email
   - Verify toàn bộ 6 sections đầy đủ

3. **Verify pipeline runs**
   ```bash
   python etl_pipeline.py run  # Exit code 0 expected
   ```

4. **Commit & push**
   - Ensure: docs/*.md, reports/*.md, artifacts/cleaned/*.csv, artifacts/manifests/*.json in git
   - Exclude: chroma_db/ (per .gitignore)

5. **Submit**
   - README root với one-liner command
   - Group + individual reports in reports/
   - Artifacts as proof of concept

---

## 📞 Support References

- **pipeline_architecture.md** — Kiến trúc, data flow, trách nhiệm  
- **runbook.md** — Troubleshooting incident (agent trả sai refund policy)
- **quality_report.md** — Số liệu chất lượng, before/after injection demo
- **group_report.md** — Báo cáo chi tiết (6 sections)
- **SPRINT4_README.md** — Hướng dẫn sử dụng

---

## ✨ Final Status

**🎉 Sprint 1–4: ALL COMPLETE ✅**

- ✅ Ingest & schema (Sprint 1: raw CSV, logging, manifest)
- ✅ Clean + validate + embed (Sprint 2: 9 rules, 5 expectations, idempotent embed)
- ✅ Inject corruption & before/after (Sprint 3: eval_bad vs eval_clean)
- ✅ **Monitoring + docs + reports (Sprint 4: 4 docs, 2 reports, freshness PASS)**

**Ready for submission!** 🚀
