# Starter · PostgreSQL + pgvector

Bộ khởi động **tùy chọn** cho lab cá nhân. Docker Compose tạo PostgreSQL + pgvector local; các file trong `src/` là pipeline đã hiện thực (đọc PDF, chunk A/B/C, embedding local, lưu pgvector, chạy benchmark). Vẫn có thể chọn Qdrant, Chroma hoặc vector store tương đương theo [GUIDE.md](../GUIDE.md).

Bài đã nộp dùng corpus **Pride and Prejudice** (Project Gutenberg #1342), dựng thành PDF 216 trang. Hướng dẫn chạy lại toàn bộ (setup, dữ liệu, benchmark, pipeline, kiểm thử) nằm trong [RUN.md](RUN.md).

## Cấu trúc

```text
starter/
├── docker-compose.yml        # PostgreSQL 16 + pgvector, chỉ mở cổng local (127.0.0.1:55433)
├── init.sql                  # bật extension khi tạo database mới
├── requirements.txt          # psycopg, pypdf và python-dotenv
├── .env.example              # cấu hình mẫu; copy thành .env
├── RUN.md                    # các lệnh chạy lại toàn bộ bài
├── scripts/
│   ├── check_setup.py        # kiểm tra kết nối và extension
│   ├── build_pdf.py          # dựng PDF và data/pages.jsonl từ nguồn Gutenberg
│   └── check_benchmark.py    # kiểm tra số câu, split, evidence, filter
├── src/
│   ├── models.py             # cấu trúc dữ liệu
│   ├── ingestion.py          # đọc PDF theo trang/chương, chuẩn hóa text
│   ├── chunking.py           # A: cửa sổ cố định 500/50; B: theo đoạn tối đa 800
│   ├── embeddings.py         # embedding model local (sentence-transformers)
│   ├── store.py              # lưu vector pgvector, metadata và filter
│   ├── evaluation.py         # Hit@1, Evidence Hit@3, tóm tắt dev/test, lưu trace
│   └── run.py                # kết nối các bước thành pipeline
├── tests/
│   └── test_pipeline.py      # test tích hợp (ghi vào strategy TEST_ONLY rồi xóa)
├── benchmark/
│   ├── questions.jsonl       # 23 câu: 15 dev / 8 test
│   └── README.md             # trường dữ liệu, quy tắc filter, tiêu chí chọn cấu hình
├── data/
│   ├── manifest.json         # nguồn, quyền, số trang, sha256
│   ├── corpus.jsonl          # corpus đã làm sạch theo chương (61 đơn vị)
│   ├── pages.jsonl           # trang đã dựng trong PDF
│   ├── pride-and-prejudice.pdf
│   └── source/pg1342.txt     # không nằm trong git; tải lại theo manifest
└── results/
    ├── summary.json          # số liệu tổng hợp
    ├── traces_{A,B,C}.jsonl  # top-3 cho từng câu hỏi
    ├── hypotheses.md
    └── audit/                # mẫu chấm coherence, metadata, grounding
```

Experiment log nằm ở thư mục gốc repo: `artifacts/experiment_log.csv`.

## 1. Cài và kiểm tra database

Cần Docker Desktop hoặc Docker Engine + Compose, Python **3.11–3.14**, và mạng để tải image/dependencies lần đầu. Các lệnh sau chạy từ thư mục `starter/`:

```bash
cd starter
python3 -m venv .venv
```

Kích hoạt `.venv`: macOS/Linux dùng `source .venv/bin/activate`; Windows PowerShell dùng `.venv\Scripts\Activate.ps1`. Sau đó:

```bash
python -m pip install -r requirements.txt
# macOS/Linux: cp .env.example .env
# Windows PowerShell: Copy-Item .env.example .env
docker compose -p day6_student_starter up -d --wait
python scripts/check_setup.py
```

Kỳ vọng: hiện tên database, phiên bản PostgreSQL và pgvector, sau đó `Setup OK`. Đây **chỉ là kiểm tra hạ tầng**, không phải kết quả benchmark. Mật khẩu trong `docker-compose.yml` là cố định và **chỉ dùng local cho lab**; nếu cổng `55433` bận, sửa đồng thời port trong `docker-compose.yml` và `DATABASE_URL` trong `.env`. `init.sql` chỉ chạy khi volume mới được tạo; nếu database sẵn có chưa bật extension, xem log Docker và kiểm tra cấu hình thay vì xóa volume chứa dữ liệu.

Dừng container nhưng giữ dữ liệu:

```bash
docker compose -p day6_student_starter stop
```

## 2. Chọn embedding model

Pipeline mặc định dùng `sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2` (384 chiều, cosine) chạy **local, không cần API key**. Model này cần thêm thư viện không có trong `requirements.txt`:

```bash
python -m pip install sentence-transformers
# chỉ cần khi dựng lại PDF bằng scripts/build_pdf.py:
python -m pip install reportlab
```

Lần chạy đầu tiên sẽ tải model (~470 MB) từ Hugging Face. Nếu dùng OpenRouter, OpenAI hoặc Gemini thì điền key tương ứng trong `.env` (**không commit `.env`**) và tự cài SDK; ghi lại version và lệnh cài trong báo cáo. Ghi tên model và số chiều; dùng cùng model cho toàn bộ chiến lược và vector query tương thích vector document. Pipeline không fallback sang vector giả khi lỗi.

## 3. Chạy và kiểm tra bài

Dữ liệu và benchmark **đã có sẵn** trong `data/` và `benchmark/`; không cần chuẩn bị corpus mới trừ khi thay đổi đề. Nếu đổi nguồn, đọc [GUIDE.md](../GUIDE.md) về **30 trang nội dung**, quyền dữ liệu và metadata, rồi cập nhật `data/manifest.json`.

1. Kiểm tra benchmark (kỳ vọng `OK`, 23 câu, 15 dev / 8 test, 5 câu có filter):

   ```bash
   python scripts/check_benchmark.py
   ```

2. Chạy pipeline để tái tạo `results/` và `artifacts/experiment_log.csv`:

   ```bash
   python -m src.run
   ```

   Pipeline dùng bảng `chunks` với khóa `(strategy, id)`. Nếu bảng được tạo bằng phiên bản cũ có khóa `id` đơn, chạy `DROP TABLE chunks;` rồi chạy lại.

3. Chạy test:

   ```bash
   python -m unittest tests.test_pipeline -v
   ```

Các cấu hình A/B/C phải dùng cùng corpus, model và benchmark; khi đổi chiến lược chunk hoặc metadata, xử lý lại dữ liệu cũ thay vì nạp chồng (nạp lại không nhân đôi). Ghi lại cấu hình, lệnh chạy, kết quả và audit để người chấm tái lập; không commit key, dữ liệu mật hoặc dữ liệu không có quyền chia sẻ.
