# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đặng Thanh Tùng  
**Vai trò:** Embed & Idempotency Owner  
**Ngày nộp:** 2026-04-15  

---

## 1. Tôi phụ trách phần nào?

**File / module chính:**

- `etl_pipeline.py` — hàm `cmd_embed_internal()`: toàn bộ logic upsert Chroma, prune chunk cũ, log embed count
- `eval_retrieval.py` — chạy retrieval evaluation, output CSV before/after
- `grading_run.py` — chạy bộ câu grading output JSONL

Tôi phụ trách bước cuối trong pipeline: nhận `cleaned_csv` từ bước clean (Trần Kiên Trường) rồi embed vào Chroma collection `day10_kb`. Đảm bảo mỗi lần `run` là một **snapshot sạch** — không để chunk cũ của run trước "lẫn" vào top-k.

**Kết nối với thành viên khác:**

- Nhận output `artifacts/cleaned/cleaned_<run_id>.csv` từ Cleaning Owner
- Manifest JSON (ghi `chroma_path`, `chroma_collection`) được Monitoring Owner (Đặng Quang Minh) dùng cho freshness check

**Bằng chứng:** log `run_id=clean-run` tại `artifacts/logs/run_clean-run.log` dòng `embed_upsert count=6 collection=day10_kb`

---

## 2. Một quyết định kỹ thuật

**Quyết định: dùng OpenAI `text-embedding-3-small` thay vì SentenceTransformers `all-MiniLM-L6-v2`**

Ban đầu baseline dùng `SentenceTransformerEmbeddingFunction` — chạy được nhưng cần tải model ~90MB lúc đầu và phụ thuộc `sentence-transformers`. Tôi quyết định chuyển sang `OpenAIEmbeddingFunction` vì:

1. **Nhất quán với Day 08–09**: toàn bộ lab đã dùng OpenAI API, tránh môi trường phân mảnh
2. **Không cần tải model local**: chạy ngay khi có `OPENAI_API_KEY` trong `.env`
3. **Vector dimension cố định**: `text-embedding-3-small` trả 1536 dims — không lo mismatch khi query vs embed

Đánh đổi duy nhất: mỗi lần `run` tốn API call. Với bộ mẫu 6 chunks thì không đáng kể. Cấu hình qua biến môi trường `EMBEDDING_MODEL` trong `.env` để dễ thay đổi sau.

---

## 3. Một lỗi hoặc anomaly đã xử lý

**Triệu chứng:** lần đầu chạy `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` → exit code 3 (không phải 0 hay 2).

**Phát hiện:** Xem log tại `artifacts/logs/run_inject-bad.log`:
```
WARN: expectation failed but --skip-validate → tiếp tục embed...
ERROR: chromadb chưa cài. pip install -r requirements.txt
```

Exit code 3 do `cmd_embed_internal()` trả về `False` khi `import chromadb` thất bại — venv chưa được activate đúng trước khi chạy. Không phải lỗi logic `--no-refund-fix`.

**Fix:** `source .venv/bin/activate` rồi chạy lại. Log lần 2 ghi `embed_upsert count=6` và `PIPELINE_OK` — xác nhận inject hoạt động đúng. Expectation `refund_no_stale_14d_window` vẫn `FAIL (halt) :: violations=1` như kỳ vọng.

---

## 4. Bằng chứng trước / sau

Hai run được lưu tại `artifacts/eval/`:

**inject-bad** (`run_id=inject-bad`, `--no-refund-fix`):
```
q_refund_window, contains_expected=yes, hits_forbidden=YES
top1_preview: "...14 ngày làm việc kể từ xác nhận đơn (bản sync cũ policy-v3)..."
```

**clean-run** (`run_id=clean-run`, pipeline chuẩn):
```
q_refund_window, contains_expected=yes, hits_forbidden=NO
top1_preview: "...7 ngày làm việc kể từ thời điểm xác nhận đơn hàng..."
```

`hits_forbidden` giảm từ `yes` → `no` — chunk stale "14 ngày" đã bị prune khỏi Chroma nhờ logic `col.delete(ids=drop)` trong `cmd_embed_internal()`. Grading run (`artifacts/eval/grading_run.jsonl`) xác nhận cả 3 câu `gq_d10_01`–`gq_d10_03` đều `contains_expected=true`, `hits_forbidden=false`.

---

## 5. Cải tiến tiếp theo

Nếu có thêm 2 giờ, tôi sẽ thêm **inject riêng cho HR leave stale**: bổ sung 1 chunk `hr_leave_policy` với text "10 ngày phép năm" và `effective_date=2026-01-15` vào raw CSV, thêm flag `--no-hr-fix` để bỏ qua rule fix text, rồi chạy eval để có bằng chứng `hits_forbidden=yes` trên `q_leave_version`. Hiện tại `q_leave_version` ổn định ở cả 2 scenario vì HR chunks không bị ảnh hưởng bởi `--no-refund-fix` — thêm case này sẽ hoàn thiện Merit của Sprint 3.
