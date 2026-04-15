# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| Raw CSV export từ hệ nguồn (DB/API) | File upload / scheduled batch | Duplicate records, missing effective_date, invalid date formats, unknown doc_id, stale HR policies | raw_records (volume), quarantine_records (quality), freshness SLA 24h |
| Canonical docs trong data/docs/ | Manual sync / version control | Version conflicts, outdated content | expectation suite pass/fail, manual review |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| chunk_id | string | Có | ID ổn định sau clean (hash của doc_id + text + seq) |
| doc_id | string | Có | Khóa logic tài liệu nguồn (vd policy_refund_v4) |
| chunk_text | string | Có | Nội dung chunk, min length 8, đã fix stale refund |
| effective_date | date | Có | Ngày hiệu lực chuẩn hoá YYYY-MM-DD |
| exported_at | datetime | Có | Timestamp export từ nguồn |

---

## 3. Quy tắc quarantine vs drop

> Record bị flag đi đâu? Ai approve merge lại?

Records bị quarantine khi:
- doc_id không trong allowlist (unknown_doc_id)
- effective_date trống hoặc format sai
- chunk_text trống
- duplicate chunk_text
- hr_leave_policy với effective_date < 2026-01-01 (stale version)

Không drop record nào, tất cả flag đều đi quarantine để review manual.
Merge lại: Data owner (team CS/IT Helpdesk) approve sau khi fix root cause (cập nhật source, fix format, etc.).

---

## 4. Phiên bản & canonical

> Source of truth cho policy refund: file nào / version nào?

Source of truth: data/docs/policy_refund_v4.txt (version 4, cửa sổ 7 ngày).
Các version cũ (v3 với 14 ngày) bị quarantine.
Canonical cho HR leave: version 2026 (12 ngày), version 2025 bị quarantine.
