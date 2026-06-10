# Lab Day 10 — Data Pipeline & Data Observability

**Môn:** AI in Action (AICB-P1)  
**Chủ đề:** ETL / cleaning / expectation suite / embed / freshness / before-after evidence  
**Thời gian:** 4 giờ (4 sprints × ~60 phút)  
**Tiếp nối:** Day 08 RAG · Day 09 Multi-agent — **cùng case CS + IT Helpdesk**, hôm nay làm **tầng dữ liệu** trước khi agent "đọc đúng version".

**Slide:** [`../lecture-10.html`](../lecture-10.html)

---

## 🚀 [CẢI TIẾN NỔI BẬT] Báo cáo xử lý "Rác" dữ liệu (Garbage In, Garbage Out)

Để ngăn chặn tình trạng dữ liệu lỗi thời, trùng lặp hoặc sai cấu trúc làm hỏng câu trả lời của RAG Agent (*Garbage In, Garbage Out*), hệ thống Data Ingestion & Quality Pipeline đã được nâng cấp toàn diện bởi **Nguyen Ho Dieu Linh (MSV: 2A202600567)** với các cải tiến chi tiết sau:

### 1. Phân loại và cách ly "Rác" (Quarantine Logic)
Dữ liệu thô từ 5 nguồn xuất (`policy_export_dirty.csv`) ban đầu chứa **247 bản ghi**. Pipeline đã tự động phân tích và cách ly **211 bản ghi rác** vào thư mục cách ly, chỉ giữ lại **36 bản ghi sạch** chất lượng cao nạp vào cơ sở dữ liệu ChromaDB. Chi tiết các loại rác dữ liệu được cách ly gồm:
- **Tài liệu lạ/Hệ thống cũ (`unknown_doc_id` - 109 bản ghi):** Các bản ghi rác từ các nguồn không được đăng ký trong Data Contract (ví dụ: `invalid_doc_*`, `legacy_*`) đã bị loại bỏ hoàn toàn.
- **Trùng lặp nội dung (`duplicate_chunk_text` - 57 bản ghi):** Nhận diện các chunk text trùng lặp về mặt chữ và loại bỏ, tránh việc phân mảnh tài nguyên vector store và làm nhiễu top-k.
- **Xung đột phiên bản chính sách HR cũ (`stale_hr_policy_effective_date` & `stale_hr_policy_version` - 30 bản ghi):** Cách ly toàn bộ các dòng thuộc chính sách nghỉ phép HR năm 2025 (quy định cũ 10 ngày phép năm) để Agent chỉ đọc bản chính sách mới nhất năm 2026 (12 ngày phép năm).
- **Lỗi ngày tháng (`missing_effective_date` - 6 bản ghi & lỗi định dạng ngày):** Cách ly các dòng trống thông tin thời gian hoặc có định dạng ngày tháng không thể chuẩn hóa sang định dạng chuẩn ISO.

### 2. Làm sạch và Tái tạo dữ liệu (Cleaning Rules)
Chúng tôi triển khai **5 quy tắc biến đổi** trực tiếp:
* **Rule 1 (Quarantine Stale HR Leave Policy):** Tự động phát hiện và cách ly các chính sách cũ thông qua so khớp ngày hiệu lực `< 2026-01-01` và các từ khóa liên quan đến "10 ngày phép năm".
* **Rule 2 (Strip Placeholder Prefix):** Cắt bỏ các tiền tố placeholder rác của hệ thống OCR/Parser như `"Nội dung không rõ ràng:"`, `"Nội dung không rõ ràng: "` và dấu `"!!!"`.
* **Rule 3 (Deduplication spacing normalization):** Chuẩn hóa khoảng trắng dư thừa trong văn bản.
* **Rule 4 (Export Date Normalization):** Định dạng lại ngày xuất từ định dạng `/` sang `-` và đồng bộ theo tiêu chuẩn ISO `YYYY-MM-DDTHH:MM:SS`.
* **Rule 5 (SLA Context Enrichment - Cải tiến ngữ nghĩa nâng cao):** Phát hiện chunk chứa thông tin leo thang sự cố P1 thô chỉ ghi `Escalation P1: ...` mà thiếu từ khóa `"ticket"`. Pipeline đã tự động làm giàu văn bản thành `Ticket P1 Escalation: ...`. Việc bổ sung context này giúp kéo chunk P1 lên vị trí số 1 trong kết quả tìm kiếm ngữ nghĩa của Agent thay vì bị nhiễu và trả về chunk của Ticket P2.

### 3. Chốt chặn chất lượng (Expectation Suite)
Triển khai bộ kiểm tra tự động trước khi nạp dữ liệu với hai chốt chặn nghiêm ngặt (halt):
- `no_placeholder_prefixes` (Halt): Phát hiện và chặn đứng mọi chunk còn sót tiền tố placeholder rác.
- `all_allowed_docs_present` (Halt): Bảo đảm toàn bộ 5 nguồn tài liệu canonical (bao gồm cả tài liệu mới đăng ký `access_control_sop`) đều có ít nhất 1 bản ghi sạch được nạp vào vector store.

---

## Bối cảnh

Vector store và agent Day 09 chỉ ổn nếu **pipeline ingest → clean → validate → publish** ổn. Lab này mô phỏng:

- Export "raw" từ **5 hệ thống nguồn** (CSV mẫu) có **duplicate**, **dòng thiếu ngày**, **doc_id lạ**, **ngày hiệu lực không ISO**, **xung đột version HR (10 vs 12 ngày phép)**, **chunk policy sai cửa sổ hoàn tiền (14 vs 7 ngày)**, và **nguồn dữ liệu chưa được đăng ký trong pipeline**.
- Pipeline baseline được cung cấp nhưng **chưa hoàn chỉnh** — học viên phải phân tích dữ liệu raw, phát hiện lỗ hổng trong code, sửa và mở rộng pipeline để embed **toàn bộ** dữ liệu cần thiết vào vector database.
- Nhóm phải có **log số record**, **quarantine**, **expectation halt có kiểm soát**, **run_id** trên manifest, và **bằng chứng before/after** trên retrieval test.

---

## Mục tiêu học tập

| Mục tiêu | Sprint |
|----------|--------|
| Phân tích raw data + phát hiện pipeline gaps + sửa pipeline | Sprint 1 |
| Cleaning rules + cleaned CSV + quarantine + embed | Sprint 1–2 |
| Expectation suite (≥2 mới) + chạy pipeline thành công | Sprint 2 |
| Inject corruption + so sánh eval + quality report | Sprint 3 |
| Freshness check + runbook + hoàn thiện docs & báo cáo | Sprint 4 |

---

## Nhiệm vụ chính — Pipeline cần sửa gì?

> **Pipeline baseline chưa hoàn chỉnh.** Dữ liệu raw chứa export từ **5 hệ thống nguồn**, nhưng pipeline hiện tại chỉ nhận diện và xử lý **một phần**. Học viên cần tự phân tích và sửa pipeline để embed đủ dữ liệu, đảm bảo trả lời đúng **tất cả 10 câu hỏi đánh giá** trong `data/grading_questions.json`.

### Quy trình gợi ý

**Bước 1 — Chạy pipeline lần đầu và quan sát:**

```bash
python etl_pipeline.py run
```

Pipeline sẽ **HALT** do expectation phát hiện dữ liệu chưa sạch. Đọc kỹ log để hiểu lý do.

**Bước 2 — Phân tích dữ liệu raw:**

- Có bao nhiêu `doc_id` **unique** trong `data/raw/policy_export_dirty.csv`?
- `ALLOWED_DOC_IDS` trong `transform/cleaning_rules.py` chứa những doc_id nào?
- Có nguồn dữ liệu hợp lệ nào trong CSV bị pipeline **bỏ qua** (quarantine nhầm) không?

**Bước 3 — Đối chiếu với câu hỏi đánh giá:**

- Mở `data/grading_questions.json`, kiểm tra trường `expect_top1_doc_id` — cần những nguồn nào?
- So sánh với những gì pipeline hiện tại cho phép — thiếu nguồn nào?

**Bước 4 — Sửa pipeline:**

Cần sửa `transform/cleaning_rules.py` (và có thể cả `quality/expectations.py`):
1. Cập nhật allowlist nếu phát hiện nguồn hợp lệ bị thiếu.
2. Thêm cleaning rules để loại bỏ dữ liệu stale (ví dụ: nội dung chính sách cũ vẫn xuất hiện dù ngày export mới).
3. Thêm ≥ **3 rule mới** và ≥ **2 expectation mới** (xem yêu cầu Sprint 2).
4. Đảm bảo `python etl_pipeline.py run` **exit 0** — tất cả expectations phải pass.

**Bước 5 — Kiểm tra kết quả:**

```bash
# Test retrieval tự kiểm (21 câu)
python eval_retrieval.py --out artifacts/eval/eval_after_fix.csv

# Grading chính thức (10 câu)
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

Kiểm tra: `contains_expected` phải `true` và `hits_forbidden` phải `false` cho tất cả câu hỏi.

---

## Cấu trúc thư mục

```
lab/
├── etl_pipeline.py           # Sprint 1–2: run ingest→clean→validate→embed
├── eval_retrieval.py         # Sprint 3–4: before/after retrieval (CSV)
├── grading_run.py            # Grading chính thức — 10 câu đánh giá
├── instructor_quick_check.py # GV: sanity artifact grading/manifest (tuỳ chọn)
│
├── transform/
│   └── cleaning_rules.py     # ⚠️ Baseline chưa đủ — sinh viên phải sửa + mở rộng
├── quality/
│   └── expectations.py       # Baseline expectations — sinh viên thêm ≥2 mới
├── monitoring/
│   └── freshness_check.py    # Đọc manifest + SLA đơn giản
│
├── contracts/
│   └── data_contract.yaml    # Contract dữ liệu — điền owner/SLA
│
├── data/
│   ├── docs/                 # 5 tài liệu gốc (policy, SLA, FAQ, HR, access control)
│   ├── raw/
│   │   └── policy_export_dirty.csv   # Export bẩn từ 5 hệ thống nguồn
│   ├── test_questions.json           # 21 câu tự kiểm (retrieval + keyword)
│   └── grading_questions.json        # 10 câu đánh giá chính thức
│
├── artifacts/
│   ├── logs/
│   ├── manifests/
│   ├── quarantine/
│   ├── cleaned/
│   └── eval/
│
├── docs/
│   ├── pipeline_architecture.md
│   ├── data_contract.md
│   ├── runbook.md
│   └── quality_report_template.md
│
├── reports/
│   ├── group_report.md
│   └── individual/
│       └── template.md
│
├── requirements.txt
└── .env.example
```

---

## Setup

```bash
cd lab
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
```

**Lần đầu** SentenceTransformers có thể tải model `all-MiniLM-L6-v2` (~90MB) — cần mạng.

---

## Chạy pipeline

### Luồng chuẩn (sau khi đã sửa pipeline)

```bash
# Chạy toàn bộ: ingest → clean → validate → embed
python etl_pipeline.py run

# Kiểm tra freshness
python etl_pipeline.py freshness --manifest artifacts/manifests/manifest_<run-id>.json
```

### Eval retrieval (sau khi đã embed)

```bash
python eval_retrieval.py --out artifacts/eval/after_fix_eval.csv
cat artifacts/eval/after_fix_eval.csv
```

> **Ghi chú eval:** `hits_forbidden` quét **toàn bộ top-k** chunk ghép lại (không chỉ top-1), để phát hiện "câu trả lời nhìn đúng nhưng context vẫn còn chunk stale".  
> **Index snapshot:** sau mỗi lần `run`, embed **upsert** theo `chunk_id` và **xoá id không còn trong cleaned** để tránh vector cũ làm fail grading.

### Sprint 3 — Inject corruption (embed dữ liệu "xấu", bỏ qua halt)

```bash
python etl_pipeline.py run --run-id inject-bad --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
# So sánh với file eval sau khi chạy lại pipeline chuẩn
```

### Grading chính thức (10 câu)

```bash
python grading_run.py --out artifacts/eval/grading_run.jsonl
```

**Giảng viên — kiểm tra nhanh artifact (tuỳ chọn):**

```bash
python instructor_quick_check.py --grading artifacts/eval/grading_run.jsonl
python instructor_quick_check.py --manifest artifacts/manifests/manifest_<run-id>.json
```

---

## 4 Sprints (chi tiết)

### Sprint 1 (60') — Phân tích & Ingest

- Đọc `data/raw/policy_export_dirty.csv` — liệt kê các `doc_id` unique, đếm số record mỗi loại.
- **Chạy pipeline lần đầu** → quan sát HALT → đọc log xác định nguyên nhân.
- **So sánh** `doc_id` trong CSV vs `ALLOWED_DOC_IDS` trong `cleaning_rules.py` → phát hiện nguồn bị thiếu.
- **Đối chiếu** `expect_top1_doc_id` trong `grading_questions.json` → xác nhận cần sửa gì.
- Bắt đầu sửa `cleaning_rules.py`: cập nhật allowlist, thêm rules cho dữ liệu stale.
- Điền **source map** ngắn trong `docs/data_contract.md` (ít nhất 2 nguồn / failure mode / metric).

**DoD:** Log có `raw_records`, `cleaned_records`, `quarantine_records`, `run_id`. Hiểu tại sao pipeline halt.

---

### Sprint 2 (60') — Clean + validate + embed

- Hoàn thiện sửa pipeline: pipeline phải **exit 0** với expectation không halt.
- Thêm ≥ **3 rule mới** và ≥ **2 expectation mới** (đếm trên file nhận được).
- **Chống trivial:** mỗi rule/expectation mới phải có **tác động đo được** — ghi trong `reports/group_report.md` bảng *metric_impact* (ví dụ: `quarantine_records` tăng khi inject, `expectation X fail` trước khi fix). Rule chỉ "strip space" mà không đổi số liệu → **trừ điểm**.
- Đảm bảo embed **idempotent** (upsert `chunk_id` + prune id thừa — baseline đã làm).

**DoD:** `python etl_pipeline.py run` exit 0. `python grading_run.py` → kiểm tra nhanh kết quả.

---

### Sprint 3 (60') — Inject corruption & before/after

- Cố ý làm hỏng dữ liệu (`--no-refund-fix --skip-validate`) → lưu eval "xấu".
- Chạy lại pipeline chuẩn → lưu eval "tốt".
- Lưu **2 file eval** so sánh + ảnh chụp / đoạn log chứng minh.
- Hoàn thành quality report theo `docs/quality_report_template.md`.

**DoD:** Có số liệu chứng minh retrieval **tệ hơn** trước fix và **tốt hơn** sau fix.

---

### Sprint 4 (60') — Monitoring + docs + báo cáo

- Điền `docs/pipeline_architecture.md`, `docs/data_contract.md`, `docs/runbook.md`.
- `python etl_pipeline.py freshness --manifest …` — giải thích PASS/WARN/FAIL trong runbook.
- Chạy `python grading_run.py` lần cuối → verify 10 câu đều pass.
- Hoàn thành `reports/group_report.md` + mỗi người `reports/individual/[ten].md`.

**DoD:** Grading JSONL hợp lệ. README nhóm có "một lệnh chạy cả pipeline". Peer review 3 câu hỏi ghi trong group report.

---

## Deliverables (nộp bài)

| Item | Ghi chú |
|------|---------|
| `etl_pipeline.py` + `transform/` + `quality/` + `monitoring/` | Có thể mở rộng file, không xóa entrypoint bắt buộc |
| `contracts/data_contract.yaml` | Điền owner, SLA, nguồn |
| `artifacts/logs/`, `manifests/`, `quarantine/`, `eval/` | Ít nhất 1 run "tốt" + evidence inject |
| `docs/*.md` (3 file + quality report) | Theo template |
| `reports/group_report.md` | |
| `reports/individual/*.md` | Mỗi thành viên |
| `artifacts/eval/grading_run.jsonl` | 10 câu: `gq_d10_01` … `gq_d10_10` |

---

## Dữ liệu trong raw CSV

Raw CSV (`data/raw/policy_export_dirty.csv`) chứa export từ nhiều hệ thống. Dưới đây là tham khảo (không phải đáp án — học viên tự phân tích):

| Nguồn dữ liệu | Tài liệu tham khảo | Ghi chú |
|----------------|---------------------|---------|
| `policy_refund_v4` | `data/docs/policy_refund_v4.txt` | Có chunk stale "14 ngày" cần fix |
| `sla_p1_2026` | `data/docs/sla_p1_2026.txt` | SLA và quy trình xử lý sự cố |
| `it_helpdesk_faq` | `data/docs/it_helpdesk_faq.txt` | FAQ IT nội bộ |
| `hr_leave_policy` | `data/docs/hr_leave_policy.txt` | Có xung đột version 2025 vs 2026 |
| `access_control_sop` | `data/docs/access_control_sop.txt` | Quy trình cấp quyền truy cập |
| `invalid_doc_*`, `legacy_*` | (không có tài liệu) | Export lỗi / hệ thống cũ |

> **Lưu ý:** Không phải tất cả nguồn dữ liệu đều được pipeline baseline xử lý. Học viên cần tự phát hiện và sửa.

---

## Phân vai (gợi ý — đồng bộ slide Hands-on 10)

| Vai | Trách nhiệm | Sprint chính |
|-----|-------------|----------------|
| **Ingestion Owner** | raw paths, logging, manifest, phân tích doc_id | 1 |
| **Cleaning / Quality Owner** | `cleaning_rules.py`, `expectations.py`, quarantine | 1–3 |
| **Embed Owner** | Chroma collection, idempotency, eval, grading verify | 2–3 |
| **Monitoring / Docs Owner** | freshness, runbook, 3 docs, group report | 4 |

---

## Debug order (nhắc từ slide Day 10)

```
Freshness / version → Volume & errors → Schema & contract → Lineage / run_id → mới đến model/prompt
```

---

## Tài nguyên tham khảo

- Slide: [`../lecture-10.html`](../lecture-10.html)
- Lab Day 09 (orchestration): [`../../day09/lab/README.md`](../../day09/lab/README.md)
- Great Expectations (tuỳ chọn nâng cao): https://docs.greatexpectations.io/
- ChromaDB: https://docs.trychroma.com/
