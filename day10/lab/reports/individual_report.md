# Báo cáo Nhóm — Lab Day 10: Data Pipeline & Data Observability

**Tên nhóm:** Nguyen Ho Dieu Linh (Cá nhân)  
**Thành viên:**
| Tên | Vai trò (Day 10) | Email |
|-----|------------------|-------|
| Nguyen Ho Dieu Linh (MSV: 2A202600567) | Ingestion / Cleaning / Embed / Monitoring (All-in-one) | hdlinh.nguyen19@gmail.com |

**Ngày nộp:** 10/06/2026  
**Repo:** Lecture-Day-10-2A202600567-NguyenHoDieuLinh  
**Độ dài:** ~850 từ

---

## 1. Pipeline tổng quan

- **Nguồn raw:** Dữ liệu thô `data/raw/policy_export_dirty.csv` chứa 247 bản ghi xuất từ 5 hệ thống nguồn (policy_refund_v4, sla_p1_2026, it_helpdesk_faq, hr_leave_policy, và access_control_sop), kèm theo các dòng rác (`invalid_doc_*`, `legacy_*`), các bản ghi trùng lặp và các dòng thiếu thông tin ngày tháng hoặc có định dạng ngày không nhất quán.
- **Lệnh chạy một dòng:**
  ```bash
  PYTHONPATH=. .venv/bin/python etl_pipeline.py run
  ```
- **Lấy run_id:** `run_id` được ghi lại trong console log và manifest dạng timestamp UTC hoặc do ta chỉ định, ví dụ `2026-06-10T08-32Z`. Log file tương ứng được lưu tại `artifacts/logs/run_2026-06-10T08-32Z.log`.

---

## 2. Cleaning & expectation

Chúng tôi đã bổ sung **5 rule mới** trong `transform/cleaning_rules.py` và **2 expectation mới** trong `quality/expectations.py` để làm sạch dữ liệu và bảo đảm chất lượng nạp.

### 2a. Bảng metric_impact

| Rule / Expectation mới (tên ngắn) | Trước (số liệu) | Sau / khi inject (số liệu) | Chứng cứ (log / CSV / commit) |
|-----------------------------------|------------------|-----------------------------|-------------------------------|
| **Rule 1: Quarantine stale HR Leave** | 0 dòng bị lọc | 30 dòng phép 2025 bị quarantine | Cột `reason` ghi `stale_hr_policy_effective_date` (22) và `stale_hr_policy_version` (8) |
| **Rule 2: Strip placeholder prefix** | 0 dòng được xử lý | Xóa bỏ cụm `"Nội dung không rõ ràng:"` và `"!!!"` | Chuẩn hóa trực tiếp vào cột `chunk_text` trong cleaned CSV |
| **Rule 4: ISO Date Validation** | 0 dòng bị lọc | Định dạng lại `/` thành `-`, lọc 6 dòng trống | Cột `reason` ghi `missing_effective_date` (6) |
| **Rule 5: SLA context enrichment** | 0 dòng enrichment | Thêm `"Ticket P1"` vào trước Escalation chunk | Cải thiện `gq_d10_06` từ `contains_expected: false` thành `true` |
| **Expectation: no_placeholder_prefixes**| Bỏ qua prefix bẩn | Phát hiện 0 prefix vi phạm (Halt) | Trả về `passed: True` cho `no_placeholder_prefixes` trong log chạy sạch |
| **Expectation: all_allowed_docs_present**| Thiếu hụt tài liệu | Đảm bảo đủ cả 5 tài liệu nguồn (Halt) | Trả về `passed: True` cho `all_allowed_docs_present` |

### Quy tắc làm sạch chính:
1. **Rule 1:** Loại bỏ chính sách cũ của HR (10 ngày phép) dựa trên ngày hiệu lực (`< 2026-01-01`) và từ khóa nhạy cảm.
2. **Rule 2:** Cắt bỏ các tiền tố placeholder như `"Nội dung không rõ ràng:"` hoặc dấu chấm than `"!!!"`.
3. **Rule 3:** Chuẩn hóa khoảng trắng thừa.
4. **Rule 4:** Định dạng lại ngày xuất `exported_at` và ngày hiệu lực sang ISO YYYY-MM-DD, cách ly các bản ghi lỗi ngày.
5. **Rule 5:** Làm giàu ngữ cảnh cho các chunk SLA P1 để tránh lỗi tìm kiếm do thiếu từ khóa "Ticket".

---

## 3. Before / after ảnh hưởng retrieval hoặc agent

- **Kịch bản inject (Sprint 3):** Chạy lệnh `--no-refund-fix --skip-validate` làm lây nhiễm chính sách hoàn tiền 14 ngày làm việc cũ vào cơ sở dữ liệu.
- **Kết quả định lượng:**
  - **Khi chưa sửa (Corrupted):** Câu hỏi `q_refund_window` trả về chunk chứa `"14 ngày làm việc"` (`hits_forbidden = yes` trong `after_inject_bad.csv`).
  - **Sau khi sửa (Cleaned):** Pipeline chuẩn hóa thành công chu kỳ thành `"7 ngày làm việc"` và trả về kết quả hoàn tiền chuẩn xác (`hits_forbidden = no` trong `eval_after_fix.csv`).

---

## 4. Freshness & monitoring

- **SLA:** 24 giờ kể từ thời điểm xuất dữ liệu (`exported_at`).
- **Ý nghĩa:**
  - **PASS:** Dữ liệu mới cập nhật dưới 24 giờ.
  - **WARN:** Độ trễ từ 24 đến 48 giờ.
  - **FAIL:** Độ trễ vượt quá 48 giờ (trong lab báo `FAIL` do dữ liệu mẫu từ tháng 4/2026, cách thời điểm hiện hành hơn 1400 giờ).

---

## 5. Liên hệ Day 09

- Dữ liệu sau khi đi qua pipeline này sẽ được nạp trực tiếp vào ChromaDB collection `day10_kb`. Khóa `chunk_id` trùng khớp với cơ chế index snapshot giúp đồng bộ hóa dữ liệu trực tiếp với Retrieval Agent ở Day 09, triệt tiêu lỗi Agent trả lời sai lệch do đọc nhầm tài liệu cũ.

---

## 6. Rủi ro còn lại & việc chưa làm

- Bộ lọc trùng lặp text (`duplicate_chunk_text`) hiện tại đang sử dụng so khớp chuỗi chữ thường. Đối với các văn bản có cấu trúc viết lại (paraphrased), bộ lọc này sẽ bỏ sót. Cần tích hợp kiểm tra độ tương đồng ngữ nghĩa (cosine similarity) để cách ly các chunk trùng lặp về mặt ngữ nghĩa sâu.
