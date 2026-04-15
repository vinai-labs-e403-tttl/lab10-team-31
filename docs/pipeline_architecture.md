# Kiến trúc pipeline — Lab Day 10

**Nhóm:** E403-team-31
**Cập nhật:** 15/04/2026

---

## 1. Sơ đồ luồng (bắt buộc có 1 diagram: Mermaid / ASCII)

```mermaid
<<<<<<< HEAD
graph LR
    A["📄 Raw CSV<br/>(policy_export_dirty.csv)<br/>10 records"] -->|load| B["🧹 Clean<br/>(CR1-CR9)<br/>quarantine: 4 records"]
    B -->|write| C["📋 Cleaned CSV<br/>(artifacts/cleaned/)"] 
    B -->|write| D["⛔ Quarantine CSV<br/>(artifacts/quarantine/)"]
    B -->|validate| E["✓ Expectations<br/>(E1-E5)<br/>halt if fail"]
    E -->|upsert| F["🧠 Chroma DB<br/>(idempotent)<br/>chunk_id key"]
    F -->|serve| G["🤖 Agent Day 09<br/>(retrieval)"]
    E -->|log| H["📊 Manifest<br/>(run_id.json)<br/>freshness metadata"]
    H -->|check| I["⏰ Freshness Monitor<br/>(SLA 24h)<br/>PASS/WARN/FAIL"]
```

**Luồng:**
- **Raw ingest:** Load CSV từ `data/raw/policy_export_dirty.csv` (10 records mẫu)
- **Clean:** Áp dụng 9 rules (allowlist doc_id, normalize date, dedupe, check refund 7-day v.v.)
  - Quarantine: 4 records (stale HR, invalid date format, duplicate, suspicious pattern)
  - Pass: 6 records → cleaned CSV
- **Validate:** Run 5 expectations (E1-E5), **halt** nếu critical fail (refund stale 14d, empty doc_id, etc.)
- **Embed:** Upsert vào Chroma collection `day10_kb` dùng `chunk_id` stable (idempotent)
- **Monitor:** Lưu manifest với `run_id`, `raw/cleaned/quarantine counts`, `latest_exported_at`
- **Freshness check:** Compare manifest timestamp vs SLA 24h → PASS/WARN/FAIL
=======
flowchart LR
    subgraph Ingest
        A[raw CSV] --> B[load_raw_csv]
        B --> C[log raw_records]
    end

    subgraph Transform
        D[clean_rows] --> E[cleaned CSV]
        D --> F[quarantine CSV]
        E --> G[write_cleaned_csv]
        F --> H[write_quarantine_csv]
    end

    subgraph Quality
        I[run_expectations] --> J{halt?}
        J -->|PASS| K[tiếp tục embed]
        J -->|FAIL| L[quay lại fix<br>hoặc --skip-validate]
    end

    subgraph Embed
        M[upsert Chroma] --> N[prune stale ids]
        M --> O[collection: day10_kb]
    end

    subgraph Monitor
        P[check_manifest_freshness] --> Q[freshness SLA 24h]
    end

    A --> D
    E --> I
    K --> M
    G --> P
    H --> P
    N --> P

    style Ingest fill:#e1f5fe
    style Transform fill:#fff3e0
    style Quality fill:#e8f5e9
    style Embed fill:#f3e5f5
    style Monitor fill:#fce4ec
```

**run_id** được ghi vào manifest và metadata của mỗi vector (dùng để trace lineage và prune theo batch).

**Quarantine** nhận các bản ghi bị loại tại mỗi cleaning rule với lý do cụ thể:
- `unknown_doc_id`, `suspicious_doc_id_prefix`
- `invalid_exported_at`, `empty_effective_date`, `invalid_effective_date_format`
- `stale_hr_policy_effective_date`, `missing_chunk_text`, `chunk_text_too_short`
- `duplicate_chunk_text`
>>>>>>> 492b600ca889c57f27018476aa79e979b875a211

---

## 2. Ranh giới trách nhiệm

| Thành phần | Input | Output | Owner nhóm |
|------------|-------|--------|--------------|
<<<<<<< HEAD
| **Ingest** | Raw CSV (`data/raw/policy_export_dirty.csv`) | Row list dict, `raw_records` count | Ingestion Owner |
| **Transform** | Row list, schema contract | Cleaned rows + quarantine flags | Cleaning/Quality Owner |
| **Quality** | Cleaned rows | Expectation results (pass/halt), detailed reasons | Cleaning/Quality Owner |
| **Embed** | Cleaned CSV, expectations passed | Chroma upsert, `chunk_id` stable, prune old | Embed Owner |
| **Monitor** | Manifest JSON, SLA config | Freshness check (PASS/WARN/FAIL), alert message | Monitoring/Docs Owner |
=======
| **Ingest** | `data/raw/policy_export_dirty.csv` | Dict rows, log `raw_records`, tạo `run_id` | Ingestion Owner |
| **Transform** (cleaning_rules.py) | Dict rows | `(cleaned, quarantine)` tuples → cleaned CSV + quarantine CSV | Cleaning Owner |
| **Quality** (expectations.py) | cleaned rows | `ExpectationResult[]` + halt flag, log từng expectation | Quality Owner |
| **Embed** (chroma utils) | cleaned CSV | upsert vector vào Chroma collection `day10_kb`, prune vector cũ | Embed Owner |
| **Monitor** (freshness_check.py) | manifest JSON | `PASS/WARN/FAIL` + `age_hours` vs SLA | Monitoring/Docs Owner |

**freshness** được đo tại điểm `publish` (sau khi embed thành công), dựa trên `latest_exported_at` trong manifest.
>>>>>>> 492b600ca889c57f27018476aa79e979b875a211

---

## 3. Idempotency & rerun

<<<<<<< HEAD
**Strategy:** Upsert số $\text{upsert}$ vào Chroma dùng `chunk_id` làm khóa stable.

**Implementation:**
- `chunk_id = sha256(doc_id | chunk_text | seq)[:16]` → deterministic
- Mỗi lần embed chạy, Chroma **cập nhật (không insert duplicate)** nếu `chunk_id` đã tồn tại
- Sau publish, prune các `chunk_id` bị xóa từ nguồn (nếu row bị quarantine lần này nhưng embed lần trước)

**Test:** Rerun 2 lần với cùng CSV → kết quả identical, không append duplicate vector.
=======
**Upsert theo `chunk_id`**: mỗi run tạo `chunk_id = {doc_id}_{seq}_{hash16}` — stable không đổi khi text không đổi. Chroma upsert sẽ cập nhật vector mà không tạo duplicate nếu cùng `chunk_id`.

**Prune sau publish**: sau upsert, pipeline lấy toàn bộ `ids` đang có trong collection, so sánh với `ids` của run hiện tại, xoá các id không còn trong cleaned (đảm bảo vector stale không còn trong top-k).

```
prev_ids = col.get(ids=[])
drop = sorted(prev_ids - set(current_ids))
col.delete(ids=drop)
```

→ **Rerun 2 lần không duplicate vector**, chỉ refresh nếu text thay đổi hoặc xoá nếu bị loại khỏi cleaned.
>>>>>>> 492b600ca889c57f27018476aa79e979b875a211

---

## 4. Liên hệ Day 09

<<<<<<< HEAD
**Share chung:** Cùng `data/docs/` (5 tài liệu: policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, access_control_sop).

**Flow:**
1. Day 09 multi-agent chạy retrieval trên Chroma collection `day10_kb`
2. Day 10 pipeline update collection:
   - Thêm/cập nhật chunk từ cleaned CSV
   - Prune chunk từ stale version (vd policy_refund_v3 hoặc hr_leave_policy 2025)
   - Trigger rerun Day 09 test để kiểm tra retrieval chất lượng

**Validation:** Test 4 golden queries (`test_questions.json`) — nếu bất kỳ `contains_expected=no` hoặc `hits_forbidden=yes` → investigate pipeline quality.
=======
Pipeline này cung cấp **corpus đã cleaned và embedded** vào Chroma collection `day10_kb`, phục vụ retrieval cho Day 09 agent.

- **Cùng nguồn**: canonical source documents ở `data/docs/` (policy_refund_v4.txt, sla_p1_2026.txt, it_helpdesk_faq.txt, hr_leave_policy.txt) — cùng content nhưng exported qua CSV dirty để simulate ingest thực tế.
- **Export riêng**: cleaned data được embed vào Chroma (không dùng lại vector từ Day 09) để đảm bảo freshness và clean hoàn toàn trước khi phục vụ agent.
- **run_id lineage**: manifest ghi `run_id` và `latest_exported_at` để Day 09 agent/biến có thể verify corpus version.
>>>>>>> 492b600ca889c57f27018476aa79e979b875a211

---

## 5. Rủi ro đã biết

<<<<<<< HEAD
- **Stale data version mixin:** Nếu source export quên xóa policy_refund_v3 (14-day), embedding lỗi → agent trả "14 ngày" thay vì "7 ngày" ✓ *mitigated: E3 halt + quarantine v3*
- **Duplicate chunk_id từ format inconsistency:** Nếu CSV lần này có Unicode khác, hash khác → tạo chunk_id mới mà content giống → duplicate vector ✓ *mitigated: normalization CR3*
- **Freshness SLA 24h quá lông:** Nếu batch chạy late (hôm sau), freshness WARN/FAIL mặc dù data ổn ✓ *mitigated: runbook có escalation path*
- **Export future timestamp (clock skew):** Nếu source server sai giờ, `exported_at` > now ✓ *mitigated: CR8 quarantine invalid_exported_at*
=======
- **Stale refund policy (14→7 ngày)**: baseline đã fix, expectation E3 fail nếu còn sót
- **HR leave policy version conflict (10 vs 12 ngày)**: HR policy trước 2026-01-01 bị quarantine; expectation E6 fail nếu còn 10 ngày phép năm
- **Chunk text ngắn**: CR9 quarantine nếu < 20 chars; E4/E5 kiểm tra post-clean
- **Future timestamp poisoning**: exported_at > 2030 hoặc < 2020 bị quarantine; E8 warn
- **Suspicious doc_id prefix** (legacy/test/beta/deprecated_): CR7 quarantine
- **Freshness SLA breach**: nếu `latest_exported_at` cũ hơn 24h, freshness_check trả FAIL và alert email
>>>>>>> 492b600ca889c57f27018476aa79e979b875a211
