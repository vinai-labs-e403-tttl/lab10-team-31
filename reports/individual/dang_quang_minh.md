# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Đặng Quang Minh  
**Vai trò:** Ingestion / Cleaning / Embed / Monitoring — Monitoring / Docs Owner  
**Ngày nộp:** 2026-04-15  
**Độ dài yêu cầu:** **400–650 từ** (ngắn hơn Day 09 vì rubric slide cá nhân ~10% — vẫn phải đủ bằng chứng)

---

> Viết **"tôi"**, đính kèm **run_id**, **tên file**, **đoạn log** hoặc **dòng CSV** thật.  
> Nếu làm phần clean/expectation: nêu **một số liệu thay đổi** (vd `quarantine_records`, `hits_forbidden`, `top1_doc_expected`) khớp bảng `metric_impact` của nhóm.  
> Lưu: `reports/individual/[ten_ban].md`

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

**File / module:**

- `monitoring/freshness_check.py`
- `docs/runbook.md`
- `docs/pipeline_architecture.md`
- `docs/quality_report.md`
- `reports/group_report.md`

Tôi phụ trách phần Monitoring và tài liệu cho sprint 4. Công việc chính của tôi là chuẩn hóa cách theo dõi freshness từ manifest, viết runbook để nhóm có quy trình chẩn đoán khi agent Day 09 trả lời sai, và tổng hợp bằng chứng before/after trong tài liệu báo cáo. Tôi bám theo `run_id=sprint2` và `run_id=clean-run` để mô tả lại đúng kết quả của pipeline, thay vì viết chung chung. Tôi kết nối với bạn phụ trách Cleaning/Quality để dùng chung các số liệu `raw_records=10`, `cleaned_records=6`, `quarantine_records=4`, rồi nối sang bạn phụ trách Embed để giải thích vì sao retrieval sạch hơn sau khi rerun.

**Kết nối với thành viên khác:**

Tôi lấy số liệu cleaning và expectation từ pipeline của bạn Cleaning/Quality, sau đó nối với phần embed/idempotency của bạn Embed để chứng minh freshness và before/after retrieval trên cùng một collection `day10_kb`.

**Bằng chứng (commit / comment trong code):**

`4657518` (data contract + manifest), `917f5b9` (quality report + eval compare), `b0e3a14` (pipeline architecture), `fdcfc6a` (runbook + group report).

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Quyết định kỹ thuật quan trọng nhất tôi chọn là đo freshness từ `latest_exported_at` trong manifest thay vì nhìn vào thời điểm chạy pipeline. Cụ thể, trong `monitoring/freshness_check.py`, hàm `check_manifest_freshness()` đọc `latest_exported_at`, fallback sang `run_timestamp` nếu thiếu, rồi trả về `PASS` hoặc `FAIL` kèm `age_hours`. Với `artifacts/manifests/manifest_sprint2.json`, tôi có bằng chứng thật: `latest_exported_at="2026-04-15T08:00:00"` và `run_timestamp="2026-04-15T09:13:01.985401+00:00"`, tức độ trễ khoảng 1.22 giờ nên PASS với SLA 24 giờ. Tôi chọn cách này vì nếu chỉ nhìn `run_timestamp` thì pipeline có thể “vừa chạy xong” nhưng dữ liệu nguồn vẫn cũ. Quyết định đó giúp phần monitoring phản ánh đúng độ tươi của dữ liệu phục vụ agent, không bị đánh lừa bởi một batch rerun muộn.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Anomaly tôi dùng để theo dõi và viết runbook là trường hợp agent trả sai chính sách hoàn tiền: trả lời “14 ngày” thay vì “7 ngày”. Dấu hiệu phát hiện không nằm ở code UI mà ở evidence retrieval. Trong `artifacts/eval/eval_bad.csv`, dòng `q_refund_window` có `contains_expected=yes` nhưng `hits_forbidden=yes`, và `top1_preview` vẫn chứa câu “14 ngày làm việc”. Sau khi chạy pipeline sạch và ghi lại trong `artifacts/eval/eval_clean.csv`, cùng câu hỏi đó chuyển thành `hits_forbidden=no`, preview chỉ còn “7 ngày làm việc”. Tôi dùng case này để viết `docs/runbook.md`: kiểm tra manifest có cờ `"no_refund_fix": true` hay không, rồi xem quarantine, cleaned CSV và file eval theo đúng thứ tự chẩn đoán. Nhờ vậy nhóm có một playbook rõ ràng khi retrieval đúng bề ngoài nhưng context vẫn còn stale.

---

## 4. Bằng chứng trước / sau (80–120 từ)

Tôi dùng hai artifact thật thay cho `before_after_eval.csv` vì repo hiện lưu tách riêng:

`run_id=inject-bad`, file `artifacts/eval/eval_bad.csv`  
`q_refund_window,...,contains_expected, hits_forbidden = yes, top1_preview = "...14 ngày làm việc..."`

`run_id=clean-run`, file `artifacts/eval/eval_clean.csv`  
`q_refund_window,...,contains_expected, hits_forbidden = no, top1_preview = "...7 ngày làm việc..."`

Điểm tôi muốn nhấn mạnh là metric thay đổi không phải `contains_expected` mà là `hits_forbidden`. Đây là bằng chứng monitoring quan trọng vì agent có thể vẫn “có vẻ đúng” ở top-1, nhưng top-k còn chứa chunk stale thì hệ thống vẫn rủi ro. Ngoài ra manifest của `clean-run` cũng giữ ổn định `quarantine_records=4`, nghĩa là phần fix retrieval không làm sai lệch số lượng record sạch.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm 2 giờ, tôi sẽ mở rộng `freshness_check.py` để có đủ 3 mức `PASS/WARN/FAIL` đúng như runbook và tài liệu sprint 4, đồng thời ghi luôn cảnh báo vào manifest khi `quarantine_records / raw_records > 0.4`. Việc này sẽ giúp monitoring nhất quán hơn giữa code, docs và cách demo trước lớp.
