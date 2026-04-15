# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trần Kiên Trường
**Vai trò:** Ingestion / Cleaning / Embed / Monitoring — Cleaning
**Ngày nộp:** 15/04/2026
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `transform/cleaning_rules.py` — 9 cleaning rules (CR1–CR9), đặc biệt CR7, CR8, CR9
- `quality/expectations.py` — 8 expectations (E1–E8), kiểm tra data quality sau clean
- Kết nối với Ingestion Owner (raw CSV) và Embed Owner (cleaned CSV → Chroma upsert)

**Kết nối với thành viên khác:**

- Ingestion Owner (Trịnh Ngọc Tú): nhận raw CSV → tôi clean và quarantine
- Embed Owner (Đặng Quang Minh): cleaned CSV được dùng để embed vào Chroma
- Monitoring Owner (Đặng Quang Minh): expectations và freshness manifest tôi hỗ trợ kiểm tra

**Bằng chứng (commit / comment trong code):**

Commit `93cdf38` "Sprint 2: Add 3 cleaning rules and 2 expectations with measurable impact":
- CR7 quarantines `legacy_catalog_xyz_zzz` với reason `suspicious_doc_id_prefix`
- CR9 quarantines fragment data (< 20 chars)
- E1–E8 kiểm tra data quality với halt/warn severity

---

## 2. Một quyết định kỹ thuật (100–150 từ)

**Quyết định:** CR7 kiểm tra suspicious doc_id pattern **trước** allowlist check, không phải sau.

```python
# transform/cleaning_rules.py:100–102
if _SUSPICIOUS_DOC_ID_PATTERN.match(doc_id):
    quarantine.append({**raw, "reason": "suspicious_doc_id_prefix"})
    continue
```

**Lý do:** Nếu check allowlist trước, doc_id `legacy_catalog_xyz_zzz` sẽ bị quarantine với reason `unknown_doc_id` — chung chung, không phân biệt được đây là test data hay production data thật. Thứ tự đúng: suspicious check → allowlist check → quarantine với specific reason.

**Kết quả:** Row 9 có `legacy_catalog_xyz_zzz` được quarantine với reason `suspicious_doc_id_prefix` (cụ thể), giúp phân biệt test data với production data lạ. Metric impact: **-1 record quarantine** cho mỗi suspicious pattern detected.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

**Triệu chứng:** Sau khi chạy `eval_retrieval.py`, câu `q_refund_window` trả về **top1:"14 ngày"** với `hits_forbidden=YES` khi chạy `--no-refund-fix --skip-validate`.

**Metric phát hiện:** `eval_bad.csv` — cột `hits_forbidden=YES` cho `q_refund_window`, và `top1_preview` chứa "14 ngày làm việc" (stale policy).

**Fix:** CR1 (allowlist) đã block doc_id không hợp lệ. CR5 quarantine stale HR policy (effective_date < 2026-01-01). Tuy nhiên, vấn đề chính là **không có check cho policy content sai**. Tôi đã thêm:

```python
# expectations.py:57–71 — E3: refund_no_stale_14d_window
bad_refund = [
    r for r in cleaned_rows
    if r.get("doc_id") == "policy_refund_v4"
    and "14 ngày làm việc" in (r.get("chunk_text") or "")
]
```

E3 **halt** nếu có violations, ngăn embed "bad" vector. Kết quả: `eval_clean.csv` trả về **top1:"7 ngày"**, `hits_forbidden=NO`.

---

## 4. Bằng chứng trước / sau (80–120 từ)

**Before (inject bad, run_id=`inject-bad`):**
```
q_refund_window,top1_preview="Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc",hits_forbidden=YES
```

**After (clean run, run_id=`sprint2`):**
```
q_refund_window,top1_preview="Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng.",contains_expected=yes,hits_forbidden=NO
```

**File:** `artifacts/eval/eval_bad.csv` (inject) vs `artifacts/eval/eval_clean.csv` (clean)

**run_id:** `inject-bad` (skipped E3 validation) vs `sprint2` (full validation pass)

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ **thêm CR10: cross-doc consistency check** — kiểm tra xem cùng một policy version (doc_id giống nhau) có conflicting content hay không (ví dụ: `hr_leave_policy` không được có cả "10 ngày" và "12 ngày" trong các chunk khác nhau sau khi clean). Hiện tại CR5 chỉ quarantine theo effective_date, không detect content conflict giữa các chunk cùng doc_id.
