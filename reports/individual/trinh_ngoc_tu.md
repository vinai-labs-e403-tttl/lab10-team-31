# Báo Cáo Cá Nhân — Lab Day 10

**Tên:** Trịnh Ngọc Tú 
**Vai trò:** Ingestion Owner
**Email:** tutncfaa@gmail.com

---

## 1. Công việc chính (Sprint gán cho vai trò)

| Sprint | Công việc | Status |
|--------|----------|--------|
| 1 | Load raw CSV, schema ingest, log setup | ✅ |

---

## 2. Learnings & challenges

**Bài học chính:**
- Học cách thiết kế pipeline ingest rõ ràng: đọc CSV, kiểm tra schema, và log đầy đủ để dễ truy vết lỗi.
- Rút ra rằng cần kiểm thử dữ liệu sớm (schema/format) để tránh lỗi lan sang các bước sau.
- Hiểu tầm quan trọng của cấu trúc code ingest gọn, tách bước đọc–kiểm tra–ghi log để dễ bảo trì.

**Thách thức gặp:**
- Cần quyết định rõ khi expectation fail thì dừng hay cho qua, và cách ghi log để dễ theo dõi.

**Giải pháp áp dụng:**
- Thiết lập rule rõ ràng: expectation fail mức nghiêm trọng thì halt pipeline, còn mức nhẹ thì cảnh báo và vẫn tiếp tục; đồng thời chuẩn hóa log lỗi để truy vết nhanh.

---

## 3. Pull request / commits

**PR links:**
- (reference to any PRs if applicable)

**Key commits:**
- 4657518 Update data contract and clean up CSV files; add quarantine records and manifests

---

## 4. Contribution to final demo

**Phần tôi đóng góp:**
- Code ingest CSV, kiểm tra schema và thiết lập logging.

**Test scenario / evidence:**
- 4657518 

---

## Ký tên & ngày
Ngày: 15/04/2026
Ký: Tú
