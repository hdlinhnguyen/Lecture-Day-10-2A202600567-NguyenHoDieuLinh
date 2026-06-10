# Hợp đồng dữ liệu (Data Contract) — Lab Day 10

**Owner:** Nguyen Ho Dieu Linh (MSV: 2A202600567)  
**Kênh cảnh báo SLA:** `#incident-alerts`  
**Cập nhật:** 2026-06-10  

---

## 1. Nguồn dữ liệu (source map)

| Nguồn | Phương thức ingest | Failure mode chính | Metric / alert |
|-------|-------------------|-------------------|----------------|
| `policy_refund_v4` | CSV Export | Chứa chunk stale "14 ngày" (cũ) | `refund_no_stale_14d_window` (Halt) |
| `sla_p1_2026` | CSV Export | Thiếu context "Ticket" gây lệch top-k search | `all_allowed_docs_present` (Halt) |
| `it_helpdesk_faq` | CSV Export | Trùng lặp chunk text, khoảng trắng thừa | `no_duplicate_chunk_text` (Warn) |
| `hr_leave_policy` | CSV Export | Trộn lẫn bản HR 2025 (10 phép) và HR 2026 (12 phép) | `hr_leave_no_stale_10d_annual` (Halt) |
| `access_control_sop` | CSV Export | Tài liệu mới chưa đăng ký trong allowlist | `all_allowed_docs_present` (Halt) |

---

## 2. Schema cleaned

| Cột | Kiểu | Bắt buộc | Ghi chú |
|-----|------|----------|---------|
| `chunk_id` | string | Có | Hash SHA-256 duy nhất đại diện cho chunk |
| `doc_id` | string | Có | Khóa tài liệu nguồn (e.g. `hr_leave_policy`, `access_control_sop`) |
| `chunk_text`| string | Có | Nội dung văn bản đã làm sạch, tối thiểu 8 ký tự |
| `effective_date`| date | Có | Ngày hiệu lực chuẩn ISO YYYY-MM-DD |
| `exported_at`| datetime | Có | Thời điểm xuất dữ liệu chuẩn ISO YYYY-MM-DDTHH:MM:SS |

---

## 3. Quy tắc quarantine vs drop

- **Quarantine (Cách ly):** Các dòng bị cách ly nếu rơi vào một trong các lỗi sau:
  - `unknown_doc_id`: `doc_id` không nằm trong allowlist của Data Contract.
  - `missing_effective_date`: Trường `effective_date` bị trống.
  - `invalid_effective_date_format`: Sai định dạng ngày hiệu lực (không parse được sang ISO YYYY-MM-DD).
  - `missing_exported_at`: Trường `exported_at` bị trống.
  - `invalid_exported_at_format`: Sai định dạng ngày xuất (không parse được sang ISO YYYY-MM-DDTHH:MM:SS).
  - `stale_hr_policy_effective_date`: Bản chính sách nghỉ phép HR có ngày hiệu lực trước năm 2026 (`eff_norm < "2026-01-01"`).
  - `stale_hr_policy_version`: Bản chính sách nghỉ phép chứa thông tin phép cũ ("10 ngày phép").
  - `missing_chunk_text`: Chunk text trống hoặc chỉ toàn khoảng trắng sau khi làm sạch.
  - `duplicate_chunk_text`: Chunk text bị trùng lặp (chỉ giữ lại bản ghi xuất hiện đầu tiên).
- **Drop (Bỏ qua):** Không tự động drop dòng nào mà không ghi log cách ly để đảm bảo tính minh bạch dữ liệu (full data lineage).
- **Phê duyệt merge lại:** Data Engineer / Owner (`Nguyen Ho Dieu Linh`) cần xác nhận và sửa lỗi nguồn dữ liệu thô, sau đó chạy lại pipeline để tự động nạp lại.

---

## 4. Phiên bản & canonical

- **Source of truth cho policy refund:** `policy_refund_v4` là canonical source. Bản export có chu kỳ hoàn tiền 14 ngày sẽ bị coi là stale và tự động được chuẩn hóa thay thế bằng "7 ngày làm việc".
- **Source of truth cho hr leave policy:** Bản chính sách năm 2026 (ngày hiệu lực từ 2026-01-01 trở đi, quy định 12 ngày phép đối với người dưới 3 năm kinh nghiệm) là canonical. Mọi dòng liên quan đến bản cũ 2025 (10 ngày phép) sẽ bị quarantine triệt để qua rule `stale_hr_policy_version` và `stale_hr_policy_effective_date`.
