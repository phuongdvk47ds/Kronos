# 04 - API DATA CONTRACTS (KRONOS)

| Thuộc tính | Giá trị |
|---|---|
| Phạm vi | Định nghĩa dùng để validate request/response: Pydantic, dataclass, TypedDict và các cơ chế thay thế |
| Phương pháp | Tìm kiếm toàn kho mã (`grep`) các thư viện và mẫu khai báo schema; đọc `webui/app.py`, `webui/templates/index.html`, `model/kronos.py`, `finetune*/` |
| Nhãn bằng chứng | `[CODE]` có trong mã; `[!]` khuyết điểm đã xác minh; `[ASSUMPTION]`, `[UNKNOWN]`, `[OPEN]`, `[PROPOSED]` |
| Tài liệu liên quan | `02_Architecture_C4.md`, `03_Core_Business_Logic.md`, `docs/WF_SRS.md` mục 8.2 |

## 1. Kết quả quét

| Cơ chế khai báo schema | Kết quả | Phương pháp kiểm chứng |
|---|---|---|
| Pydantic (`BaseModel`, `Field`, `@validator`) | Không tồn tại | Tìm `pydantic`, `BaseModel`, `Field(`, `@validator` trong `*.py`, `*.txt`, `*.md`, `*.yaml` |
| `dataclasses` (`@dataclass`) | Không tồn tại | Tìm `dataclass` |
| `typing.TypedDict`, `NamedTuple` | Không tồn tại | Tìm `TypedDict`, `NamedTuple` |
| marshmallow, jsonschema, attrs, FastAPI | Không tồn tại | Tìm theo tên |
| Phụ thuộc khai báo | `flask==2.3.3`, `flask-cors==4.0.0`; không có `pydantic` | `webui/requirements.txt`, `requirements.txt` |

Kết luận: hệ thống không có model validate request/response khai báo tường minh. Hợp đồng dữ liệu tồn tại dưới bốn dạng ngầm định:

| Dạng | Vị trí | Cách validate |
|---|---|---|
| Từ điển JSON tự do | `webui/app.py` (request: `request.get_json()`; response: `jsonify({...})`) | `data.get(key, default)`, ép kiểu `int()`, `float()` thủ công |
| DataFrame có cột quy ước | `KronosPredictor.predict`, `load_data_file`, `CustomKlineDataset` | Kiểm tra sự hiện diện cột và NaN bằng mã (`ValueError`) |
| Đối số `__init__` của mô hình | `Kronos`, `KronosTokenizer` (kế thừa `PyTorchModelHubMixin`) | Tham số vị trí; cấu hình lưu thành `config.json` trên Hub |
| Cấu hình huấn luyện | Lớp `Config` (`finetune/config.py`), YAML (`finetune_csv/`) | Không validate kiểu; `Dict[str, Any]` trong `ConfigLoader`; `ValueError` cho chuẩn so sánh |

## 2. Danh mục Data Contract cốt lõi

| ID | Tên | Hướng | Giao diện | Định nghĩa tại | Cơ chế validate |
|---|---|---|---|---|---|
| DC-01 | LoadDataRequest | Vào | `POST /api/load-data` | `webui/app.py:load_data` | `data.get`; kiểm tra rỗng |
| DC-02 | LoadDataResponse | Ra | như trên | như trên | Không (từ điển dựng tay) |
| DC-03 | DataFileItem | Ra | `GET /api/data-files` | `load_data_files` | Không |
| DC-04 | LoadModelRequest | Vào | `POST /api/load-model` | `load_model` | Kiểm tra `model_key` thuộc `AVAILABLE_MODELS` |
| DC-05 | LoadModelResponse | Ra | như trên | như trên | Không |
| DC-06 | ModelConfig | Ra, hằng số | `GET /api/available-models` | `AVAILABLE_MODELS` | Không |
| DC-07 | ModelStatusResponse | Ra | `GET /api/model-status` | `get_model_status` | Không |
| DC-08 | PredictRequest | Vào | `POST /api/predict` | `predict` | Ép kiểu thủ công; không kiểm tra miền giá trị |
| DC-09 | PredictResponse | Ra | như trên | như trên | Không |
| DC-10 | CandleRecord | Ra (phần tử mảng) | `prediction_results[]`, `actual_data[]` | `predict` | Không |
| DC-11 | ErrorResponse | Ra | Mọi route | `webui/app.py` | Không (`{error: string}`) |
| DC-12 | PredictionArchive | Ra (tệp) | `prediction_<ts>.json` | `save_prediction_results` | Không |
| DC-13 | OhlcvFrame | Vào | `predict`, `predict_batch` | `model/kronos.py` | `ValueError` theo điều kiện |
| DC-14 | PredictionFrame | Ra | như trên | như trên | Không |
| DC-15 | KronosConfig | Vào/Hub | `Kronos.from_pretrained` | `Kronos.__init__` | Không |
| DC-16 | TokenizerConfig | Vào/Hub | `KronosTokenizer.from_pretrained` | `KronosTokenizer.__init__` | Không |
| DC-17 | TrainingConfig (Qlib) | Vào | `finetune/config.py:Config` | như trên | `ValueError` cho chuẩn so sánh |
| DC-18 | FinetuneYaml (CSV) | Vào | `finetune_csv/configs/*.yaml` | `ConfigLoader`, `CustomFinetuneConfig` | `FileNotFoundError` khi thiếu tệp |
| DC-19 | ProcessedDataset | Trung gian (tệp) | `train_data.pkl`, `val_data.pkl`, `test_data.pkl` | `QlibDataPreprocessor` | Không |
| DC-20 | SignalFrame | Ra | `qlib_test.py` | `generate_predictions` | Không |
| DC-21 | TradeRecord | Ra (nội bộ) | `run_backtest_kronos.py` | `run_backtest` | Không |
| DC-22 | RegressionFixture | Vào (tệp) | `tests/data/*.csv` | `tests/test_kronos_regression.py` | Kiểm tra số hàng |

## 3. Đặc tả chi tiết các Contract Web API

### 3.1 DC-01 LoadDataRequest và DC-02 LoadDataResponse

`POST /api/load-data` (`Content-Type: application/json`)

| Trường | Kiểu | Bắt buộc | Mặc định | Ràng buộc thực thi |
|---|---|---|---|---|
| `file_path` | string | Có | - | Không rỗng (400 `File path cannot be empty`); đuôi `.csv` hoặc `.feather` (400 `Unsupported file format`); không ràng buộc thư mục `[!]` |

Response 200:

| Trường | Kiểu | Mô tả |
|---|---|---|
| `success` | bool | `true` |
| `message` | string | `Successfully loaded data, total <n> rows` |
| `data_info.rows` | int | Số hàng sau khi loại NaN |
| `data_info.columns` | string[] | Tên cột sau chuẩn hóa |
| `data_info.start_date`, `end_date` | string ISO-8601 hoặc `N/A` | Nhỏ nhất, lớn nhất của `timestamps` |
| `data_info.price_range.min`, `max` | float | Cực trị trên `open, high, low, close` |
| `data_info.prediction_columns` | string[] | `open, high, low, close` và `volume` nếu tồn tại |
| `data_info.timeframe` | string | `<n> seconds`, `<n> minutes`, `<n> hours`, `<n> days` hoặc `Unknown` |

### 3.2 DC-03 DataFileItem

`GET /api/data-files`: mảng; phần tử:

| Trường | Kiểu | Mô tả |
|---|---|---|
| `name` | string | Tên tệp `.csv` hoặc `.feather` |
| `path` | string | Đường dẫn tuyệt đối trên máy chủ |
| `size` | string | `<x.y> KB` (dưới 1 MB) hoặc `<x.y> MB` |

Mảng rỗng khi thư mục `<repo>/data` không tồn tại.

### 3.3 DC-04 đến DC-07 Mô hình

DC-04 `POST /api/load-model`:

| Trường | Kiểu | Bắt buộc | Mặc định | Ràng buộc |
|---|---|---|---|---|
| `model_key` | string | Không | `kronos-small` | Thuộc `kronos-mini`, `kronos-small`, `kronos-base`; ngoài danh sách: 400 `Unsupported model: <key>` |
| `device` | string | Không | `cpu` | Không kiểm tra; giao diện chọn `cpu`, `cuda`, `mps` |

DC-05 Response 200: `success` (bool), `message` (string), `model_info{name, params, context_length, description}`.

DC-06 `ModelConfig` (giá trị của `AVAILABLE_MODELS`):

| Trường | Kiểu | Ví dụ |
|---|---|---|
| `name` | string | `Kronos-small` |
| `model_id` | string | `NeoQuasar/Kronos-small` |
| `tokenizer_id` | string | `NeoQuasar/Kronos-Tokenizer-base` |
| `context_length` | int | 512 (small, base), 2048 (mini) |
| `params` | string | `4.1M`, `24.7M`, `102.3M` |
| `description` | string | Mô tả ngắn |

`GET /api/available-models` trả `{models: {<model_key>: ModelConfig}, model_available: bool}`.

DC-07 `GET /api/model-status` trả:

| Trường | Kiểu | Điều kiện |
|---|---|---|
| `available` | bool | `MODEL_AVAILABLE` |
| `loaded` | bool | `predictor is not None` |
| `message` | string | Ba biến thể: loaded, available nhưng chưa nạp, library không khả dụng |
| `current_model.name` | string | Tên lớp (`Kronos`), không phải `model_key` `[i]` |
| `current_model.device` | string | Thiết bị của tham số đầu tiên |

### 3.4 DC-08 PredictRequest

`POST /api/predict`:

| Trường | Kiểu | Bắt buộc | Mặc định | Ràng buộc thực thi | Ràng buộc giao diện |
|---|---|---|---|---|---|
| `file_path` | string | Có | - | Không rỗng | Chọn từ danh sách |
| `lookback` | int | Không | 400 | `int()`; số hàng của tệp lớn hơn hoặc bằng `lookback` | Chỉ đọc (400) |
| `pred_len` | int | Không | 120 | `int()` | Chỉ đọc (120) |
| `temperature` | float | Không | 1.0 | `float()`; không kiểm tra miền | 0.1 đến 2.0, bước 0.1 |
| `top_p` | float | Không | 0.9 | `float()`; không kiểm tra miền | 0.1 đến 1.0, bước 0.1 |
| `sample_count` | int | Không | 1 | `int()`; không kiểm tra miền | 1 đến 5 |
| `start_date` | string | Không | vắng | `pd.to_datetime`; số hàng từ mốc này lớn hơn hoặc bằng `lookback + pred_len` | Thanh trượt cửa sổ thời gian |

`[!]` Ràng buộc miền (`lookback > 0`, `pred_len > 0`, `0 < top_p <= 1`, `temperature > 0`, giới hạn trên của `sample_count`) chỉ tồn tại ở giao diện. Máy chủ chấp nhận giá trị bất kỳ qua API trực tiếp; `sample_count` lớn nhân kích thước batch nội bộ.

`[ASSUMPTION]` Giá trị không ép kiểu được (ví dụ `lookback = "abc"`) phát `ValueError` bên trong khối `try`, và `except Exception` trả 500 `Prediction failed: ...` thay vì 400. Thân yêu cầu không phải JSON được xử lý bởi `request.get_json()` trong cùng khối `try`, nên cũng rơi vào 500.

### 3.5 DC-09 PredictResponse và DC-10 CandleRecord

Response 200:

| Trường | Kiểu | Mô tả |
|---|---|---|
| `success` | bool | `true` |
| `prediction_type` | string | Mô tả chế độ cửa sổ (kèm khoảng thời gian khi có `start_date`) |
| `chart` | string | Chuỗi JSON của hình Plotly (`json.dumps(..., cls=PlotlyJSONEncoder)`) |
| `prediction_results` | CandleRecord[] | `pred_len` phần tử |
| `actual_data` | CandleRecord[] | Rỗng khi không đủ dữ liệu so sánh |
| `has_comparison` | bool | `len(actual_data) > 0` |
| `message` | string | Tóm tắt số điểm dự báo và số điểm so sánh |

`CandleRecord`:

| Trường | Kiểu | Quy tắc |
|---|---|---|
| `timestamp` | string ISO-8601 hoặc `T<i>` | Dự báo: dựng từ mốc thời gian cuối cộng bước cách quãng của hai hàng đầu tệp; `T<i>` khi vượt `future_timestamps` |
| `open`, `high`, `low`, `close` | float | Giá |
| `volume` | float | 0 khi cột vắng |
| `amount` | float | Dự báo: do mô hình sinh; thực tế: 0 khi tệp không có cột `amount` |

`[!]` Không nhất quán dấu thời gian: khi không có `start_date`, `prediction_results[].timestamp` được dựng từ hàng cuối của toàn tệp, còn `actual_data[].timestamp` lấy từ các hàng `lookback` đến `lookback + pred_len` trong tệp. Hai mảng mô tả cùng các bước dự báo nhưng mang dấu thời gian khác nhau. Khi có `start_date`, hai mảng nhất quán (xem mẫu `prediction_20250826_163800.json`).

`[i]` Bước cách quãng `df['timestamps'].iloc[1] - iloc[0]` được dùng cho toàn bộ dải dự báo; dữ liệu không đều (ví dụ nghỉ cuối tuần) cho dấu thời gian lệch với phiên giao dịch thực tế.

### 3.6 DC-11 ErrorResponse

| Mã HTTP | Thân | Điều kiện |
|---|---|---|
| 400 | `{"error": string}` | Đầu vào thiếu hoặc sai, mô hình chưa nạp, dữ liệu không đủ |
| 500 | `{"error": string}` | Ngoại lệ chưa lường; tiền tố `Failed to load data:`, `Kronos model prediction failed:`, `Prediction failed:`, `Model loading failed:` |

Không có `code`, `details`, `request_id` `[CODE]`.

### 3.7 DC-12 PredictionArchive (tệp JSON)

| Trường | Kiểu | Mô tả |
|---|---|---|
| `timestamp` | string ISO-8601 | Thời điểm lưu |
| `file_path` | string | Tệp dữ liệu nguồn |
| `prediction_type` | string | Như DC-09 |
| `prediction_params` | object | `lookback, pred_len, temperature, top_p, sample_count, start_date` (hoặc `latest`) |
| `input_data_summary` | object | `rows` (int), `columns` (string[]), `price_range{open,high,low,close}{min,max}`, `last_values{open,high,low,close}` |
| `prediction_results` | CandleRecord[] | |
| `actual_data` | CandleRecord[] | |
| `analysis` | object | `{}` hoặc `{continuity{last_prediction, first_actual, gaps, gap_percentages}}` |

`[!]` Khóa `model_key` và `device` không được lưu; không thể truy nguyên mô hình đã sinh kết quả từ tệp lưu `[CODE]`.

## 4. Contract thư viện và huấn luyện

### 4.1 DC-13 OhlcvFrame và DC-14 PredictionFrame

| Hướng | Cột | Kiểu | Quy tắc |
|---|---|---|---|
| Vào | `open, high, low, close` | float | Bắt buộc |
| Vào | `volume` | float | Thiếu: 0.0 |
| Vào | `amount` | float | Thiếu: `volume x mean(open, high, low, close)` hoặc 0.0 |
| Vào | `x_timestamp`, `y_timestamp` | `pd.Series` datetime | Có `.dt`; `DatetimeIndex` không được chấp nhận trực tiếp (Web UI chuyển đổi) |
| Ra | `open, high, low, close, volume, amount` | float | Chỉ mục `y_timestamp`; thang đo của đầu vào |

Ngoại lệ: `ValueError` cho kiểu, cột thiếu, NaN; `predict_batch` thêm kiểm tra danh sách, số lượng và độ dài đồng nhất.

### 4.2 DC-15 KronosConfig và DC-16 TokenizerConfig

Tham số `__init__` được `PyTorchModelHubMixin` lưu thành `config.json` của kho Hub.

| Contract | Tham số |
|---|---|
| `Kronos` | `s1_bits, s2_bits, n_layers, d_model, n_heads, ff_dim, ffn_dropout_p, attn_dropout_p, resid_dropout_p, token_dropout_p, learn_te` |
| `KronosTokenizer` | `d_in, d_model, n_heads, ff_dim, n_enc_layers, n_dec_layers, ffn_dropout_p, attn_dropout_p, resid_dropout_p, s1_bits, s2_bits, beta, gamma0, gamma, zeta, group_size` |

Không có kiểm tra kiểu hay miền giá trị; sai tham số thể hiện qua lỗi PyTorch khi dựng hoặc nạp trọng số `[UNKNOWN]`.

### 4.3 DC-17 và DC-18 Cấu hình huấn luyện

| Contract | Nhóm trường | Ghi chú |
|---|---|---|
| `Config` (Qlib) | `qlib_data_path, instrument, dataset_begin_time, dataset_end_time, lookback_window, predict_window, max_context, feature_list, time_feature_list, train/val/test/backtest_time_range, dataset_path, clip, epochs, batch_size, n_train_iter, n_val_iter, tokenizer_learning_rate, predictor_learning_rate, accumulation_steps, adam_*, seed, use_comet, comet_config, save_path, *_save_folder_name, pretrained_*_path, finetuned_*_path, backtest_*, inference_*` | Thuộc tính của một lớp thường; `backtest_benchmark` do `_set_benchmark` quyết định |
| YAML CSV | `data{data_path, lookback_window, predict_window, max_context, clip, train_ratio, val_ratio, test_ratio}`, `training{tokenizer_epochs, basemodel_epochs, batch_size, log_interval, num_workers, seed, learning rates, adam_*, accumulation_steps}`, `model_paths{...}`, `experiment{name, description, use_comet, train_tokenizer, train_basemodel, skip_existing}`, `device{use_cuda, device_id}` | `CustomFinetuneConfig` đặt giá trị mặc định bằng `dict.get`; tương thích khóa `epochs` |

### 4.4 DC-19 đến DC-22 Dữ liệu trung gian

| Contract | Cấu trúc | Ghi chú |
|---|---|---|
| ProcessedDataset | `dict[symbol -> DataFrame]` chỉ mục `datetime`, cột `open, high, low, close, vol, amt` (đã lọc theo `feature_list`) | Pickle; nạp bằng `pickle.load` (rủi ro giải tuần tự hóa) |
| SignalFrame | `DataFrame` chỉ mục `datetime`, cột `instrument`, giá trị `score`; bốn loại `last, mean, max, min` | Dùng trực tiếp cho `TopkDropoutStrategy` |
| TradeRecord | `{date, action (BUY hoặc SELL), price, shares, capital}` | Dict nội bộ |
| RegressionFixture | CSV `timestamps, open, high, low, close, volume, amount` | `tests/data/regression_input.csv`, `regression_output_{256,512}.csv` |

## 5. Khoảng trống hợp đồng và đề xuất

| Mã | Khoảng trống | Hệ quả | Đề xuất `[PROPOSED]` |
|---|---|---|---|
| G-01 | Không có model validate request | Thông điệp lỗi và mã HTTP không nhất quán (500 cho lỗi đầu vào kiểu) | Dùng Pydantic cho DC-01, DC-04, DC-08 |
| G-02 | Ràng buộc miền chỉ ở giao diện | API trực tiếp nhận `top_p`, `temperature`, `sample_count`, `lookback`, `pred_len` bất kỳ | Khai báo `ge`, `le`, `gt` cho từng trường |
| G-03 | `file_path` tự do | Đọc tệp tùy ý trên máy chủ | Chỉ nhận định danh tệp thuộc danh sách `/api/data-files` |
| G-04 | Response không có schema | Giao diện và API phụ thuộc cấu trúc ngầm | Khai báo `response_model`, phát hành OpenAPI |
| G-05 | Lỗi chỉ có chuỗi tự do | Máy khách không phân loại lỗi | `{code, message, details, request_id}` |
| G-06 | Dấu thời gian dự báo và thực tế lệch nhau khi không có `start_date` | Biểu đồ và phân tích liên tục không đối chiếu đúng bước | Dùng cùng một nguồn dấu thời gian |
| G-07 | Kết quả lưu không ghi mô hình và thiết bị | Không truy nguyên | Bổ sung `model_key`, `device` |
| G-08 | Contract cấu hình huấn luyện không kiểm tra kiểu | Lỗi phát hiện muộn trong lúc huấn luyện | Dùng dataclass hoặc Pydantic cho `Config` và YAML |
| G-09 | Hai biến thể schema CSV mẫu (thứ tự cột khác nhau, BOM) | Phụ thuộc tên cột, không phụ thuộc thứ tự | Ghi rõ trong hướng dẫn dữ liệu |

Minh họa `[PROPOSED]` (chưa tồn tại trong kho mã):

```python
from typing import Optional
from pydantic import BaseModel, Field

class PredictRequest(BaseModel):
    file_path: str = Field(min_length=1)
    lookback: int = Field(default=400, gt=0)
    pred_len: int = Field(default=120, gt=0)
    temperature: float = Field(default=1.0, gt=0, le=2.0)
    top_p: float = Field(default=0.9, gt=0, le=1.0)
    sample_count: int = Field(default=1, ge=1, le=5)
    start_date: Optional[str] = None

class ErrorResponse(BaseModel):
    code: str
    message: str
    request_id: Optional[str] = None
```
