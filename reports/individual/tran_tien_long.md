# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Trần Tiến Long  
**Vai trò:** Monitoring / Docs Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài:** ~580 từ

---

## 1. Tôi phụ trách phần nào?

**File / module:**

- `monitoring/freshness_check.py` — mở rộng logic đo freshness từ 1 lên **2 boundary** (ingest + publish), thêm tính `end_to_end_lag_hours`.
- `contracts/data_contract.yaml` — điền `owner_team`, `alert_channel`, bổ sung mục `freshness.measured_at` với 2 boundary, thêm `policy_versioning`.
- `docs/pipeline_architecture.md` — vẽ diagram Mermaid, điền bảng ranh giới trách nhiệm, mô tả idempotency.
- `docs/data_contract.md` — điền source map 4 nguồn, schema cleaned, quy tắc quarantine.
- `docs/runbook.md` — viết đủ 5 mục Symptom → Detection → Diagnosis → Mitigation → Prevention.
- `docs/quality_report.md` — tổng hợp số liệu từ log của nhóm, giải thích freshness WARN.

**Kết nối với thành viên khác:**

Tôi phụ thuộc vào Embed Owner để lấy `run_id` và `manifest_*.json` sau mỗi sprint. Sau khi nhóm chạy `python etl_pipeline.py run --run-id sprint4`, tôi đọc manifest để kiểm tra freshness và điền số liệu vào `quality_report.md`.

**Bằng chứng:** function `check_manifest_freshness()` trong `monitoring/freshness_check.py` có docstring ghi rõ logic 2 boundary do tôi viết thêm so với baseline.

---

## 2. Một quyết định kỹ thuật

**Chọn đo freshness tại 2 boundary thay vì 1**

Baseline chỉ đo `latest_exported_at` (timestamp data nguồn). Khi tôi chạy thử với CSV mẫu, freshness ra `FAIL` vì `exported_at = 2026-04-10` — cũ 5 ngày. Nhưng **pipeline vừa chạy xong** — tức là pipeline không sai, chỉ là data nguồn chưa được re-export.

Nếu chỉ có 1 boundary, cả hai tình huống "pipeline không chạy lại" và "data nguồn cũ" đều báo `FAIL` giống nhau → on-call không biết phải làm gì. Tôi quyết định tách thành 2 boundary:

- **ingest** (`latest_exported_at`): data nguồn cũ → liên hệ data owner re-export.
- **publish** (`run_timestamp`): pipeline chưa chạy lại → rerun `etl_pipeline.py run`.

Kết quả trên CSV mẫu: status `WARN` (publish PASS, ingest FAIL) thay vì `FAIL` — phân biệt được nguyên nhân. Tôi ghi logic này vào `runbook.md` mục Detection và Mitigation để nhóm xử lý đúng hướng.

---

## 3. Một lỗi / anomaly đã xử lý

**Triệu chứng:** Sau Sprint 4, tôi chạy freshness check và nhận `WARN` với `reason: no_timestamp_in_manifest` thay vì lý do rõ ràng.

**Phát hiện:** Đọc manifest:
```json
{
  "run_id": "sprint4",
  "latest_exported_at": "",
  ...
}
```
Trường `latest_exported_at` rỗng vì CSV mẫu có row bị quarantine hết trước khi vào bước tính `max(exported_at)` — `cleaned` list rỗng ở run thử đó.

**Fix:** Kiểm tra lại lệnh chạy — nhóm đã dùng `--run-id sprint4` với CSV sai đường dẫn, nên `cleaned_records=0`. Sau khi sửa lại đường dẫn raw:
```bash
python etl_pipeline.py run --run-id sprint4 --raw data/raw/policy_export_dirty.csv
```
`latest_exported_at = "2026-04-10T08:00:00"` xuất hiện đúng trong manifest → freshness check ra `WARN` (ingest stale) thay vì `no_timestamp_in_manifest`.

---

## 4. Bằng chứng trước / sau

**`run_id=sprint4`** — sau khi fix đường dẫn:

```
freshness_check=WARN {"ingest_boundary": {"latest_exported_at": "2026-04-10T08:00:00",
"age_hours": 125.3, "sla_hours": 24.0}, "publish_boundary": {"run_timestamp":
"2026-04-15T09:12:00+00:00", "age_hours": 0.05, "sla_hours": 24.0},
"end_to_end_lag_hours": -125.2}
```

Pipeline chuẩn từ Embed Owner (`run_id=2026-04-15T09-00Z`):
```
q_refund_window: contains_expected=yes, hits_forbidden=no
q_leave_version: contains_expected=yes, hits_forbidden=no, top1_doc_expected=yes
```

Hai chỉ số trên cho thấy pipeline sạch — docs tôi viết trong `runbook.md` phản ánh đúng trạng thái này.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ thêm **alert tự động** khi `freshness_check=FAIL`: gọi webhook Slack (`alert_channel` trong `data_contract.yaml`) ngay cuối `etl_pipeline.py run` thay vì chỉ log ra terminal. Hiện tại on-call phải chủ động đọc log mới biết — không đủ cho môi trường production.
