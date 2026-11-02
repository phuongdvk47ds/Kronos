# 02 - ARCHITECTURE C4 (KRONOS)

| Thuộc tính | Giá trị |
|---|---|
| Phạm vi | Tương tác giữa các package; sơ đồ C4 mức System Context, Container, Component |
| Phương pháp | Phân tích câu lệnh `import` và `sys.path` của từng package; đọc `webui/app.py`, `model/kronos.py` |
| Nhãn bằng chứng | `[CODE]` có trong mã; `[OPEN]`, `[PROPOSED]`, `[UNKNOWN]` theo nghĩa thông thường |
| Tài liệu liên quan | `01_System_Overview.md` |

## 1. Ánh xạ tầng Router - Service - Provider

Kronos không áp dụng kiến trúc phân tầng chính thức. Bảng dưới ánh xạ các thành phần thực tế vào mô hình ba tầng để phục vụ phân tích.

| Tầng | Vai trò | Thành phần trong mã |
|---|---|---|
| Router / Controller | Nhận yêu cầu từ Tác nhân, kiểm tra đầu vào, trả phản hồi | Route Flask trong `webui/app.py` (`/`, `/api/data-files`, `/api/load-data`, `/api/load-model`, `/api/predict`, `/api/available-models`, `/api/model-status`); hàm `main()` của các tập lệnh CLI (`finetune/`, `finetune_csv/`, `examples/`); hàm kiểm thử trong `tests/` |
| Service | Điều phối nghiệp vụ | `KronosPredictor` (`predict`, `predict_batch`, `generate`), `auto_regressive_inference`, `sample_from_logits`; `SequentialTrainer` (`finetune_csv/train_sequential.py`); `train_model` trong các tập lệnh huấn luyện; `QlibBacktest`, `KronosBacktester`, `HistoricalBacktester` |
| Domain / Model | Thuật toán mô hình | `Kronos`, `KronosTokenizer` (`model/kronos.py`); `BSQuantizer`, `TransformerBlock`, `DualHead`, `DependencyAwareLayer` (`model/module.py`) |
| Provider | Truy cập tài nguyên ngoài tiến trình | Hugging Face Hub qua `from_pretrained` (`PyTorchModelHubMixin`); `load_data_file`, `save_prediction_results`, `create_prediction_chart` (`webui/app.py`); `QlibDataPreprocessor`, `QlibDataset`; `CustomKlineDataset`, `ConfigLoader`; Qlib, AKShare, Comet ML |

Giới hạn: Router của Web UI chứa cả logic Provider (đọc tệp, ghi JSON, dựng biểu đồ) và một phần Service (cắt cửa sổ dữ liệu, tính chỉ số liên tục) trong cùng tệp `webui/app.py` `[CODE]`. Không có lớp Repository hay Dependency Injection.

## 2. Phụ thuộc giữa các package

| Package nguồn | Phụ thuộc vào | Cơ chế nhập | Bằng chứng |
|---|---|---|---|
| `webui` | `model` | `sys.path.append(<repo root>)` rồi `from model import ...`; lỗi import đặt cờ `MODEL_AVAILABLE=False` | `webui/app.py:15-22`, `webui/run.py:55-56` |
| `finetune` | `model.kronos`; `finetune.config`, `dataset`, `utils` | `sys.path.append("../")`; `from config import Config`, `from dataset import QlibDataset` | `train_tokenizer.py`, `train_predictor.py`, `qlib_test.py` |
| `finetune_csv` | `model`; nội bộ `config_loader`, `finetune_tokenizer`, `finetune_base_model` | `sys.path.append('../')`; nhập tên mô-đun trần | `train_sequential.py:10-15` |
| `examples` | `model` (7 tệp dự báo); không phụ thuộc `model` ở `run_backtest_kronos.py`, `historical_backtest.py` | `sys.path.append("../")` | `examples/prediction_example.py:4-5` |
| `tests` | `model` | `from model import ...` | `tests/test_kronos_regression.py:10` |
| `model.kronos` | `model.module` | `sys.path.append("../")`; `from model.module import *` | `model/kronos.py:9-10` |

Chiều phụ thuộc: mọi package vỏ trỏ vào `model`; `model` không phụ thuộc package nào khác. Không có phụ thuộc vòng `[CODE]`. Hai luồng `finetune` và `finetune_csv` không tham chiếu lẫn nhau.

`[OPEN]` Cơ chế `sys.path.append("../")` và `from config import Config` phụ thuộc thư mục làm việc. README hướng dẫn chạy `torchrun ... finetune/train_tokenizer.py` từ thư mục gốc; khi đó `"../"` trỏ tới thư mục cha của kho mã, và việc phân giải `model` phụ thuộc môi trường (`PYTHONPATH` hoặc chạy trong `finetune/`). `[PROPOSED]` đóng gói thành gói Python hoặc chuẩn hóa `PYTHONPATH`.

## 3. C4 Mức 1 - System Context

```mermaid
C4Context
    title System Context - Kronos
    Person(quant, "Nha nghien cuu dinh luong", "Du bao chuoi nen qua thu vien Python")
    Person(analyst, "Nha phan tich thi truong", "Du bao qua trinh duyet")
    Person(mle, "Ky su hoc may", "Tinh chinh tokenizer va mo hinh")
    Person(evaluator, "Nguoi danh gia chien luoc", "Backtest tin hieu du bao")
    Person(maintainer, "Nguoi bao tri", "Chay hoi quy pytest")
    System(kronos, "Kronos", "Mo hinh nen tang cho chuoi nen tai chinh: thu vien, Web UI, tap lenh huan luyen va backtest")
    System_Ext(hub, "Hugging Face Hub", "Luu tru trong so NeoQuasar Kronos va Tokenizer")
    System_Ext(qlib, "Qlib cn_data", "Du lieu gia co phieu Trung Quoc")
    System_Ext(akshare, "AKShare", "API du lieu thi truong dung trong vi du")
    System_Ext(comet, "Comet ML", "Theo doi thi nghiem huan luyen")
    Rel(quant, kronos, "Goi KronosPredictor", "Python")
    Rel(analyst, kronos, "Nap du lieu va du bao", "HTTP cong 7070")
    Rel(mle, kronos, "Chay torchrun va train_sequential", "CLI")
    Rel(evaluator, kronos, "Chay backtest", "CLI")
    Rel(maintainer, kronos, "Chay pytest", "CLI")
    Rel(kronos, hub, "Tai trong so mo hinh", "HTTPS")
    Rel(kronos, qlib, "Doc du lieu huan luyen", "Tep cuc bo")
    Rel(kronos, akshare, "Thu thap du lieu vi du", "HTTPS")
    Rel(kronos, comet, "Ghi chi so huan luyen", "HTTPS")
```

| Tương tác | Giao thức | Ghi chú |
|---|---|---|
| Tác nhân tới Kronos | Lời gọi Python, HTTP cổng 7070, CLI | Không có xác thực ở bất kỳ kênh nào |
| Kronos tới Hub | HTTPS | Cần ở lần nạp đầu tiên hoặc khi bộ nhớ đệm trống |
| Kronos tới Qlib | Tệp cục bộ | `qlib_data_path` |
| Kronos tới Comet | HTTPS | `use_comet=True` mặc định, khóa giữ chỗ |

## 4. C4 Mức 2 - Container Diagram

Container là đơn vị thực thi hoặc lưu trữ độc lập. Kronos không có container triển khai riêng; mỗi tiến trình sau do Tác nhân khởi chạy cục bộ.

```mermaid
C4Container
    title Container Diagram - Kronos
    Person(analyst, "Nha phan tich thi truong", "Trinh duyet")
    Person(quant, "Nghien cuu dinh luong va Ky su hoc may", "Notebook va terminal")
    System_Boundary(kronos, "Kronos repository") {
        Container(spa, "Web UI frontend", "HTML, axios, Plotly", "index.html: Control Panel, Time Window, bieu do")
        Container(flask, "Web API", "Flask, flask-cors", "webui/app.py: 7 route, trang thai predictor toan cuc, cong 7070")
        Container(core, "Core library", "PyTorch, einops", "model/: KronosTokenizer, Kronos, KronosPredictor, auto_regressive_inference")
        Container(qpipe, "Qlib pipeline", "PyTorch DDP, Qlib", "finetune/: tien xu ly, train_tokenizer, train_predictor, qlib_test")
        Container(cpipe, "CSV pipeline", "PyTorch DDP, YAML", "finetune_csv/: SequentialTrainer, CustomKlineDataset")
        Container(bt, "Backtest va vi du", "Python, matplotlib", "examples/: du bao, thu thap du lieu, backtest")
        Container(tests, "Regression tests", "pytest", "tests/: hoi quy dau ra va MSE")
        ContainerDb(files, "Local file store", "CSV, Feather, pickle, JSON", "data, train_data.pkl, prediction_results")
        ContainerDb(ckpt, "Checkpoint store", "save_pretrained", "best_model cua tokenizer va mo hinh")
    }
    System_Ext(hub, "Hugging Face Hub", "NeoQuasar")
    System_Ext(qlibdata, "Qlib cn_data", "Du lieu thi truong")
    System_Ext(akshare, "AKShare", "API du lieu")
    System_Ext(comet, "Comet ML", "Theo doi thi nghiem")
    Rel(analyst, spa, "Su dung", "HTTPS hoac HTTP")
    Rel(spa, flask, "Goi /api/*", "JSON HTTP")
    Rel(flask, core, "Import va goi predict", "Python")
    Rel(flask, files, "Doc CSV, ghi JSON", "Tep")
    Rel(quant, core, "Goi truc tiep", "Python")
    Rel(quant, qpipe, "torchrun", "CLI")
    Rel(quant, cpipe, "python --config", "CLI")
    Rel(quant, bt, "Chay", "CLI")
    Rel(qpipe, core, "Nap Kronos va KronosTokenizer", "Python")
    Rel(cpipe, core, "Nap Kronos va KronosTokenizer", "Python")
    Rel(bt, core, "Goi KronosPredictor", "Python")
    Rel(tests, core, "Goi predict", "Python")
    Rel(core, hub, "from_pretrained", "HTTPS")
    Rel(qpipe, qlibdata, "Doc", "Tep")
    Rel(qpipe, files, "Ghi pickle", "Tep")
    Rel(qpipe, ckpt, "Ghi best_model", "Tep")
    Rel(cpipe, files, "Doc CSV", "Tep")
    Rel(cpipe, ckpt, "Ghi best_model", "Tep")
    Rel(qpipe, comet, "Ghi chi so", "HTTPS")
    Rel(bt, akshare, "Thu thap du lieu", "HTTPS")
```

| Container | Đường dẫn | Điểm vào | Trạng thái | Trách nhiệm |
|---|---|---|---|---|
| Web UI frontend | `webui/templates/index.html` | `GET /` | Không trạng thái phía máy chủ | Giao diện và gọi `/api/*` |
| Web API | `webui/app.py`, `webui/run.py` | `python webui/run.py` | Biến toàn cục `tokenizer`, `model`, `predictor` | Router, nạp dữ liệu, dựng biểu đồ, lưu JSON |
| Core library | `model/` | `from model import ...` | Không trạng thái ngoài tham số mô hình | Mã hóa, sinh tự hồi quy, dự báo |
| Qlib pipeline | `finetune/` | `torchrun ... train_*.py`, `qlib_test.py` | Tiến trình theo lần chạy | Tiền xử lý, huấn luyện, backtest |
| CSV pipeline | `finetune_csv/` | `train_sequential.py --config` | Tiến trình theo lần chạy | Huấn luyện hai pha từ CSV |
| Backtest và ví dụ | `examples/` | Tập lệnh riêng lẻ | Tiến trình theo lần chạy | Minh họa dự báo và đánh giá tín hiệu |
| Regression tests | `tests/` | `pytest` | Không | Phát hiện thay đổi hành vi |

## 5. C4 Mức 3 - Component (Web API và Core library)

```mermaid
C4Component
    title Component Diagram - Web API va Core library
    Container_Boundary(flask, "Web API webui/app.py") {
        Component(routes, "Routes", "Flask route", "7 endpoint: kiem tra dau vao, ma 400 va 500")
        Component(loader, "Data loader", "pandas", "load_data_file, load_data_files")
        Component(chart, "Chart builder", "Plotly", "create_prediction_chart")
        Component(store, "Result store", "json", "save_prediction_results")
        Component(state, "Global model state", "module variables", "tokenizer, model, predictor, AVAILABLE_MODELS")
    }
    Container_Boundary(core, "Core library model/") {
        Component(predictor, "KronosPredictor", "Service", "predict, predict_batch, generate, chon thiet bi")
        Component(infer, "auto_regressive_inference", "Service", "Vong lap pred_len buoc, cua so max_context")
        Component(sampler, "sample_from_logits", "Service", "T, top_k, top_p")
        Component(tok, "KronosTokenizer", "Domain", "encode, decode, BSQuantizer")
        Component(mdl, "Kronos", "Domain", "decode_s1, decode_s2, DualHead")
        Component(blocks, "module.py", "Domain", "TransformerBlock, RoPE, RMSNorm, embeddings")
    }
    ContainerDb(files, "Local file store", "CSV, JSON", "")
    System_Ext(hub, "Hugging Face Hub", "NeoQuasar")
    Rel(routes, loader, "Doc tep")
    Rel(routes, state, "Doc va gan predictor")
    Rel(routes, predictor, "predict")
    Rel(routes, chart, "Dung bieu do")
    Rel(routes, store, "Luu ket qua")
    Rel(loader, files, "Doc")
    Rel(store, files, "Ghi JSON")
    Rel(state, hub, "from_pretrained")
    Rel(predictor, infer, "generate")
    Rel(infer, sampler, "Lay mau s1 va s2")
    Rel(infer, tok, "encode va decode")
    Rel(infer, mdl, "decode_s1 va decode_s2")
    Rel(tok, blocks, "Su dung")
    Rel(mdl, blocks, "Su dung")
```

## 6. Trình tự gọi qua các tầng (`POST /api/predict`)

```mermaid
sequenceDiagram
    actor U as Trinh duyet
    participant R as Router predict
    participant L as Provider load_data_file
    participant P as Service KronosPredictor
    participant A as Service auto_regressive_inference
    participant T as Domain Tokenizer va Kronos
    participant C as Provider create_prediction_chart
    participant S as Provider save_prediction_results
    U->>R: POST /api/predict tham so
    R->>L: file_path
    L-->>R: DataFrame
    R->>P: predict df x_timestamp y_timestamp
    P->>A: generate
    loop pred_len buoc
        A->>T: decode_s1 va decode_s2
        T-->>A: logits
    end
    A-->>P: mang du bao
    P-->>R: pred_df
    R->>C: df pred_df
    C-->>R: bieu do JSON
    R->>S: luu ket qua
    R-->>U: 200 chart prediction_results
```

## 7. Nhận định kiến trúc

| Mã | Phát hiện | Hệ quả | Đề xuất `[PROPOSED]` |
|---|---|---|---|
| C-01 | Router `webui/app.py` chứa logic Provider và Service | Khó kiểm thử độc lập; không có kiểm thử Web UI | Tách `data_service`, `result_store`, `chart_service` |
| C-02 | Trạng thái mô hình là biến toàn cục theo tiến trình | Không an toàn với nhiều worker; nạp mô hình thay thế mô hình đang dùng | Đóng gói trong lớp quản lý mô hình có khóa |
| C-03 | `KronosPredictor` đảm nhiệm cả kiểm tra đầu vào, chuẩn hóa, sinh và khôi phục thang đo | Logic trùng lặp giữa `predict` và `predict_batch` | Trích hàm chuẩn bị đầu vào dùng chung |
| C-04 | Phụ thuộc `sys.path.append("../")` | Kết quả nhập phụ thuộc thư mục làm việc | Đóng gói với `pyproject.toml` |
| C-05 | Hai pipeline tinh chỉnh trùng lặp (tập dữ liệu, vòng huấn luyện) | Bảo trì kép; chuẩn hóa mẫu khác nhau | Hợp nhất lớp dữ liệu và vòng huấn luyện |
| C-06 | Provider Hub gắn trực tiếp vào lớp mô hình qua `PyTorchModelHubMixin` | Không thay thế được nguồn trọng số khi kiểm thử | Cho phép nạp từ đường dẫn cục bộ qua cấu hình (đã hỗ trợ bởi `from_pretrained` với đường dẫn) |
| C-07 | Không có xác thực ở Router | Xem `01_System_Overview.md` mục 7 | Đặt sau lớp xác thực |

Ghi chú định dạng: cú pháp C4 của Mermaid yêu cầu chuỗi trong dấu ngoặc kép cho tham số; ngoại lệ so với quy tắc cấm ngoặc kép trần áp dụng cho flowchart. Nhãn trong sơ đồ dùng chữ không dấu để bảo đảm trình phân tích cú pháp.
