# Báo cáo Chất lượng Dữ liệu (Quality Report) — Lab Day 10

**run_id:** `2026-06-10T08-32Z`  
**Ngày thực hiện:** 2026-06-10  

---

## 1. Tóm tắt số liệu

| Chỉ số | Trước (Corrupted Run - `inject-bad`) | Sau (Cleaned Run - `2026-06-10T08-32Z`) | Ghi chú |
|--------|-------|-----|---------|
| `raw_records` | 247 | 247 | Tổng số dòng nạp thô từ CSV |
| `cleaned_records` | 36 | 36 | Số dòng đi qua biến đổi thành công |
| `quarantine_records`| 211 | 211 | Số dòng bị cách ly do vi phạm quy tắc |
| Expectation halt? | **CÓ (FAIL ở refund_no_stale_14d_window)** | **KHÔNG (Tất cả 8 expectations đều PASS)** | Chạy chuẩn exit 0, chạy lỗi exit 2 |

---

## 2. Before / after retrieval (Đánh giá chất lượng truy vấn)

- Dẫn link file so sánh đánh giá:
  - Trước khi sửa (Corrupted): [after_inject_bad.csv](file:///Users/nguyenhodieulinh/Documents/Lecture-Day-10-2A202600567-NguyenHoDieuLinh/day10/lab/artifacts/eval/after_inject_bad.csv)
  - Sau khi sửa (Cleaned): [eval_after_fix.csv](file:///Users/nguyenhodieulinh/Documents/Lecture-Day-10-2A202600567-NguyenHoDieuLinh/day10/lab/artifacts/eval/eval_after_fix.csv)

### Trình bày câu hỏi then chốt: refund window (`q_refund_window`)

- **Trước khi sửa (Corrupted Run):**
  - **top1_doc_id:** `policy_refund_v4`
  - **top1_preview:** `Yêu cầu hoàn tiền được chấp nhận trong vòng 14 ngày làm việc kể từ xác nhận đơn.`
  - **contains_expected:** `yes`
  - **hits_forbidden:** `yes` (Lỗi nặng: Trả về thông tin stale "14 ngày làm việc")
  - **top1_doc_expected:** `yes`

- **Sau khi sửa (Cleaned Run):**
  - **top1_doc_id:** `policy_refund_v4`
  - **top1_preview:** `Yêu cầu được gửi trong vòng 7 ngày làm việc làm việc kể từ thời điểm xác nhận đơn hàng.`
  - **contains_expected:** `yes`
  - **hits_forbidden:** `no` (Hoàn hảo: Đã thay thế thành công cửa sổ 7 ngày làm việc)
  - **top1_doc_expected:** `yes`

### Đánh giá bổ sung: phép năm HR (`q_hr_annual_leave_under3`)

- **Trước khi sửa (Chính sách cũ năm 2025 bị trộn lẫn):**
  - Nếu không lọc bỏ bản 2025 (10 ngày phép), hệ thống RAG sẽ bị nhiễu do xung đột phiên bản.
- **Sau khi sửa:**
  - Nhờ rule `stale_hr_policy_version` và `stale_hr_policy_effective_date`, toàn bộ 30 bản ghi HR cũ đã bị quarantine.
  - Kết quả truy vấn trả về chính xác: `Nhân viên dưới 3 năm kinh nghiệm được 12 ngày phép năm theo chính sách 2026.`

---

## 3. Freshness & monitor (Cập nhật dữ liệu)

- **Kết quả check:** `freshness_check=FAIL`
- **Chi tiết log:** `{"latest_exported_at": "2026-04-10T00:00:00", "age_hours": 1472.551, "sla_hours": 24.0, "reason": "freshness_sla_exceeded"}`
- **Giải thích:** Do dữ liệu mẫu trong lab được xuất vào ngày 2026-04-10 (hơn 1400 giờ so với ngày hiện tại của máy chấm 2026-06-10), việc cảnh báo quá hạn 24 giờ là hoàn toàn chính xác và phản ánh đúng hoạt động của bộ đo SLA. Để kiểm tra thành công trong môi trường thật, thời điểm xuất dữ liệu phải nằm trong ngày hiện hành.

---

## 4. Corruption inject (Bơm dữ liệu xấu - Sprint 3)

- **Phương pháp thực hiện:** Chạy câu lệnh `python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate` để tắt bỏ rule thay thế ngày hoàn trả (giữ nguyên stale 14 ngày) đồng thời bỏ qua việc dừng pipeline khi xảy ra lỗi.
- **Kết quả phát hiện:** Bộ Expectation Suite phát hiện vi phạm nghiêm trọng tại `refund_no_stale_14d_window` (FAIL), chứng minh bộ kiểm tra tự động hoạt động chính xác. Khi truy vấn thử, kết quả trả về bị nhiễm độc với thông tin hoàn tiền 14 ngày làm việc.

---

## 5. Hạn chế & việc chưa làm

- Hệ thống đo Freshness hiện tại mới chỉ đọc một boundary duy nhất từ manifest xuất bản (`publish`). Để tối ưu hơn, cần đo ở cả thời điểm nạp thô (`ingest`) để phân tách rõ độ trễ của hệ thống xuất file và độ trễ của đường ống ETL.
- Bộ so sánh truy vấn tự động mới chỉ dừng lại ở kiểm tra từ khóa (keyword contains), chưa có sự tích hợp của mô hình LLM-Judge để chấm điểm ngữ nghĩa sâu sắc hơn.
