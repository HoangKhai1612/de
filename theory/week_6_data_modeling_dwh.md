# 🏛 Tuần 6: Data Modeling & Warehouse

## 🎯 Mục tiêu
- Hiểu được cách thiết kế và tổ chức dữ liệu bên trong một Kho dữ liệu (Data Warehouse - DWH).
- Phân loại tư duy giữa cơ sở dữ liệu xử lý tác vụ thông thường (OLTP) và cơ sở dữ liệu phục vụ phân tích (OLAP).
- Nắm vững kiến trúc đa chiều: Dimensional Modeling (Star Schema & Snowflake Schema).
- Làm chủ nghệ thuật quản lý dữ liệu lịch sử (Slowly Changing Dimensions - SCD).

## 🛠 Công cụ & Môi trường cần thiết
- **Công cụ thiết kế (Design):** Draw.io, Lucidchart, dbdiagram.io để vẽ sơ đồ ERD/Star Schema.
- **Database:** PostgreSQL (Có thể nâng cấp lên Google BigQuery hoặc Snowflake trong thực tế).
- **Transformation tool (Tích hợp thêm):** `dbt` (Data Build Tool) - dùng để thiết kế Data model cực nhanh trong DWH.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Sự khác biệt OLTP vs OLAP
- **OLTP (Online Transaction Processing):** Dành cho hệ thống phần mềm (App, Website). 
    - Đặc điểm: Có nhiều giao dịch ngắn, `INSERT/UPDATE` từng dòng liên tục (Ví dụ: Đặt một ly cafe, thanh toán tiền).
    - Chuẩn hóa (Normalization - 3NF): Dữ liệu phân mảnh vào vô số các bảng con để tránh dư thừa dữ liệu (Tránh update sai sót). Phải `JOIN` rất nhiều bảng để có cái nhìn tổng quát.
- **OLAP (Online Analytical Processing):** Dành cho Kho dữ liệu (DWH) để phân tích báo cáo.
    - Đặc điểm: Có vài truy vấn `SELECT` dài và phức tạp, quét hàng chục triệu bản ghi để tính Tổng, Trung bình, Theo một khoảng thời gian.
    - Phi chuẩn hóa (Denormalization): Gộp chung dữ liệu lại hoặc chấp nhận trùng lặp để cho tốc độ `SELECT` báo cáo ở mức siêu tốc mà không cần thao tác `JOIN` đắt đỏ.

### 1.2 Mô hình Dữ liệu Đa chiều (Dimensional Modeling)
Được phát mình bởi Ralph Kimball, thay thế cho Normalization phức tạp, dữ liệu sẽ được phân tách thành 2 loại bảng: Bảng Sự thật (Fact) và Bảng Chiều (Dimension).

- **Fact Table:** Chứa "Các Sự kiện" mang tính định lượng được sinh ra liên tục. Thường chứa **Foreign Keys** trỏ tới các Dimension và các **Số liệu thống kê (Measures/Metrics)**.
    - Ví dụ: Bảng `fact_sales` có các cột (Date_ID, Store_ID, Product_ID, quantity_sold, total_price, discount).
- **Dimension Table:** Chứa "Bối cảnh" của vạn vật, dùng để mô tả thông tin chi tiết bằng văn bản phục vụ cho việc cắt/lọc/nhóm (Slice and Dice). 
    - Ví dụ: Bảng `dim_product` có (Product_ID, Product_Name, Category, Brand, Weight).

### 1.3 Star Schema vs Snowflake Schema
- **Hệ Giữa/Khuôn Mẫu Ngôi Sao (Star Schema):** 1 bảng Fact nằm tại trung tâm xung quanh nối thẳng ra các bảng Dimension (VD: Bán Hàng -> Sản Phẩm, Cửa Hàng, Thời Gian, Khách Hàng). Cực kỳ phổ biến vì query một bước cực nhanh, dễ hiểu đối với Data Analyst.
- **Hệ Tuyết Rơi (Snowflake Schema):** Các Dimension trong lớp Star lại phân nhánh nhỏ lẻ tiếp tục (VD: Cửa Hàng lại bị chia rẽ sang Dimension Vị Trí -> Quốc Gia -> Thành Phố). Tiết kiệm dung lượng đĩa nhưng `JOIN` làm chậm tốc độ query. Hiện nay **Star Schema** tối ưu và phổ biến hơn tất cả nhờ ổ lưu trữ Cloud DWH rẻ mạt.

### 1.4 Quản lý Lịch sử với SCD (Slowly Changing Dimensions)
Khi một User/Sản phẩm thay đổi tên/địa chỉ theo thời gian, chúng ta lưu thế nào trong DWH để không phá huỷ các báo cáo trong quá khứ?
- **SCD Type 1 (Ghi đè - Overwrite):** Xóa thẳng dữ liệu cũ, ghi đè địa chỉ mới vào dòng cũ. (Đơn giản, nhưng làm mất lịch sử. Phù hợp cho sửa lỗi đánh máy).
- **SCD Type 2 (Thêm dòng mới - Create New Row):** Đây là chuẩn công nghiệp. Thêm một dòng thứ 2 mới hoàn toàn mô tả User đó (dùng một khóa định danh duy nhất - **Surrogate Key** khác). Lưu kèm các cột `valid_from` và `valid_to` đánh dấu vòng đời của dòng đó, cùng với dòng Active (`is_current = True`).
- **SCD Type 3 (Thêm Cột - Add Column):** Chỉ lưu lại bản cũ bằng cách cấp thêm một cột mới (Ví dụ có 2 cột `current_address` và `previous_address`). Hiếm dùng vì ko lưu được nhiều hơn 1 bản sao.

---

## 2. Hướng dẫn sử dụng & Mẫu Thiết Kế

Một kiến trúc điển hình bằng SQL mã hóa cho **Star Schema** (SCD Type 2 xử lý trong Lớp Gold)

```sql
-- DDL tạo Dimension Table (Có SCD Type 2 tracking)
CREATE TABLE dim_customer (
    customer_sk SERIAL PRIMARY KEY,    -- Surrogate Key (Mã tự sinh hệ thống cấp, k dùng mã của Backend web App)
    customer_id INT NOT NULL,          -- Khóa thật (Natural Key) từ App source code đẩy về
    full_name VARCHAR(100),
    city VARCHAR(50),
    
    -- Các trường SCD Type 2
    valid_from TIMESTAMP NOT NULL,
    valid_to TIMESTAMP,                -- NULL nghĩa là phiên bản hiện tại
    is_current BOOLEAN DEFAULT TRUE
);

-- DDL tạo Fact Table (Chứa các ID của Surrogate Dimension)
CREATE TABLE fact_orders (
    order_id INT PRIMARY KEY,
    date_sk INT,                       -- Nối bảng Dim Time/Date
    customer_sk INT REFERENCES dim_customer(customer_sk), -- Nối bản ghi của chính ngày tháng đó, không bị biến dạng sau này
    product_sk INT,                    -- Nối bảng Dim Product
    
    -- Measures
    quantity INT,
    total_amount DECIMAL(10,2)
);
```

**Cách Report Hoạt Động Cực Nhanh!**
```sql
SELECT 
    d.city,
    SUM(f.total_amount) AS revenue
FROM fact_orders f
-- Chỉ tốn 1 phép JOIN với Star Schema
JOIN dim_customer d ON f.customer_sk = d.customer_sk 
GROUP BY d.city
ORDER BY revenue DESC;
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Grain Control:** Khi thiết kế Fact table, câu hỏi xương máu đầu tiên bạn phải trả lời cùng người xem báo cáo: "Hạt Bụi của bảng này là gì?". **Grain** chính là mức độ chi tiết sâu nhất của 1 record. Ví dụ: Dịch vụ ngân hàng (Mỗi lượt Cào thẻ hay là Tóm tắt cuối Ngày của cái Thẻ đó?).
2. **Luôn sử dụng Date/Time Dimension:** Đừng bao giờ lưu hàm tính thứ hai, hàm tính quý theo năm trực tiếp mỗi khi báo cáo. Hãy làm một cấu trúc `dim_date` riêng (chứa trước các ngày mùng Một Tết, Chủ Nhật, Năm nhuần, Quý, Năm tài chính kéo dài đến tận năm 2050), sau đó bắt data join vào `dim_date` qua `date_sk`. Báo cáo tài chính sẽ nhàn hạ cực kỳ.
3. **Surrogate Keys là bắt buộc:** Khóa của Front-Eend/Back-End (`id` = 1, `uuid`) có thể bị các bộ phận trên tái cấu trúc xóa sửa tự động. DWH phải độc lập nên phải tự sinh key tăng tự động (`_sk`). Nó cô lập Data Model khởi tác động từ Data Source cũ/mới bị nhập nhèm lại.

---

## 4. Tài liệu tham khảo
- **"The Data Warehouse Toolkit" của Ralp Kimball:** Đây có thể coi là **"Kinh Thánh Thần Chú"** của bộ môn Data modeling. Mọi DE đều phải có trên giá sách hoặc trên máy tính bản PDF cuốn sách này.
- Mô hình luyện vẽ SQL: Vẽ và xuất SQL cấu trúc thử thông qua [dbdiagram.io](https://dbdiagram.io).
