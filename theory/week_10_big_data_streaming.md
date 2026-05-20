# 🌊 Tuần 10: Big Data Ecosystem & Tích hợp Data Streaming

## 🎯 Mục tiêu
- Xóa nhòa ranh giới giới hạn thiết bị (Vertical Scale) để nhập môn Thế giới Máy tính Song song có thể mở rộng khổng lồ (Horizontal Scale / Cluster).
- Khái niệm tính toán phân tán MapReduce và kiến trúc bộ chíp HDFS/Spark ecosystem.
- Đổi luật chơi thành dữ liệu Luồng Tức Thời (Real-Time Data Streaming) nhờ "Broker" trung tâm lõi Apache Kafka.
- Hiểu được sự khác biệt các công nghệ NoSQL khi kết hợp pipeline DE.

## 🛠 Công cụ & Môi trường cần thiết
- Năng lực thực hành trên Spark Local / PySpark.
- Chạy cụm Docker Kafka (Broker + Zookeeper) trên máy.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Khởi sinh: Hadoop Ecosystem & Tín chỉ của sự Trễ Ký Ức
- **Vertical Scaling vs Horizontal Scaling:** Thay vì nâng cấp 1 con CPU 16 Cores, RAM 1TB (Chi phí Khủng Khiếp), ta sẽ mua 10 Cái Máy Tính (CPU 4 Core RAM 8GB rẻ mạt) và kết nối chúng bằng Dây LAN. Đó là Horizontal (Scale ngang).
- **HDFS (Hadoop Distributed File System):** Một ổ cứng "Dàn Trải". Khi File CSV lớn 1TB đổ về, nó Cưa Đứt file này thành Từng Phân Mảnh nhỏ (Blocks, ví dụ 128MB) phân chia lưu ở khắp các đĩa ở 10 Máy tính kia.
- **Apache Spark / PySpark:** Thế hệ thứ 2 thay thế cho MapReduce. Đem cơ chế tính Toán Bảng (DataFrame) đặt trực tiếp lên in-Memory (RAM máy tính). Gấp trăm lần tốc độ. Spark Master (Cầm trịch) ra lệnh cho Spark Workers (Nô lệ) thi hành Job filter Data khổng lồ tại cùng một khoảnh khắc ở 10 máy con gộp kết quả trả về bằng RDD/DataFrame logic.

### 1.2 Luồng tức tốc - Hệ thần kinh Apache Kafka
Các cơ sở hạ tầng (Database/API) chạy Streaming sẽ ngập lụt nếu Cửa Hàng App phải Kết nối Bắt 10 vạn lệnh Request/s.
Kafka đứng ra làm Cơ Chế Bus Đệm (Message Broker):
- **Producer:** Chiếc cảm biến IOT Sensor hoặc Điện thoại User sẽ BẮN tín hiệu kiện tin vứt vào. (Lưu Trữ Log Vĩnh Cửu).
- **Topic:** Một "Ống Dẫn" đặc điểm ví dụ ống dẫn *'Click-Log-Hành-Vi'*. Ống này phân tán nhánh rẽ tiếp đựt ngầm trên Cluester bằng **Partitions** (Chi nhánh xử lý).
- **Consumer:** Hệ thống Data Engineer của bạn làm nhiệm vụ đứng ở Quầy Đuôi Ống và *HÚT* (Subscribe) ngay lập tức lượng Data đè trong phân đoạn ống. Data vừa bay vào là Consumer lấy đem vào Spark Streaming xử lý, rồi nhét BigQuery.

### 1.3 Database Thế kỷ NoSQL
Data Engine là kiến trúc Data Lake và NoSQL kết hợp sinh tồn. Thay vì bị ép theo Tabular (SQL Cấu Trúc Hàng/Cột khắt khe):
- **Document Store (MongoDB):** Dữ liệu trả về API dạng JSON? Đẩy nguyên cụm JSON siêu to lồng nhau vào đó mà không cần cắt bảng. Hệ thống Mobile App đổi Schema xoành xoạch cũng rất an toàn.
- **Key-Value Store (Redis):** Cơ sở dữ liệu nằm bám in-Memory RAM. Tốc độ Tra cứu chỉ Mất vài phần ngàn Giây. (Rất hợp tính Caching).
- **Search Engine (ElasticSearch):** Dữ liệu văn bản text không thể xài lệnh "SELECT WHERE name LIKE" của Postgres. ElasticSearch là 1 dạng Đảo ngược Tokenizer xử lý và giúp tìm text y như thuật toán tìm từ khóa trên Google. (Tăng cường Log monitoring hệ thống Kibana).

---

## 2. Hướng dẫn sử dụng & Cú pháp quan trọng

### 2.1 Demo PySpark Script (Khác với thư viện Pandas)
```python
# Yêu cầu cài pyspark
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, sum

# Create Spark Session Driver (Cầm trịch mọi thứ)
spark = SparkSession.builder \
    .appName("Sales_Analyzer_Pro") \
    .getOrCreate()

# Load dữ liệu (Đằng sau có thể là 1 Tỉ Cột nằm vãi tại S3 - Data vô biên, Pandas sập ngay)
df = spark.read.csv("s3://dataset/sales_years/*.csv", header=True, inferSchema=True)

# Computation Transform (Lazy Evaluation) - Nó chưa CHẠY NGAY đâu!
transformed_df = df.filter(col("quantity") > 0) \
    .groupBy("region") \
    .agg(sum("revenue").alias("total_rev"))
    
# Phải có một "Aciton" kéo kết quả nó mới nhào vô thực thi song song Cluster
transformed_df.show() # In màn 
```

### 2.2 Kafka Command Line & Code Cơ Bản (Producer / Consumer Python)
*Start Server:* `docker-compose up kafka`

**Phát tin đi qua Python (`kafka-python` lib)**
```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers='localhost:9092',
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Chạy App đẩy Event
user_event = {"user_id": 99, "action": "ADD_TO_CART"}
producer.send("Ecom_User_Actions_Topic", user_event)
producer.flush() 
```

**Bắt tin về bằng Hút ống Kafka (Bởi DE Job Consumer)**
```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    "Ecom_User_Actions_Topic",
    bootstrap_servers='localhost:9092',
    auto_offset_reset='earliest', # Bắt lấp lại log trước đây nếu cần
    value_deserializer=lambda x: json.loads(x.decode('utf-8'))
)

# Loop Cổng Mở Vĩnh Viễn Không Tắt
for message in consumer:
    # Ở đây Code Transform/nhét Warehouse tức thời.
    print(f"Nhận được giao dịch Rela-time: {message.value}")
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Lazy Evaluation Sự Lười Biếng của Spark:** Điểm khác biệt chí mạng bạn viết code `PySpark` với máy `Pandas` là lệnh tính toán Transform của bạn (Filter, Join) nó chỉ GHI FILE NHÁP VÀO RAM DAG-PLAN. Tốc độ Code Chạy đi qua nháy mắt. Mã code Data thực tế chỉ Chạy "Action" khi bạn gõ câu lệnh cuỗi như `.count(). show(), df.write()`.
2. **Hiểu rõ Partition (Kafka & Spark):** "Độ Chia Nhỏ" của dữ liệu. Nếu bạn có 1 File khổng lồ nhưng bạn Set Partition = 1 (Tương đương cho Hệ Thống 10 máy nhưng bạn chỉ cử 1 Nhân công đi làm bài này, 9 người ngồi ngó thảnh thơi). Phải cấu hình cân bằng Partition Load cho Máy Chạy. Tránh Data Skew (Data đổ lệch tập trung vào 1 máy con bị ngộp CPU bóp cả Team).

---

## 4. Tài liệu Tham khảo
- Khóa Học nổi tiếng nhất: [Data Engineering with Databricks (Hãng tạo ra Spark)](https://www.coursera.org/professional-certificates/databricks-data-engineering)
- Tạp Chí Dữ Liệu: [Confluent Kafka Blog: Giải thích toàn bộ cơ chế của Message Broker](https://www.confluent.io/blog/).
