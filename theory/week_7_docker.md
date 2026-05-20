# 🐳 Tuần 7: Containerization với Docker

## 🎯 Mục tiêu
- Hiểu bản chất của Containerization thay thế cho Virtual Machine truyền thống.
- Đóng gói (Dockerize) ứng dụng Data Pipeline Python để chạy mượt mà trên bất kì máy chủ nào.
- Dùng Docker Compose khởi tạo nhanh chóng các môi trường sinh thái Data (Postgres, Redis, Airflow, pgAdmin) chỉ bằng một lệnh.

## 🛠 Công cụ & Môi trường cần thiết
- **Docker Desktop** (hoặc Docker Engine thuần trên Linux).
- Visual Studio Code với extention **Docker**.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Khái niệm Cốt Lõi (Docker Core)
- **Container vs Virtual Machine (VM):** VM "ảo hóa" cả phần cứng và chạy toàn bộ Hệ điều hành (1 VM = 1 Hệ Điều Hành riêng => Tốn tài nguyên). Container "ảo hóa" ứng dụng, chia sẻ nhân Kernel của Hệ điều hành máy chủ tĩnh nên cực nhẹ, bật/tắt trong vài giây.
- **Image:** Bản thiết kế (Read-only blueprint) chứa mã nguồn phần mềm, thư viện, environment variables và Hệ điều hành tối giản.
- **Container:** Một "thực thể đang chạy" từ Image. Nếu Image là file `.exe` thì Container chính là ứng dụng phần mềm đang mở trên RAM.
- **Dockerfile:** Tệp tin kịch bản quy định các bước (`FROM`, `RUN`, `COPY`, `CMD`) để nhào nặn ra 1 Image.

### 1.2 Docker Networking & Volumes
- **Network (Mạng nội bộ):** Mặc định các Container không bị cô lập sẽ không nhìn thấy nhau trừ khi join chung một Network (`Bridge`, `Host`). Thay vì kết nối Database qua `localhost`, chúng sẽ nối nhau bằng tên service mạng.
- **Volume:** Cơ chế chia sẻ tệp vĩnh viễn với máy chủ gốc (Host machine). Mặc định khi một Container bị Xóa (`rm`), dữ liệu đã ghi sẽ chết theo nó. Bằng cách mount Volume, dữ liệu PostgreSQL sẽ lưu trên ổ cứng của bạn ngoài đời thực.

### 1.3 Multi-container Management
- **Docker Compose:** Thay vì gõ 10 câu lệnh dài dòng để chạy 10 Containers kết nối với nhau, ta định nghĩa toàn bộ hạ tầng (Networks, Volumes, Services) trong một file `docker-compose.yml` duy nhất.

---

## 2. Hướng dẫn sử dụng & Cú pháp quan trọng

### 2.1 Viết Dockerfile tối ưu (Multi-stage build) cho PIPELINE PYTHON
```dockerfile
# --------- 1. Stage Builder (Lớn) ---------
FROM python:3.10-slim AS builder
WORKDIR /app

# Khắc phục bộ cài compiler thiếu (nếu có các package C/C++)
RUN apt-get update && apt-get install -y gcc

# Copy requirements và cài đặt vào thư mục đặc biệt
COPY requirements.txt .
RUN pip install --user -r requirements.txt

# --------- 2. Stage Final (Nhẹ, Bảo mật) ---------
FROM python:3.10-slim
WORKDIR /app

# Lấy lại các package đã build ở Stage 1 (bỏ qua gcc làm image xẹp đi đáng kể)
COPY --from=builder /root/.local /root/.local

# Biến môi trường báo Python dùng app trong user local
ENV PATH=/root/.local/bin:$PATH

# Mount Source Code
COPY src/ ./src/

# Chạy App
CMD ["python", "src/main.py"]
```
*(Build Image: `docker build -t data-pipeline-app:v1.0 .` )*

### 2.2 Xây dựng Data Stack với `docker-compose.yml`
```yaml
version: '3.8'

services:
  # Kho dữ liệu
  postgres_dwh:
    image: postgres:15-alpine
    container_name: de_postgres
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password123
      POSTGRES_DB: analytics
    ports:
      - "5432:5432"
    volumes:
      - pg_data:/var/lib/postgresql/data
      
  # Giao diện Database UI
  pgadmin:
    image: dpage/pgadmin4
    container_name: de_pgadmin
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@test.com
      PGADMIN_DEFAULT_PASSWORD: password123
    ports:
      - "8080:80"
    depends_on:
      - postgres_dwh

volumes:
  pg_data:  # Cấu trúc lưu trữ Volume
```
*(Khởi chạy môi trường: `docker-compose up -d`)*

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Hiểu bản chất localhost:** Mã Python chạy trong 1 Container A (Ví dụ: Airflow) muốn gọi vào Container B (Ví dụ: Postgres) chung Docker-Compose, thay vì khai báo connection string `host: localhost`. Cần khai báo dùng Service Name: `host: postgres_dwh`.
2. **Kích thước Image (Layer Caching):** Mỗi lệnh `RUN`, `COPY` trên Dockerfile sẽ tạo ra một layer cache. Hãy đảm bảo file `requirements.txt` được `COPY` và `RUN pip install` ở trên cùng, xếp TRƯỚC LỆNH `COPY src/ ./src/`. Lợi ích: Tốc độ Re-build của bạn sẽ chỉ tốn 1 Giây thay vì tốn 21 Giây chờ tải cài đặt lại library mỗi lần sửa đổi 1 dòng nhỏ của source python.
3. **Mật khẩu hớ hênh:** Tuyệt đối không hardcode AWS Secret Key, Mật Khẩu Database vào Dockerfile. Dùng file `.env` kèm theo và truyền qua lệnh chạy (--env-file).

---

## 4. Tài liệu tham khảo
- [Docker Curriculum (Tutorial)](https://docker-curriculum.com/) - Rất nên đọc để thực hành những lệnh cơ bản.
- [Docker Hub](https://hub.docker.com/) - Tìm kiếm Official Image cho Postgres, Airflow thay vì tự build từ đầu.
