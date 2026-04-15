# Quality report — Lab Day 10 (nhóm)

**run_id:** sprint2 (clean run) + inject-bad (Sprint 3 corruption)  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (Sprint 1 raw) | Sau clean (Sprint 2) | Ghi chú |
|--------|--------|-----|----|
| raw_records | 10 | - | CSV mẫu: policy_export_dirty.csv |
| cleaned_records | - | 6 | Pass cleaning rules CR1-CR9 |
| quarantine_records | - | 4 | Unknown doc_id (1), stale HR (2), duplicate (1) |
| Expectation E3 `refund_no_stale_14d_window` | - | PASS ✓ | Sau apply CR fix, không halt |
| Chroma embed | - | ✓ Idempotent upsert | 6 chunks × 5 splits ≈ 30 vectors |

---

## 2. Before / after retrieval (bắt buộc)

**Câu hỏi then chốt:** `q_refund_window` — "Khách hàng có bao nhiêu ngày để yêu cầu hoàn tiền?"

### Before (Sprint 3 inject corruption — `--no-refund-fix --skip-validate`)
```
File: artifacts/eval/eval_bad.csv
Kết quả: policy_refund_v4 | "...14 ngày làm việc..." | contains_expected=yes | hits_forbidden=YES ❌
```
**Issue:** top1_preview trả "14 ngày" (sai), agent sẽ kết luận sai.

### After (Sprint 2 clean run — no flags, E3 pass)  
```
File: artifacts/eval/eval_clean.csv
Kết quả: policy_refund_v4 | "...7 ngày làm việc..." | contains_expected=yes | hits_forbidden=NO ✓
```
**Fix:** CR refund fix + E3 halt + Chroma prune → trả "7 ngày" đúng.

---

### Merit: `q_leave_version` — HR policy 2026 vs 2025
**Result:** Quarantine 2 records với effective_date < 2026-01-01 → clean run chỉ lấy latest version (12 ngày 2026).  
**Evidence:** Bảng `metric_impact` trong group_report.md ghi rõ impact trên quarantine_records.

---

## 3. Freshness & monitor

**Manifest (sprint2):** latest_exported_at = 2026-04-15T08:00:00, run_timestamp = 2026-04-15T09:13:01 = ~1h 13m lag  
**SLA:** 24h → **PASS ✓** (dữ liệu fresh)

**Nếu lag > 12h:** WARN  
**Nếu lag > 24h:** FAIL (stale data, alert team)

**Run:** `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate`

**Corruption type:** Bỏ CR fix refund + skip E3 halt → 14 ngày policy vẫn embed  
**Chứng bằng:** `eval_bad.csv` có hàng refund với `hits_forbidden=yes` vs `eval_clean.csv` có `hits_forbidden=no`  
**Impact:** Retrieval kém (agent trả sai chính sách)

## 4. Corruption inject (Sprint 3)

> Mô tả cố ý làm hỏng dữ liệu kiểu gì (duplicate / stale / sai format) và cách phát hiện.

--Custom expectation (không Great Expectations) → cải thiện: dùng GE framework
- Embedding model hard-coded → cải thiện: tham số hóa config
- Monitoring thủ công → cải thiện: Prometheus + Grafana real-time
- Rollback Chroma manual → cải thiện: snapshot + auto-rollback script

## 5. Hạn chế & việc chưa làm

- …
