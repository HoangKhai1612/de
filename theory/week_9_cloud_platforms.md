# ☁ Tuần 9: Cloud Data Platform (AWS/GCP/Azure)

## 🎯 Mục tiêu
- Vượt qua vòng kìm kẹp của máy móc "vật lý" (Local/On-Premise) để chuyển dịch hạ tầng về dịch vụ Đám mây công cộng (Public Cloud).
- Nắm bắt 3 trụ cột của Đám mây dữ liệu: **Storage** (Lưu trữ tĩnh), **Compute** (Năng lực tính toán), **Data Warehouse** (Kho xử lý cao tần).
- Nhập môn Kiến trúc phần mềm phân tán không Máy chủ (Serverless).
- Chìa khóa vàng về cách tổ chức và bảo mật IAM.

## 🛠 Công cụ & Môi trường cần thiết
- Tài khoản miễn phí (Free Tier) tại **Google Cloud Platform (GCP)** hoặc **Amazon Web Services (AWS)** (Khuyến khích dùng GCP vì BigQuery dễ tiếp cận nhất cho dân Data).
- Google Cloud SDK CLI hoặc AWS CLI cài đặt dưới máy tính.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Object Storage - Hồ dữ liệu Trái tim (Data Lake)
Thay vì sử dụng hệ thống tệp tin phân cấp như Máy Tính Windows (phải chia theo Drive C, Folder A..), Storage Engine Đám Mây lưu theo định nghĩa **"Object"**. Mỗi ảnh, dòng log, file CSV là 1 cấu trúc vô hạn nằm dưới đại dương.
- **AWS:** Amazon S3 (Simple Storage Service).
- **GCP:** Google Cloud Storage (GCS).
- *Ứng dụng DE:* Mọi dữ liệu trích xuất (Lớp Bronze/Silver trong Medallion Architecture) sẽ bay thẳng vào các thư mục vô tận của Bucket trên S3. Chi phí rẻ kỷ lục (chỉ vài Cent/GB/Tháng).

### 1.2 Dịch Vụ Máy Chủ/Năng Lực Tính Toán (Compute)
- Tự Quản lý (IaaS): Bạn thuê cấu hình máy chủ ảo qua mạng và toàn quyền làm chủ HĐH Linux trên đó (Bảo trì tự lo).
   - AWS **EC2**, GCP **Compute Engine**. Ứng dụng: Cắm Airflow 24/7 trên đó.
- Năng lực dùng 1 Lần / Tiêu chuẩn không máy phủ (Serverless - FaaS):
   - Thay vì thuê trọn tháng 1 máy chủ, bạn quăng một đoạn script Python nhỏ lẻ lên đó, Cloud sẽ "Tự bật điện" để chạy code và tắt máy ngay khi kết thúc trong vài giây/phút để tiết kiệm cực điểm năng lượng.
   - AWS **Lambda**, GCP **Cloud Functions**. Ứng dụng: Khi có File Excel thả vào S3 Bucket -> Lambda Kích chạy Event lọc file đẩy qua DB -> Mất chỉ vài đồng/tháng.

### 1.3 Cỗ máy Phân Tích Warehouse Cao Cấp
- Không như Postgres/SQLServer trên mạng, Mảnh ghép Data Engineer đẳng cấp nằm tại các cỗ máy này được thiết kế theo dạng Multi-Massive Parallel Processing (MPP). 
- AWS **Redshift**, GCP **BigQuery**, hoặc Tổ chức độc lập **Snowflake** / **Databricks**.
- Ví dụ **BigQuery**: Kiến trúc lưu trữ (Tách Rời) Serverless, bạn không cần phải trả tiền "Giữ Database", bạn chỉ mất phí Storage (Rẻ) và Phí mỗi lần Cú nhấp chuột `SELECT *` quét dữ liệu (`~5$/1TB scanned`). Quét vài Petabytes chỉ trong mili-giây.

### 1.4 Quyền, Nhận Diện và Secrets (IAM)
- **IAM (Identity and Access Management):** Kiểm soát phân quyền Ai (hoặc cái gì) CÓ QUYỀN tham chiếu những mục nào. (Cấp phát tài khoản: User Account cho con người, Service Account cho một Công Cụ Code giao tiếp chéo).
- **Secret Manager/AWS Secrets:** Nhà băng cất chứa Password DB. Airflow hoặc App Python sẽ dùng SDK của Cloud để chui vào "Băng" móc mật khẩu ra gán vào RAM mà không lộ lên màn hình chữ trắng.

---

## 2. Làm việc qua Command Line SDK
*Dưới đây dùng ví dụ của Google Cloud (GCP SDK)*

**2.1 Xác thực an toàn (Tạo file JSON Service Account)**
Tải file JSON bảo mật cấp phát cho DE từ Console -> Download xuống Local.
```bash
# Cho ứng dụng thư viện báo kết nối Google biết file Json kia nằm ở đâu!
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/my-secret-key.json"
```

**2.2 Tải & Lấy file Data Lake với hệ thống GCS (gsutil)**
```bash
# Đẩy (upload) file log CSV lên thùng chứa Google Cloud System (GS):
gsutil cp ECommerce_Sales_2023.csv gs://my-company-bronze-datalake/

# Đồng bộ đồng loạt cả folder máy lên Cloud
gsutil rsync -r ./local_logs_folder gs://my-company-bronze-datalake/logs/
```

**2.3 Truy vấn BigQuery siêu cấp bằng CLI (bq)**
```bash
# Quét trong DataLake nối thẳng vào Bigquery lấy ra TOP 10 Sales Doanh Thu
bq query --use_legacy_sql=false \
'SELECT Store_ID, SUM(Revenue) FROM `my-project.analytics_gold.sales` GROUP BY 1 LIMIT 10'
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Dữ liệu mây thì Hữu Hạn, Bill Tiền thì Vô Hạn (Cost Awareness):**
   - Sự nguy hiểm ở Cloud Database là Dễ Dàng. Hãy cẩn thận khi gõ: `SELECT * FROM bq_fact_sales` - Cú lệnh tưởng chừng vô hại đó trên Bigquery nếu bảng `sales` nặng `10TB` nghĩa là bạn vừa *bay mất* `~50$` vào quỹ ngân sách tiền túi/công ty chỉ phục vụ trò chơi Debug.
   - Nguyên lý Sống: CHỈ SELECT những cái CỘT CẦN THIẾT. Và áp dụng PARTITION bảng lọc theo `WHERE partition_date = current_date()`. 

2. **Dừng Cấp phát Over-Permission:** Data Engineer thường lười và Setup API KEY phân quyền **"Admin-Full-Access"** để Airflow không báo đỏ dòng. Nếu Key này rò rỉ ra ngoài GitHub, đội bẻ khóa sẽ thâm nhập xài chùa Bitcoin Miner qua EC2 để xé bill tài khoản của bạn `100.000 USD` sau 1 đêm.
   -> *Chỉ cấp quyền "Principle of Least Privilege" - Cấp quyền vừa ĐỦ để ứng dụng đó sửa bảng A (Role: BigQuery Data Editor) và không làm được bất cứ hành động nào khác.*

---

## 4. Tài liệu tham khảo
- [GCP For Data Engineers Specialization (Coursera/Google)](https://www.coursera.org/professional-certificates/gcp-data-engineering)
- [Snowflake Documentation](https://docs.snowflake.com/) (Dành cho công ty Mỹ thịnh hành dùng Snowflake).
