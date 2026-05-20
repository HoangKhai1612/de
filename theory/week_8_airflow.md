# 🕰 Tuần 8: Orchestration với Apache Airflow

## 🎯 Mục tiêu
- Hiểu được Orchestration là gì và tại sao script Cronjob thủ công lại không phù hợp cho Enterprise.
- Kiến trúc, Cài đặt và vận hành hệ thống Apache Airflow.
- Lên lịch xây dựng DAGs mạnh mẽ, xử lý trễ hạn (catching up), lỗi (retries), và cơ chế giám sát.

## 🛠 Công cụ & Môi trường cần thiết
- Môi trường Local: Docker Compose chứa trọn bộ stack Airflow khởi tạo qua image `apache/airflow:X.X.X-python3.X` chính chủ.
- Visual Studio Code kết nối Local hoặc WSL2 để code DAG.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Airflow Core Concepts
- **DAG (Directed Acyclic Graph):** Chữ "Graph" - Đồ thị biểu diễn công việc dưới dạng Code Python. Mỗi Workflow (ví dụ chạy Job ETL) là một DAG.
- **Task:** Nút trên đồ thị - Một hành động cụ thể. Được thực thi qua **Operator** (BashOperator chạy lệnh shell, PythonOperator chạy hàm python, PostgresOperator gọi SQL, v.v.).
- **Dependencies (Phụ thuộc):** Dấu mũi tên `>>` hoặc `<<`. Ràng buộc Job 2 chỉ chạy sau khi Job 1 báo "Success".
- **XCom (Cross-Communication):** Phương tiện để chia sẻ trạng thái giá trị nhẹ (nhỏ hơn 48KB) giữa hàm `Extract()` ở Task A và hàm `Transform()` ở Task B.
- **Hooks & Connections:** Trung tâm lưu trữ mật khẩu kết nối tập trung (API, Database SSH). Khiến các Task hoàn toàn sạch bóng khỏi mật khẩu bị hardcode.
- **Sensors:** Một loại Operator đặc biệt "đứng đợi vô tận" cho đến khi Event đến. (VD: "Đợi khi nào file test.csv có mặt ở S3 bucket thì mới bung Job tiếp theo chạy").

### 1.2 Kiến trúc Nội bộ Airflow
1. **Webserver:** UI Giao diện cho Admin để vẽ cây DAG, bật tắt lịch, xem logs.
2. **Scheduler:**  Động cơ trung tâm. Liên tục đánh giá DAGs dựa trên thời gian và quăng task sẵn sàng vào Executor.
3. **Database (Metadata DB):** Chứa các dữ liệu DAG, logs, lịch sử chạy, XComs. Không dùng SQLite ở Production (Hãy dùng Postgres hoặc MySQL).
4. **Executor/Worker:** Cánh tay làm việc thật (Ví dụ Sequential cho test, Celery hoặc Kubernetes cho scale chạy song song 1000 task cùng lúc).

### 1.3 Quản Lý Cơ Chế Lỗi và Thời Gian
- **Catchup & Backfill:** Nếu bạn có lịch DAG chạy ngày 1-1, nhưng Server bị ngắt kết nối đến ngày 4-1. Nếu bật `catchup=True` (Mặc định), Scheduler sẽ tự động đẻ khẩn cấp 3 Job của ngày 1, 2, 3 bù vào để lấp đầy hố trống lịch sử.
- **Retry Logic:** Thiết lập task hỏng mạng thì Retry 3 lần, delay 5 phút rồi Retry. Nếu rớt sạch mới gửi Alert Tele/Slack.

---

## 2. Hướng dẫn sử dụng & Mẫu DAG Chuẩn
Một DAG mẫu gồm trích xuất dữ liệu và load dữ liệu theo sự phụ thuộc.

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.bash import BashOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from datetime import datetime, timedelta

# Arguments chia sẻ cấu hình mặc định (Mỗi task tự lấy)
default_args = {
    'owner': 'data_team',
    'depends_on_past': False, # Nếu hôm qua chết, Job hôm nay có chạy không? -> Có
    'email_on_failure': True,
    'email': ['admin@mycompany.com'],
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

# Bước 1: Khai báo DAG Context
with DAG(
    dag_id='my_ecommerce_etl_pipeline',
    default_args=default_args,
    description='Nhập dữ liệu doanh thu E-com hằng ngày',
    schedule_interval='0 3 * * *', # Chạy mỗi 3h sáng
    start_date=datetime(2023, 1, 1),
    catchup=False,  # Khóa lại, không bắt lấp hố dữ liệu
    tags=['core', 'sales'],
) as dag:

    # Bước 2: Tạo Tasks/Operators
    # Task 1: Check server API bằng lệnh curl
    check_api = BashOperator(
        task_id='check_api_status',
        bash_command='curl -sIf https://api.myshop.com'
    )

    # Khai báo hàm python
    def _extract_sales_data(ti):
        # ... logic gọi request ...
        print("Đang crawl data từ API")
        # Chia sẻ 1 text note qua XCom
        ti.xcom_push(key='file_name', value='data_export_01.csv')

    # Task 2: Chạy code Python extract Data
    extract_data = PythonOperator(
        task_id='extract_sales_data',
        python_callable=_extract_sales_data
    )

    # Task 3: Load bằng SQL có tham số
    load_db = PostgresOperator(
        task_id='load_to_dwh',
        postgres_conn_id='my_postgres_dwh_conn',
        sql="""
            INSERT INTO raw_sales (date_create) 
            VALUES (CURRENT_DATE);
        """
    )

    # Bước 3: Định hình Sự phụ thuộc DAG Graph
    check_api >> extract_data >> load_db
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Hiểu rõ Execution Date (Chướng ngại vật số 1 cho người mới):** Khi Airflow lên lịch DAG Daily lúc Nửa Đêm (`0 0 * * *`) tên là Ngày `2024-10-02`. Nó sẽ kích hoạt bắt đầu Chạy VÀO LÚC **Nửa đêm của ngày `2024-10-03`**. Bởi vì nguyên tắc ETL của Data Engineer: "Chúng ta chỉ lấy được thông tin Data của cả 1 ngày trọn vẹn khi ngày đó ĐÃ KẾT THÚC". Logic này là logic `execution_date` khiến Airflow khác biệt và uy quyền trong giới DE.
2. **Không làm tính toán trong File gốc DAG:** Trình Scheduler Parsing quét cập nhật file DAG của bạn mỗi 30 giây liên tục. Nếu bạn gán 1 lệnh mở kết nối DB trên đầu File `(import.. connection)`, bạn sẽ làm DB sập vì vài nghìn Connection đổ về mỗi giờ. Lệnh xử lý Dữ liệu phải bỏ gọn trong ruột hàm nội bộ hoặc gọi file `.py` bên ngoài.
3. **No Catchup:** Mặc định tham số `catchup` là True. Khi bạn cấu hình `start_date` lùi về tận 3 năm ngoái, ngay khi bạn Bật công tắc ON. Airflow sẽ kích sát CPU bắn ra **1000** Pipeline Jobs chạy cùng lúc làm sập máy Server. Điểm danh: `catchup=False`.

---

## 4. Tài liệu tham khảo
- [Airflow Best Practices (Official)](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html)
- [Marc Lamberti - Astronomer](https://www.youtube.com/@MarcLamberti) (Kênh cực đỉnh và duy nhất bạn cần xem để hiểu rõ bản chất Airflow Mới Nhất).
