# ⚙️ Tuần 11-12: Infrastructure as Code (IaC) & Automation CI/CD

## 🎯 Mục tiêu
- Vượt ngoài khuôn khổ viết Python/SQL Data Pipeline bằng việc có thể gõ Code để Xây Dựng cả Cơ Sở Tòa Nhà (Mạng, Máy Chủ, Database) dựa theo cấu hình Cloud.
- Làm quen khái niệm Tư Duy Infrastructure as Code bằng **Terraform**.
- Thay đổi hệ tư tưởng Cài tay thành Tự động hóa Deploy. Kiểm nghiệm pipeline nghiêm ngặt bằng **GitHub Actions** của văn hóa DevOps.

## 🛠 Công cụ & Môi trường cần thiết
- Năng lực Môi Trường: Git đã trơn tru từ Tuần 4.
- **Terraform** Binary cài đặt trên CLI Local.
- Repository GitHub Cài đặt Actions Runners (Dây chuyền CI/CD).
- Tài nguyên Cloud như AWS/GCP chuẩn bị deploy thử nghiệm S3 Bucket rẻ tiền.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 IaC (Infrastructure as Code) - Nắm Sứ Mệnh Máy Chủ
- Thay vì bạn phải cử một Sysadmin/DE vào trình duyệt Web AWS, click chuột vài chục bước tạo Network, mở Port tường lửa, đặt Password RDS cho một Cụm Database Airflow... Bạn có thể Khai Báo nó thành "Văn Bản Markdown-like" (HCL - HashiCorp Configuration Language).
- Lệnh Vung lên `terraform apply` -> Sau 3 Phút, Toàn Bộ Hệ Thống Máy Chủ Bật lên tự động 100%. (Khả Năng Sinh Sản Môi Trường Nhanh Chống như Dev, Prod, Staging Y hệt nhau).
- Góp chung "Sở Hữu Căn Nhà" vào trong 1 Source Code Git để Đội Review Team (Sửa lỗi sai khi config Server yếu).

### 1.2 Cấu trúc cơ bản của hệ sinh thái Terraform
- **Providers:** Các tổ chức/hãng thứ giả như (AWS, Google, Snowflake, DataDog..). Terraform dùng file Binary Provider để kết nối điều khiển hàm API tới hãng đó.
- **State File (`.tfstate`):** Chữ ký lưu giữ Kí Ức. Terraform biết hiện trạng ở Ngoài Đời Cloud kia có thực thể máy chủ đang chạy nào có trùng khớp với Text cấu hình của Bạn không. Đóng vai trò là Não Bộ (Tránh mất tuyệt đối, luôn khóa kĩ trên S3 lưu trữ Remote backend).

### 1.3 Tự Động Hóa DE Delivery với CI/CD (Continuous Integration & Continuous Deployment)
- Công việc DE tạo 1 Airflow DAG mới (Job Tính Doanh Thu). Đẩy Code Git. Nó tự trôi qua Pipeline DevOps như sau:
    - **CI (Tích Hợp Liên Tục):** Gõ Lệnh (Linting/Black/Flake8), chạy `pytest` đánh giá thử (Unit Test, Data Mock) trên máy chủ ảo GitHub. Sạch sẽ Không Lỗi -> Đánh Bật Tích Xanh. (Phát hiện Bug Của Pipeline NGAY do Team code lơ đễnh).
    - **CD (Deploy/Triển Khai Liên Tục):** Nhận lệnh Tích Xanh CI. Lấy Code, Tự động copy đồng bộ (Sync) lên Bucket/Airflow Server thật (Production), Khẩy cập nhật Job, Refresh Cluster.

---

## 2. Hướng dẫn sử dụng & Cú pháp quan trọng

### 2.1 Cấu Hiệu IaC Terraform - Lên mây tạo Storage
*(File `main.tf`)*
```hcl
# Định nghĩa tôi gọi vào hãng Provider AWS Cloud
provider "aws" {
  region = "us-east-1"
  # Authentication tự ngầm gọi từ Biến .env aws_access_key của Máy!
}

# Câu Lệnh: Cho Tôi Xin 1 Đơn Đặt Hàng Storage S3
resource "aws_s3_bucket" "my_data_lake_bucket" {
  bucket = "company-bronze-layer-datalake" 
  
  tags = {
    Environment = "Production"
    Team        = "DataEngineers"
  }
}

# Bảo mật Bucket không public ra mảng lưới Trái Của Chùa Internet
resource "aws_s3_bucket_public_access_block" "block_public" {
  bucket = aws_s3_bucket.my_data_lake_bucket.id

  block_public_acls       = true
  block_public_policy     = true
}
```
**Luồng làm việc Command của DE:**
`terraform init` (Khởi tạo kết nối lõi tải Provider).
`terraform plan` (Hệ thống Scan dự đoán và Report sẽ Thay đổi những gì lên Cloud? Tránh Hỏng Cấu trúc).
`terraform apply` (Chốt, Mở Ví Đẩy Request Lên Thi Hành Trên Account Cloud).

### 2.2 Xây Cấu Trúc Kiểm Duyệt CI Bằng GitHub Actions (Git YAML)
*Tạo trong dường dẫn repo: `.github/workflows/deploy_airflow_dags.yaml`*
```yaml
name: Data Pipeline CI/CD

# Khi có Tác động Đẩy Code lên Nhánh `main`
on:
  push:
    branches: [ "main" ]

jobs:
  test_and_deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Kiểm tra Source Tải App Chứa DAG
      uses: actions/checkout@v3
    
    # Bước Unit Test
    - name: Cài đặt Python và Pytest
      uses: actions/setup-python@v4
      with:
        python-version: '3.10'
    - name: Setup & Test
      run: |
        pip install -r requirements.txt
        pytest tests/  # Máy ảo chặn tại đây nếu có Bug Logic DB

    # Bước tự Deploy (Bắn code DAG lên S3 của Airflow đang liên kết)
    - name: Copy DAG file vào Production AWS S3
      if: success() # Nếu trên Đạt Tích Xanh
      run: aws s3 sync ./dags/ s3://my-airflow-env-bucket/dags/
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_SECRET_KEYID }}  # Mật khẩu trốn ở kho GitHub Secrets
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_PASS }}
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Rule Cấm của State:** File bộ nhớ Terraform State (`.tfstate`) THIẾT YẾU có thể chứa Nhận diện Mật Khẩu Tanh Máu RDS Database dưới dạng text thuần trong file. Hãy gạt Rule vào `.gitignore` để Cấm không đẩy `.tfstate` lên GitHub. Giấu khóa State đó trên một thùng AWS S3 hay GCS Remote để quản lý bởi toàn bộ Team DevOps nội bộ.
2. **Kỹ nghệ Kiểm thử Data (Data Testing/Mocking):** Lập trình DE khác Back-End. Nếu bạn Unit test một Hàm Python có tính Lãi của Doanh Nghiệp, bạn phải `Mock` dữ liệu DataFrame Pandas/SQLAlchemy lên DB RAM (SQLite tạm thờ) trên con CI GitHub action (chứ không Test phá làm sập Server Production DB khổng lồ ở Công Ty, hoặc đợi Job 3 tiếng mới nhận được Error Log!).

---

## 4. Bàn Kết Lộ Trình (Bonus)
Đến đây bạn đã đi qua một Khóa Tu Luyện "Full-Cycle Data Engineering". Từ viết Cú SQL Cổ đại ở Tuần 1, kéo Tải Python Tăng Tốc ở Medallion Lakehouse, và Khóa hạ Bằng Kiến Trúc Quản Lý Phân Tán Cloud CI/CD chuyên nghiệp.

> "Bây giờ bạn đã Đủ Bản Lĩnh Vẽ nên Bức Tranh Dự Án Pipeline E2E Hoàn Hảo để trình diễn Phỏng vấn Tự Tin Trước Nhà Tuyển Dụng!"
