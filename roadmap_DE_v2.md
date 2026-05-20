# 🚀 LỘ TRÌNH DATA ENGINEER CHUẨN CHỈ (PHIÊN BẢN 2.0)

Lộ trình này được tối ưu hóa từ bản gốc, bổ sung các kiến thức "sống còn" trong doanh nghiệp hiện đại, tập trung vào tính tự động hóa, hiệu năng và bảo mật.

---

## 🏗 GIAI ĐOẠN 1: NỀN TẢNG KỸ THUẬT (Tháng 1)
*Mục tiêu: Làm chủ công cụ giao tiếp với dữ liệu và hệ thống.*

### 📅 Tuần 1-2: Advanced SQL & Database Internals
- **Cơ bản:** SELECT, JOINs, Subqueries, CTEs.
- **Nâng cao:** 
    - Window Functions (`RANK`, `LEAD/LAG`, `OVER`).
    - **Query Optimization:** Giải mã `EXPLAIN ANALYZE`, cách tạo và sử dụng `Index` (B-Tree, Hash, GIN) hiệu quả.
    - **Database Internals:** ACID, Isolation Levels, Deadlocks, Locks.
- **Thực hành:** Giải 100 bài tập trên HackerRank/LeetCode (Medium-Hard). Cài đặt và cấu hình PostgreSQL.

### 📅 Tuần 3: Python cho Data Engineering (Nâng cao)
- **Data Structures:** List, Dict, Set, Tuple, Collections module.
- **Thư viện DE:** `pandas`, `requests`, `psycopg2`, `sqlalchemy`.
- **Validation:** Sử dụng **Pydantic** hoặc **Pandera** để định nghĩa và validate schema dữ liệu.
- **Concurrency:** Hiểu về `threading` vs `multiprocessing` vs `asyncio` để tăng tốc pipeline.
- **Clean Code:** `PEP8`, `logging`, `exception handling`, `pytest`.

### 📅 Tuần 4: Linux, Bash & Git Pro
- **Linux:** Thành thạo dòng lệnh (`grep`, `awk`, `sed`, `find`, `top`, `ss/netstat`).
- **Bash Script:** Viết script tự động hóa backup database, gửi thông báo khi task lỗi.
- **Git:** Làm chủ `rebase`, `cherry-pick`, `conflict resolution` và quy trình Code Review qua Pull Request.

---

## ⚡ GIAI ĐOẠN 2: DATA ENGINEERING CORE (Tháng 2)
*Mục tiêu: Xây dựng hệ thống xử lý dữ liệu chuẩn công nghiệp.*

### 📅 Tuần 5: ETL/ELT & Medallion Architecture
- **Concept:** ETL vs ELT, Batch vs Streaming, Idempotency (tính nhất quán).
- **Kiến trúc Hiện đại:** Tìm hiểu **Medallion Architecture** (Bronze/Silver/Gold) - Tiêu chuẩn trong các hồ dữ liệu (Data Lakehouse).
- **Thực hành:** Viết Python pipeline: API → Bronze (Raw) → Silver (Cleaned) → Gold (Aggregated) → PostgreSQL.

### 📅 Tuần 6: Data Modeling & Warehouse
- **Lý thuyết:** OLTP vs OLAP, Fact vs Dimension, Star Schema vs Snowflake.
- **Nâng cao:** Slowly Changing Dimension (SCD Type 1, 2, 3), Surrogate Keys, Grain Control.
- **Design:** Thiết kế DWH cho một hệ thống E-commerce hoặc Ride-sharing thực tế.

### 📅 Tuần 7: Containerization với Docker
- **Docker Core:** Image, Container, Network, Volume.
- **Multi-container:** Sử dụng `Docker Compose` để chạy cả hệ sinh thái (Postgres + Airflow + pgAdmin + Redis).
- **Optimization:** Viết Dockerfile tối ưu (Multi-stage build) để giảm kích thước Image.

### 📅 Tuần 8: Orchestration với Apache Airflow
- **Airflow Core:** DAG, Operators, Hooks, Sensors, XCom.
- **Advanced:** Backfilling, Catchup, Retries logic, SLA, Variable/Connection management.
- **Thực hành:** Xây dựng DAG hoàn chỉnh xử lý lỗi, gửi Alert qua Slack/Telegram khi pipeline fail.

---

## ☁ GIAI ĐOẠN 3: CLOUD & BIG DATA ECOSYSTEM (Tháng 3)
*Mục tiêu: Đưa hệ thống lên mây và mở rộng quy mô.*

### 📅 Tuần 9: Cloud Data Platform (AWS/GCP/Azure)
- **Storage:** S3 (AWS) hoặc GCS (GCP).
- **Compute:** EC2, Lambda (Serverless ETL).
- **Data Warehouse:** **BigQuery (GCP)** hoặc **Snowflake** (Đây là những skill "hái ra tiền").
- **Security:** Quản lý IAM, **Secret Management** (Không bao giờ để API Key trong code).

### 📅 Tuần 10: Big Data & Streaming
- **Distributed Computing:** Kiến trúc Hadoop/Spark. Spark vs Pandas.
- **Kafka:** Hiểu về Pub/Sub, Producer-Consumer, Topics, Partitions.
- **NoSQL:** Khi nào dùng MongoDB, Redis, Cassandra, ElasticSearch.

### 📅 Tuần 11-12: Infrastructure as Code (IaC) & Automation
- **Terraform (Cơ bản):** Dùng code để tạo S3 bucket, RDS instance thay vì bấm tay trên console.
- **CI/CD cho Data:** Setup **GitHub Actions** để tự động chạy Unit Test (`pytest`) và Deploy pipeline khi có code mới.

---

## 🏆 GIAI ĐOẠN 4: PROJECT END-TO-END & CHUYÊN NGHIỆP HÓA (Tháng 4)
*Mục tiêu: Tạo portfolio "Pro" để chinh phục nhà tuyển dụng.*

### 🚀 Dự án Cuối khóa gợi ý: "Real-time Crypto/Stock Analytics Platform"
- **Nguồn:** Steam dữ liệu từ API (CoinGecko, Binance).
- **Pipe:** Kafka (Ingestion) → Spark/Python (Processing) → Cloud DWH (BigQuery) → dbt (Transformation) → Looker/Tableau (Visualization).
- **Phụ trợ:** Deploy bằng Docker Compose trên EC2, quản lý bằng Airflow, CI/CD qua GitHub Actions.
- **Data Quality:** Dùng **Great Expectations** để kiểm tra tính đúng đắn của dữ liệu cuối cùng.

### 📝 Checklist Kỹ năng mềm cho DE
- **System Thinking:** Nhìn tổng thể hệ thống, không chỉ từng tool đơn lẻ.
- **Cost Awareness:** Luôn đặt câu hỏi "Pipeline này chạy tốn bao nhiêu tiền?".
- **Documentation:** Viết README và Technical Docs chuẩn chỉnh để đồng nghiệp dễ bảo trì.

---

### 🎓 Lời nhắn từ Mentor:
Data Engineering là một cuộc chạy Marathon, không phải 100m. Hãy tập trung vào **TƯ DUY giải quyết vấn đề** thay vì chỉ học thuộc lòng các Tool. Khi bạn hiểu bản chất của dữ liệu, tool nào cũng chỉ là phương tiện.

**Chúc bạn thành công trên con đường trở thành một "Kiến trúc sư dữ liệu" thực thụ!**
