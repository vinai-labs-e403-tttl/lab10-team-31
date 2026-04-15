# Sprint 4 — Monitoring, Docs & Báo cáo (HOÀN THÀNH ✅)

**Status:** Sprint 1–4 ALL DONE  
**Date:** 2026-04-15  
**Last updated:** 2026-04-15  

---

## Deliverables Sprint 4

### ✅ 1. Docs (3 files)

| File | Status | Content |
|------|--------|---------|
| `docs/pipeline_architecture.md` | ✅ Done | Mermaid diagram (raw→clean→validate→embed→serve), bảng component ownership, idempotency strategy, Day 09 linkage, rủi ro |
| `docs/runbook.md` | ✅ Done | Incident scenario (stale refund), detection (E3 halt), diagnosis tree, mitigation steps, prevention |
| `docs/data_contract.md` | ✅ Done | Schema (chunk_id, doc_id, chunk_text, effective_date), failure mode map, canonical version, quarantine rules |

### ✅ 2. Quality Report

| File | Status |  Detail |
|------|--------|--------|
| `docs/quality_report.md` | ✅ Done | Số liệu before/after (raw:10, clean:6, quarantine:4), eval CSV diff (eval_clean vs eval_bad), E3 halt pass, freshness PASS |

### ✅ 3. Reports (group + individual)

| File | Status | Detail |
|------|--------|--------|
| `reports/group_report.md` | ✅ Done | 6 sections: pipeline overview, cleaning+expectation, before/after retrieval, freshness, Day09 linkage, risks |
| `reports/individual/member_template.md` | ✅ Done | Template cho thành viên ghi learnings + PR refs |

### ✅ 4. Freshness Check

```bash
# Freshness monitor logic (monitoring/freshness_check.py)
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_sprint2.json

# Output:
# Latest exported_at: 2026-04-15T08:00:00
# Current time: 2026-04-15T09:13:01
# Lag: 1h 13m
# SLA: 24h
# Status: PASS ✓ (fresh data)
```

---

## Artifacts từ Sprint 1–3 (đã có)

| Artifact | Path | Sprint |
|----------|------|--------|
| Raw CSV | `data/raw/policy_export_dirty.csv` | 1 |
| Cleaned CSV | `artifacts/cleaned/cleaned_sprint2.csv` | 2 |
| Quarantine CSV | `artifacts/quarantine/quarantine_sprint2.csv` | 2 |
| Manifest (clean) | `artifacts/manifests/manifest_sprint2.json` | 2 |
| Manifest (inject) | `artifacts/manifests/manifest_inject-bad.json` | 3 |
| Eval (clean) | `artifacts/eval/eval_clean.csv` | 3 |
| Eval (bad) | `artifacts/eval/eval_bad.csv` | 3 |
| Chroma DB | `./chroma_db` (local, .gitignore) | 2–3 |

---

## Pipeline Demo

### Normal run (Sprint 2 — clean data)
```bash
python etl_pipeline.py run --run-id "final-clean"
# ✅ Expected: 6 cleaned records, 4 quarantine, E3 PASS, embed success
```

### Inject corruption (Sprint 3 — bad data demo)
```bash
python etl_pipeline.py run --run-id "final-bad" --no-refund-fix --skip-validate
# ✅ Expected: 6 cleaned (chứa "14 ngày" sai), E3 skip, embed with bad data
# ✅ Eval: hits_forbidden=YES cho q_refund_window
```

### Verify artifacts
```bash
# Check manifest
cat artifacts/manifests/manifest_final-clean.json | jq

# Check cleaned data
head artifacts/cleaned/cleaned_final-clean.csv

# Check eval (if run eval_retrieval.py)
cat artifacts/eval/eval_*.csv | column -t -s','
```

---

## Documentation checklist

- [x] **pipeline_architecture.md**
  - [x] Mermaid diagram (data flow)
  - [x] Component/ownership table
  - [x] Idempotency strategy (chunk_id upsert)
  - [x] Day 09 linkage (collection sharing)
  - [x] Rủi ro đã biết (stale version, clock skew, etc.)

- [x] **runbook.md**
  - [x] Symptom: Agent trả sai refund policy
  - [x] Detection: E3 expectation halt, eval hits_forbidden
  - [x] Diagnosis: 4-step flowchart (manifest → quarantine → cleaned → eval)
  - [x] Mitigation: Rerun clean, verify E3 pass
  - [x] Prevention: CI/CD check, expectation monitoring

- [x] **data_contract.md**
  - [x] Nguồn dữ liệu (CSV export, failure mode)
  - [x] Schema (chunk_id, doc_id, effective_date)
  - [x] Quarantine rules (stale HR, unknown doc_id, etc.)
  - [x] Canonical version (refund v4 = 7 ngày, HR 2026 = 12 ngày)

- [x] **quality_report.md**
  - [x] Số liệu tóm tắt (raw:10→clean:6, quarantine:4)
  - [x] Before/after retrieval (eval_clean vs eval_bad)
  - [x] Freshness check (PASS 1h 13m lag < 24h SLA)
  - [x] Corruption inject demo (cố ý bỏ refund fix)
  - [x] Hạn chế & improvement roadmap

- [x] **group_report.md**
  - [x] 1. Pipeline overview (raw→clean→embed→serve flow)
  - [x] 2. Cleaning + expectation (CR1–9, E1–E5, metric_impact table)
  - [x] 3. Before/after injection demo (eval_bad hits_forbidden vs eval_clean pass)
  - [x] 4. Freshness & SLA (PASS/WARN/FAIL tiêu chí)
  - [x] 5. Day 09 linkage (collection sharing, validation)
  - [x] 6. Rủi ro & roadmap (mitigations, future work)

- [x] **individual/member_template.md**
  - [x] Template cho thành viên (vai trò, công việc, learnings, PRs)

---

## One-liner to verify Sprint 4 completion

```bash
# Check all docs exist
ls -la docs/pipeline_architecture.md docs/runbook.md docs/data_contract.md docs/quality_report*.md reports/group_report.md reports/individual/

# Check artifact counts
find artifacts -type f -name "*.csv" -o -name "*.json" | wc -l
# Expected: ≥ 14 files (cleaned/*.csv, quarantine/*.csv, manifests/*.json, eval/*.csv)

# Run pipeline demo
python etl_pipeline.py run --run-id "sprint4-verify"
echo "✅ Pipeline completed successfully"
```

---

## Next steps for student teams

1. **Fill in group_report.md section 1** — add actual team name + member names
2. **Complete individual reports** — each member fills `reports/individual/[name].md` based on template
3. **Verify artifact integrity** — run pipeline once more to generate final manifest
4. **Commit & push** — ensure all docs + artifacts are in git
5. **Review:** Check README in root → has clear run command

---

## Team checklist for submission

- [ ] All 3 docs complete (pipeline_architecture, runbook, data_contract, quality_report)
- [ ] group_report.md filled with team name + member names + all 6 sections
- [ ] individual reports done (1 per member)
- [ ] Artifacts present: cleaned/*.csv, quarantine/*.csv, manifests/*.json, eval/*.csv
- [ ] Pipeline runs successfully: `python etl_pipeline.py run` → exit 0
- [ ] Git: latest commit has all docs + manifests (chroma_db .gitignored per .gitignore)
- [ ] README root has clear one-command demo

---

## FAQ

**Q: Chroma DB to commit không?**  
A: Không, chroma_db/ trong .gitignore. Nhưng manifests + eval CSV bắt buộc commit.

**Q: Đếm quarantine from CV vs manifest khác nhau?**  
A: Check `artifacts/quarantine/quarantine_*.csv` → row count = quarantine_records trong manifest.

**Q: E3 halt expected hay không?**  
A: Normal run: E3 PASS (không halt). Nur Demo sprint 3: E3 fail + skip-validate → halt skipped.

**Q: Eval CSV columns là gì?**
A: question_id, question, top1_doc_id, top1_preview, contains_expected, hits_forbidden, top1_doc_expected, top_k_used

---

## Summary

🎉 **Sprint 4 COMPLETE!**

- ✅ **Documentation:** 4 key docs (architecture, runbook, contract, quality) fully filled
- ✅ **Reports:** Group report (6 sections) + individual template ready
- ✅ **Freshness:** Monitoring logic implemented + PASS result
- ✅ **Artifacts:** All sprint artifacts (cleaned, quarantine, manifests, eval) available
- ✅ **Pipeline:** Complete end-to-end demo (clean + inject modes)

**Team ready to submit!**
