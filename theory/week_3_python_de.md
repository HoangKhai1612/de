# 🐍 Tuần 3: Python cho Data Engineering (Nâng cao)

## 🎯 Mục tiêu
- Sử dụng Python hiệu quả để viết các script tự động hóa, pipeline ETL.
- Nắm vững các cấu trúc dữ liệu tối ưu trong Python và sử dụng bộ thư viện tiêu chuẩn/bên ngoài phục vụ DE.
- Đảm bảo chất lượng dữ liệu thông qua Data Validation.
- Cải thiện hiệu suất pipeline Python với lập trình đồng thời (Concurrency).
- Viết code sạch, có cấu trúc tốt, có khả năng maintain (Clean Code & Testing).

## 🛠 Công cụ & Môi trường cần thiết
- **Ngôn ngữ:** Python 3.9+.
- **IDE:** VS Code, PyCharm.
- **Thư viện cần cài:** `pandas`, `requests`, `psycopg2-binary`, `sqlalchemy`, `pydantic`, `pytest`.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Data Structures & Collections (Cấu trúc dữ liệu)
- **Built-in:**
  - `List`: Mảng động, có thứ tự, có thể thay đổi (mutable).
  - `Dict`: Bảng băm (Hash map), key-value, tra cứu với độ phức tạp `O(1)`.
  - `Set`: Tập hợp không có thứ tự, các phần tử là duy nhất, dùng để khử trùng lặp và phép toán tập hợp (Union, Intersection).
  - `Tuple`: Giống list nhưng không thể thay đổi (`immutable`), an toàn dữ liệu, dùng làm key trong hash map được.
- **Collections module (`import collections`):**
  - `defaultdict`: Xử lý lỗi `KeyError` bằng cách gán giá trị mặc định nếu key không tồn tại.
  - `Counter`: Đếm số lượng phần tử xuất hiện trong một iterable một cách cực kỳ nhanh chóng.
  - `namedtuple`: Tuple có kèm tên cho các trường, làm code dễ đọc thay vì truy cập bằng index `tupl[0]`.

### 1.2 Các thư viện cốt lõi cho DE
- **`pandas`:** Thư viện quyền lực để thao tác và phân tích dữ liệu dạng bảng (DataFrames). (Lưu ý: pandas tốt cho dữ liệu tải vào được RAM).
- **`requests`:** Gọi API (RESTful API), crawl data từ web resource bên ngoài.
- **`psycopg2` & `sqlalchemy`:** Kết nối, giao tiếp với PostgreSQL. 
  - `psycopg2`: Driver DB mức thấp.
  - `sqlalchemy`: ORM (Object-Relational Mapper) và Core Engine cung cấp một cầu nối hoàn hảo giữa Python và DB.

### 1.3 Data Validation (Xác thực dữ liệu) với Pydantic / Pandera
Trong hành trình biến đổi dữ liệu, DE cần đảm bảo dữ liệu đầu vào đúng định dạng (kiểu int, string, email, ...) trước khi lưu vào Warehouse.
- **Pydantic:** Dùng Data classes, type hints trong Python để định nghĩa Schema. Nếu dữ liệu JSON tải về từ API không chuẩn, Pydantic sẽ ném ra lỗi rõ ràng ngay lập tức.
- **Pandera:** Dùng khi muốn validate dữ liệu dạng bảng của `pandas` DataFrame (VD: cột 'age' phải > 0, cột 'email' phải có '@').

### 1.4 Concurrency: Tăng tốc Pipeline
- **`threading` (Đa luồng):** Phù hợp với I/O bound tasks (Network request, file I/O). (Giới hạn bởi GIL - Global Interpreter Lock của Python).
- **`multiprocessing` (Đa tiến trình):** Phù hợp với CPU bound tasks (tính toán nặng, biến đổi dữ liệu phức tạp trên nhiều lõi CPU). Tránh được GIL.
- **`asyncio` (Bất đồng bộ):** Phù hợp cho số lượng kết nối I/O cực lớn (VD: Gọi 10,000 API đồng thời, query database liên tục) thông qua event loop đơn luồng.

### 1.5 Clean Code & Error Handling
- **PEP8:** Tiêu chuẩn viết code Python (Formatting, Naming convention).
- **Logging:** Dùng logging module thay cho `print()`. Log cần có level (INFO, WARNING, ERROR, DEBUG) và time stamp để trace bug dễ dàng.
- **Exception handling:** Bẫy lỗi bằng `try...except...finally` cụ thể, hạn chế dùng `except Exception as e:` mù quáng mà không biết lỗi từ đâu.
- **Pytest:** Viết Unit Test để bảo vệ hàm xử lý dữ liệu. Hàm biến đổi Data (Transform) không được có lỗi logic.

---

## 2. Hướng dẫn sử dụng & Mẫu Code

### 2.1 Sử dụng Pydantic
```python
from pydantic import BaseModel, ValidationError

class User(BaseModel):
    id: int
    name: str
    email: str

data = {"id": "123", "name": "John", "email": "john@example.com"}
try:
    user = User(**data)
    print(user.id) # tự động ép kiểu từ '123' (str) sang 123 (int)
except ValidationError as e:
    print(e)
```

### 2.2 Đẩy dữ liệu vào DB (Pandas + SQLAlchemy)
```python
import pandas as pd
from sqlalchemy import create_engine

# Kết nối (Sử dụng psycopg2 backend)
engine = create_engine('postgresql+psycopg2://user:pass@localhost:5432/mydb')

df = pd.DataFrame({'id': [1,2], 'name': ['Alice', 'Bob']})

# Đẩy dữ liệu, 'append' nếu có rồi, không dùng index của pandas
df.to_sql('users_table', engine, if_exists='append', index=False)
```

### 2.3 Logging chuẩn
```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger("ETL_Pipeline")

def extract_data():
    logger.info("Starting data extraction...")
    try:
        # Giả lập lội API
        x = 1 / 0
    except ZeroDivisionError as e:
        logger.error(f"Failed to extract data: {e}")
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Hiểu bộ nhớ pandas:** Nếu xử lý file CSV vài GB mà dùng `pd.read_csv()` trực tiếp có thể gây tràn RAM (OOM). Khắc phục bằng cách sử dụng tham số `chunksize` (đọc từng phần) hoặc dùng framework Big Data (Spark, Dask, Polars).
2. **Batch Processing:** Khi chèn (INSERT) hàng tỷ dòng dữ liệu qua API hoặc DB, luôn dùng batching (chèn từng lô 10.000 dòng) thay vì chèn từng dòng một (Row-by-Row). `df.to_sql(method='multi', chunksize=10000)`.
3. **Mật khẩu & Secrets:** Đừng bao giờ hardcode chuỗi kết nối Database, API Key vào file `.py`. Sử dụng `python-dotenv` để quản lý qua file `.env` hoặc tham chiếu qua OS Environment Variables (`os.getenv()`).
4. **Idempotency:** Code Python viết cho một Job DE luôn phải duy trì tính Idempotency, nghĩa là nếu Job đó chạy đi chạy lại nhiều lần, trạng thái của Database không bị sai lệch (ví dụ: dùng `UPSERT` thay vì `INSERT` mù quáng).

---

## 4. Tài liệu tham khảo
- Documentation của [Pydantic](https://docs.pydantic.dev/) và [Pandas](https://pandas.pydata.org/docs/).
- Sách: "Fluent Python" (Lucien Borges) - Rất tốt để hiểu Cấu trúc dữ liệu và mô hình Python nâng cao.
- [Real Python - Concurrency in Python](https://realpython.com/python-concurrency/)
