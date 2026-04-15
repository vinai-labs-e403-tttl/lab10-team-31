# Runbook — Lab Day 10 (incident tối giản)

---

## Scenario: Agent trả sai chính sách hoàn tiền

### Symptom

**Observed:** Agent Day 09 khi hỏi "Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền?" trả lời lại: **"14 ngày"** (sai) thay vì 7 ngày (đúng).

**Root:** Chunk từ policy_refund_v3 (version cũ, window 14 ngày) bị lẫn trong vector store, agent retrieve được chunk sai.

---

### Detection

**Indicator 1:** Expectation suite run fail
- E3 `refund_no_stale_14d_window` → FAIL: "violations=1: policy_refund_v4 contains '14 ngày làm việc' (sai)"

**Indicator 2:** Retrieval eval shows `hits_forbidden=yes`
- File `artifacts/eval/eval_bad.csv` có hàng refund với `hits_forbidden=yes` (wrong chunk retrieved)

**Indicator 3:** Manifest flag `"no_re Lệnh |
|------|----------|------------------|------|
| 1 | Xem manifest gần nhất | `"no_refund_fix": false` (mặc định) hoặc `"skipped_validate": false` | `cat artifacts/manifests/manifest_*.json \| jq` |
| 2 | Kiểm tra quarantine | Xem có dòng refund bị quarantine (reason=contains_stale_14d) không | `head artifacts/quarantine/quarantine_*.csv` |
| 3 | Xem cleaned CSV | Đoạn refund phải chỉ có "7 ngày", không "14 ngày" | `grep -i "refund\|ngày" artifacts/cleaned/cleaned_*.csv \| head` |
| 4 | Mở file eval test | So sánh `eval_clean.csv` vs `eval_bad.csv` | `diff artifacts/eval/eval_*.csv`
## Diagnosis

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra `artifacts/manifests/*.json` | … |
| 2 | Mở `artifacts/quarantine/*.csv` | … |
| 3 | Chạy `python eval_retrieval.py` | … |

---

## Mitigation
### Immediate (0-5 min)
```bash
# Kiểm tra root cause: manifest flag
cat artifacts/manifests/manifest_sprint2.json | jq '.no_refund_fix'
# Nếu true → someone ran with --no-refund-fix (inject mode)
# Nếu false → pipeline bug hoặc source file sai
```
**Short term (tối nay):**
- CI/CD check: Pull request phải run `python eval_retrieval.py` + check `hits_forbidden=0` trước merge
- Alert: Nếu `quarantine_records > 40%` → WARN vào manifest

**Long term (Day 11+):**
- Implement automated alert khi expectation halt
- Schema version control: Lưu `doc_version` trên mỗi chunk → easy rollback
- Upstream validation: Hệ source phải output doc_id version (policy_refund_v4, không v3)
### Recovery (5-15 min)
```bash
# Rerun pipeline (normal mode)
python etl_pipeline.py run --run-id "hotfix-1"
# Verify: E3 pass, expectation không halt
# Check eval: q_refund_window.contains_expected=yes, hits_forbidden=no
```

### If still fails
- **Check source:** Xem `data/raw/policy_export_dirty.csv` - dòng refund chỉ có "7 ngày"?
- **Check rules:** Xem `transform/cleaning_rules.py` - CR refund fix có enable không?
- **Rollback embed:** Xóa `chroma_db/` (nếu cần reset), rerun sẽ tạo lại
> Rerun pipeline, rollback embed, tạm banner “data stale”, …

---

## Prevention

> Thêm expectation, alert, owner — nối sang Day 11 nếu có guardrail.
