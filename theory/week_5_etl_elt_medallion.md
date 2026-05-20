# 🔄 Tuần 5: ETL/ELT & Medallion Architecture

## 🎯 Mục tiêu
- Phân biệt sự khác nhau, ưu nhược điểm giữa quá trình ETL (Extract-Transform-Load) và ELT (Extract-Load-Transform).
- Nắm vững kiến trúc dữ liệu hiện đại cho Data Lakehouse: **Medallion Architecture**.
- Hiểu các pattern thiết kế xử lý dữ liệu chuẩn chỉ, đặc biệt là khái niệm Idempotency (Tính nhất quán) và Data Quality check.

## 🛠 Công cụ & Môi trường cần thiết
- **Ngôn ngữ:** Python/SQL kết hợp.
- **Lưu trữ tĩnh (Data Lake storage):** Amazon S3, Google Cloud Storage, hoặc Local File System (để thực hành).
- **Lưu trữ có cấu trúc (Database):** PostgreSQL, hoặc DuckDB.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Mảng ghép thứ nhất: ETL vs ELT
- **Extract (Trích xuất):** Lấy dữ liệu từ vô số nguồn khác nhau (API, CRM, Mobile App, RDBMS, NoSQL, logs event...) về.
- **Transform (Biến đổi):** Clean data (làm sạch), Join data (kết hợp), Filter (lọc), Aggregate (tổng hợp tính toán).
- **Load (Tải vào):** Lưu vào Data Warehouse (Kho dữ liệu) hoặc Data Lake (Hồ dữ liệu).

👉 **Sự chuyển dịch từ ETL sang ELT:**
- **ETL (Truyền thống):** Phép **T**ransform xảy ra ở máy chủ trung gian (VD bằng Spark, Python, Informatica) TRƯỚC khi đẩy vào Data Warehouse. Nhược điểm: Tốn năng lực tính toán trung gian. Dữ liệu gốc (raw) bị loại bỏ nếu xảy ra lỗi biến đổi thì khó hồi phục.
- **ELT (Hiện đại):** **L**oad xuất dữ liệu nguyên gốc trút tất cả vào Warehouse/Lakehouse trước. Sau đó dùng chính khả năng tính toán khổng lồ của BigQuery/Snowflake/Databricks để làm **T**ransform thông qua SQL. Ưu điểm: Đơn giản hóa kiến trúc, giữ lại dữ liệu thô để audit sau này, tận dụng năng lực tính toán rẻ của Cloud.

### 1.2 Mảng ghép thứ hai: Batch vs Streaming
- **Batch Processing:** Xử lý dữ liệu theo lô (Ví dụ: chạy định kỳ mỗi đêm lúc 1h sáng, tải dữ liệu cả ngày hôm qua). Tối ưu về chi phí và tài nguyên.
- **Streaming Processing:** Xử lý dữ liệu dạng luồng tức thì ngay khi nó được sinh ra (Ví dụ: Theo dõi giao dịch thẻ tín dụng gian lận bằng Kafka, Flink). Đắt đỏ và phức tạp hơn. Đa phần các doanh nghiệp sẽ sử dụng 90% Batch, 10% Streaming cho các use-case đặc biệt.

### 1.3 Kiến trúc Vàng: Medallion Architecture
Được phổ biến bởi Databricks, đây là mô hình phân lớp dữ liệu trong một môi trường Data Lakehouse:
1. **Bronze Layer (Lớp Đồng - Raw Data):**
   - Chứa dữ liệu vừa Extract về ở trạng thái nguyên vẹn nhất (Dữ liệu rác/thô nhặt như JSON, thư mục logs, file nháp CSV). Phục vụ cho Data Scientist lục tìm mẫu và làm backup.
2. **Silver Layer (Lớp Bạc - Cleansed Data):**
   - Lọc bỏ rác, cast định dạng phù hợp (Date sang Date type, JSON -> Bảng), đổi tên cột cho chuẩn, khử bỏ bản ghi trùng. Tại đây dữ liệu "sạch sẽ" và "chuẩn hóa", có thể xài được nhưng chưa sát nghiệp vụ kinh doanh.
3. **Gold Layer (Lớp Vàng - Aggregated/Business Level):**
   - Kết hợp (Join), gom cụm dữ liệu phục vụ các câu hỏi của Doanh nghiệp (Ví dụ: Bảng Summary Báo cáo Doanh thu Tháng). Dành cho BI (PowerBI, Tableau) hoặc Machine Learning sử dụng.

---

## 2. Hướng dẫn sử dụng & Mẫu Flow cơ bản

Giả sử bạn phải tải thông tin User từ Web API vào Hệ thống.

### Bước 1: Extract to Bronze (Dạng Raw JSON)
```python
import json
import requests
from datetime import datetime

# Giả lập lấy data
response = requests.get("https://jsonplaceholder.typicode.com/users")
data = response.json()

# Lưu lại BẢN GỐC xuống Bronze layer (Local folder hoặc S3 bucket)
today = datetime.now().strftime("%Y%m%d")
with open(f"./bronze/users_{today}.json", "w") as file:
    json.dump(data, file)
```

### Bước 2: Load & Clean to Silver (Dạng chuẩn Table)
```python
import pandas as pd

# Đọc từ Bronze
df = pd.read_json(f"./bronze/users_{today}.json")

# Clean (Bạc hóa): Đổi tên cột, tách dict ra thành cột phẳng, lọc Null
df.rename(columns={'name': 'full_name'}, inplace=True)
df['city'] = df['address'].apply(lambda x: x['city'] if isinstance(x, dict) else None)
df = df.drop(columns=['address', 'company'])
df = df.dropna(subset=['full_name', 'email'])  # Bỏ user rác no email

# Lưu xuống DB PostgreSQL layer Silver
# (Hoặc save thành định dạng Parquet cho Data Lake)
from sqlalchemy import create_engine
engine = create_engine('postgresql://user:pass@localhost:5432/lakehouse')
df.to_sql('silver_users', engine, if_exists='replace', index=False)
```

### Bước 3: Aggregate to Gold (Viết bằng SQL qua dbt hoặc Script)
```sql
-- Chạy trên Database Engine chứa bảng Silver
-- Biến đổi phục vụ Business: "Thành phố nào có nhiều User nhất?"
CREATE TABLE gold_user_city_summary AS 
SELECT 
    city,
    COUNT(id) as total_users,
    CURRENT_DATE as report_date
FROM silver_users
GROUP BY city;
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Idempotency (Tính Lũy Đẳng):** Đây là luật sống còn của DE. Một script biến đổi dữ liệu nếu chạy 1 lần hay 100 lần (do Airflow lỡ trigger retry khi lỗi) kết quả lưu xuống DB vẫn luôn giống nhau. Không bị nhân đôi bản ghi (Duplicate records) hay tính tổng dư. Hãy dùng câu lệnh `MERGE INTO`, `UPSERT` thay cho `INSERT`.
2. **Defensive Programming (Data Contract):** Tuyến phòng thủ cuối cùng. Luôn giả định dữ liệu nguồn sẽ tệ. Hãy áp dụng Schema Validation (dùng `pandera`, `pydantic` hoặc `Great Expectations`) vào đầu lớp Silver để block lại dữ liệu sai logic.
3. **Save as Parquet:** Với Data Lake, đừng lưu lớp Silver và Gold bằng định dạng `.csv`. Hãy lưu thành `.parquet` - một định dạng dạng Cột (Columnar format) được mã hóa nén, rất nhanh để truy vấn đọc và tiếm kiệm gấp 10 lần dung lượng đĩa so với CSV.

---

## 4. Tài liệu tham khảo
- [Databricks: Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture) - Khái niệm chuẩn từ tổ chức tạo ra Apache Spark.
- Fundamentals of Data Engineering (O'Reilly): Một cuốn sách giáo khoa hoàn thiện về toàn bộ pipeline dữ liệu.
