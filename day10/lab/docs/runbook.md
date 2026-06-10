# Sổ tay vận hành (Runbook) — Sự cố đường ống dữ liệu (Lab Day 10)

**Nhóm:** Cá nhân (Nguyen Ho Dieu Linh — MSV: 2A202600567)  
**Cập nhật:** 2026-06-10  

---

## Triệu chứng (Symptom)

- Hệ thống RAG trả về kết quả sai lệch: ví dụ, trả lời khách hàng có "14 ngày làm việc" để hoàn tiền thay vì "7 ngày làm việc", hoặc trả lời chính sách nghỉ phép của nhân viên dưới 3 năm kinh nghiệm là "10 ngày phép năm" thay vì "12 ngày".
- Pipeline chạy bị dừng đột ngột (crashing/halt) với mã thoát khác 0 (exit code != 0).

---

## Phát hiện (Detection)

- Cảnh báo SLA freshness kích hoạt khi `freshness_check=FAIL` hoặc `freshness_sla_exceeded` xuất hiện trong file manifest của lượt chạy.
- Mong đợi chất lượng dữ liệu thất bại (`expectation[...] FAIL (halt)`) được log lại, hoặc quá trình tự động kiểm tra định lượng retrieval (`hits_forbidden=yes` hoặc `contains_expected=no`) trong file đánh giá định kỳ báo lỗi.

---

## Chẩn đoán (Diagnosis)

| Bước | Việc làm | Kết quả mong đợi |
|------|----------|------------------|
| 1 | Kiểm tra file manifest `artifacts/manifests/manifest_<run_id>.json` | Xác định timestamp chạy gần nhất, số bản ghi raw, cleaned, quarantine, và lý do freshness check thất bại. |
| 2 | Mở file cách ly dữ liệu `artifacts/quarantine/quarantine_<run_id>.csv` | Tìm kiếm các cột lỗi và cột `reason` để biết tại sao bản ghi bị cách ly (e.g. `stale_hr_policy_version`, `invalid_effective_date_format`). |
| 3 | Chạy thử truy vấn tự động `python eval_retrieval.py` | Kiểm tra file `artifacts/eval/before_after_eval.csv` xem cột `hits_forbidden` và `contains_expected` có dòng nào báo `yes` / `no` bất thường không. |

---

## Khắc phục (Mitigation)

1. **Rerun Pipeline (Chạy lại):** Sửa lỗi dữ liệu thô trong hệ thống nguồn hoặc cập nhật quy tắc dọn dẹp trong `transform/cleaning_rules.py`. Sau đó chạy lại:
   ```bash
   PYTHONPATH=. .venv/bin/python etl_pipeline.py run
   ```
2. **Rollback Embed (Quay lui):** Nếu dữ liệu xấu đã bị nạp vào database (do chạy bằng `--skip-validate`), khôi phục trạng thái database sạch bằng cách chạy lại pipeline chuẩn không kèm tham số bỏ qua validate. Do cơ chế tự dọn dẹp (pruning) của pipeline, Chroma DB sẽ tự động loại bỏ các vector ID xấu ngay lập tức.
3. **Cảnh báo tạm thời:** Bật banner thông báo hệ thống bảo trì dữ liệu nếu cần trì hoãn sửa chữa lớn.

---

## Phòng ngừa (Prevention)

- Bổ sung thêm các quy tắc kiểm tra nghiêm ngặt (Expectations) đối với các trường quan trọng trước khi ghi đè cơ sở dữ liệu.
- Kênh Slack `#incident-alerts` phải được đăng ký nhận webhook cảnh báo tự động khi pipeline kích hoạt Halt.
- Đảm bảo versioning của chính sách luôn đi kèm với cập nhật ngày hiệu lực (`effective_date`) rõ ràng và chuẩn ISO.
