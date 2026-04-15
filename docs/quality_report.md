# Quality Report — Lab Day 10

**run_id:** clean-run  
**Ngày:** 2026-04-15

---

## 1. Tóm tắt số liệu

| Chỉ số | inject-bad (bẩn) | clean-run (sạch) | Ghi chú |
|--------|-----------------|-----------------|---------|
| `raw_records` | 10 | 10 | Cùng nguồn `policy_export_dirty.csv` |
| `cleaned_records` | 6 | 6 | Sau cleaning rules |
| `quarantine_records` | 4 | 4 | Doc_id lạ, ngày thiếu, duplicate |
| `no_refund_fix` | `true` | `false` | Flag inject: giữ nguyên "14 ngày" sai |
| `skipped_validate` | `true` | `false` | Flag inject: bỏ qua expectation halt |
| Expectation halt? | Bỏ qua (`--skip-validate`) | PASS (exit 0) | Expectation suite chạy đầy đủ ở clean-run |

---

## 2. Before / After Retrieval (Sprint 3 — bắt buộc)

> File đầy đủ: `artifacts/eval/eval_bad.csv` (inject) và `artifacts/eval/eval_clean.csv` (sau fix).

### Tổng hợp kết quả

| `question_id` | Scenario | `contains_expected` | `hits_forbidden` | `top1_doc_id` | `top1_preview` (tóm tắt) |
|--------------|----------|-------------------|-----------------|--------------|--------------------------|
| `q_refund_window` | inject-bad | `yes` | **`yes`** ⚠️ | `policy_refund_v4` | "…14 ngày làm việc…" (chunk stale) |
| `q_refund_window` | clean-run | `yes` | **`no`** ✅ | `policy_refund_v4` | "…7 ngày làm việc…" (đúng policy) |
| `q_leave_version` | inject-bad | `yes` | `no` | `hr_leave_policy` | "…12 ngày phép năm…" |
| `q_leave_version` | clean-run | `yes` | `no` | `hr_leave_policy` | "…12 ngày phép năm…" |
| `q_p1_sla` | inject-bad | `yes` | `no` | `sla_p1_2026` | "…SLA 15 phút…" |
| `q_p1_sla` | clean-run | `yes` | `no` | `sla_p1_2026` | "…SLA 15 phút…" |
| `q_lockout` | inject-bad | `yes` | `no` | `it_helpdesk_faq` | "…5 lần đăng nhập sai…" |
| `q_lockout` | clean-run | `yes` | `no` | `it_helpdesk_faq` | "…5 lần đăng nhập sai…" |

### Phân tích câu `q_refund_window` (bắt buộc — DoD)

**Trước fix (inject-bad):**
```
q_refund_window, contains_expected=yes, hits_forbidden=YES
top1_preview: "Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn (ghi chú: bản sync cũ policy-v3 — lỗi migration)."
```
→ Top-k chunks vẫn chứa chunk stale **"14 ngày"** (keyword cấm `must_not_contain`). Agent nếu đọc context này sẽ trả lời **sai** cho khách hàng.

**Sau fix (clean-run):**
```
q_refund_window, contains_expected=yes, hits_forbidden=NO
top1_preview: "Yêu cầu được gửi trong vòng 7 ngày làm việc kể từ thời điểm xác nhận đơn hàng."
```
→ Chunk stale đã bị **prune** khỏi Chroma (upsert + delete id thừa). Retrieval trả về đúng **"7 ngày"**.

**Kết luận:** `hits_forbidden` giảm từ `yes` → `no` trên `q_refund_window` sau khi chạy pipeline sạch. Đây là bằng chứng rõ ràng về tác động của corruption đến retrieval quality.

### Phân tích câu `q_leave_version` (Merit)

| Scenario | `contains_expected` | `hits_forbidden` | `top1_doc_expected` |
|----------|-------------------|-----------------|-------------------|
| inject-bad | `yes` | `no` | `yes` |
| clean-run | `yes` | `no` | `yes` |

→ `q_leave_version` ổn định ở cả 2 scenario vì HR leave policy không bị inject (flag `--no-refund-fix` chỉ ảnh hưởng chunk refund). `top1_doc_expected=yes` cả 2 lần — retrieval luôn trả đúng `hr_leave_policy`. Không có chunk stale HR trong bộ inject này.

---

## 3. Corruption Inject — Mô tả kỹ thuật (Sprint 3)

**Cách inject:**
```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
```

| Flag | Tác động |
|------|----------|
| `--no-refund-fix` | Bỏ qua cleaning rule fix "14 ngày → 7 ngày"; chunk stale **được embed vào Chroma** |
| `--skip-validate` | Bỏ qua expectation suite, không halt dù có vi phạm |

**Phát hiện corruption:**
- `hits_forbidden=yes` trên `q_refund_window` trong `eval_bad.csv`
- Manifest ghi `"no_refund_fix": true` → truy vết được run nào inject
- Expectation suite (khi chạy đầy đủ) sẽ **halt** và báo lỗi cửa sổ hoàn tiền

---

## 4. Freshness & Monitoring

```bash
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_clean-run.json
```

- `latest_exported_at`: `2026-04-15T08:00:00` (UTC)
- SLA được cấu hình: ≤ 24 giờ → **PASS** (chạy cùng ngày 2026-04-15)
- Nếu data không được refresh trong >24h → **WARN**; >48h → **FAIL**

---

## 5. Hạn chế & việc chưa làm

- `q_leave_version` không thể hiện rõ before/after vì HR chunks không bị inject trong scenario này — cần thêm inject riêng cho HR nếu muốn Merit đầy đủ
- Chưa test inject BOM / encoding lỗi cho cleaning rule mới
- `grading_run.jsonl` chưa chạy (chờ sau 17:00 theo đề)
