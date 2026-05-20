# 📚 Tuần 1-2: Advanced SQL & Database Internals

## 🎯 Mục tiêu
- Làm chủ ngôn ngữ SQL từ cơ bản đến nâng cao để truy vấn, biến đổi dữ liệu một cách hiệu quả.
- Hiểu sâu về cách thức hoạt động bên trong của Hệ quản trị cơ sở dữ liệu (Database Internals) để tối ưu hóa truy vấn và thiết kế hệ thống tốt hơn.
- Nắm vững các công cụ phân tích hiệu năng và xử lý đồng thời (concurrency) trong database.

## 🛠 Công cụ & Môi trường cần thiết
- **Database:** PostgreSQL (phiên bản 14 trở lên được khuyến nghị).
- **Công cụ quản lý:** pgAdmin, DBeaver, hoặc DataGrip.
- **Môi trường thực hành:** HackerRank, LeetCode, hoặc database mẫu (như DVD Rental Database của PostgreSQL).

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Cơ bản về SQL
- **SELECT, FROM, WHERE:** Cú pháp cơ bản để trích xuất và lọc dữ liệu.
- **JOINs (INNER, LEFT, RIGHT, FULL):** Cách kết hợp dữ liệu từ nhiều bảng dựa trên khóa (keys).
- **Subqueries (Truy vấn con):** Truy vấn lồng bên trong truy vấn khác (ở mệnh đề SELECT, FROM, hoặc WHERE).
- **CTEs (Common Table Expressions):** Sử dụng mệnh đề `WITH` để tạo các bảng tạm thời trong bộ nhớ giúp truy vấn dễ đọc, dễ bảo trì hơn so với Subqueries lồng nhau phức tạp.

### 1.2 SQL Nâng cao (Advanced SQL)
- **Window Functions:** Các hàm cho phép thực hiện tính toán trên một tập hợp các dòng có liên quan đến dòng hiện tại (gọi là window) mà *không làm hợp nhất (group)* các dòng đó thành một dòng duy nhất.
    - `RANK()`, `DENSE_RANK()`, `ROW_NUMBER()`: Đánh số thứ tự hoặc xếp hạng.
    - `LEAD()`, `LAG()`: Truy cập dữ liệu của dòng đứng trước hoặc đứng sau dòng hiện tại (rất hữu ích trong chuỗi thời gian).
    - Mệnh đề `OVER (PARTITION BY ... ORDER BY ...)`: Xác định phân vùng dữ liệu và thứ tự trong từng phân vùng để áp dụng window function.

### 1.3 Tối ưu hóa truy vấn (Query Optimization)
- **EXPLAIN & EXPLAIN ANALYZE:** Công cụ để xem kế hoạch thực thi (Execution Pla n) của Database Engine cho một chuỗi truy vấn. `EXPLAIN ANALYZE` thực thi truy vấn thật và báo cáo thời gian thực tế.
- **Index (Chỉ mục):** Cấu trúc dữ liệu giúp tăng tốc thao tác đọc (nhưng làm chậm thao tác ghi do phải cập nhật index).
    - **B-Tree Index:** Mặc định trong PostgreSQL, tốt cho các phép so sánh `=`, `<`, `>`, `BETWEEN`.
    - **Hash Index:** Tốt cho phép so sánh bằng (`=`).
    - **GIN (Generalized Inverted Index):** Tốt cho việc tìm kiếm full-text hoặc tìm kiếm trong các mảng, tập hợp (JSONB).
- **Các kỹ thuật tối ưu:** Tránh dùng `SELECT *`, filter dữ liệu càng sớm càng tốt (trong bước `WHERE` hoặc `JOIN`), sử dụng index một cách cẩn thận, quản lý thống kê (statistics) của bảng (`ANALYZE table_name`).

### 1.4 Database Internals
- **ACID Properties:** Đảm bảo độ tin cậy của giao dịch (Transaction).
    - **Atomicity:** Giao dịch là một khối thống nhất (All or Nothing).
    - **Consistency:** Dữ liệu luôn ở trạng thái hợp lệ trước và sau giao dịch.
    - **Isolation:** Các giao dịch chạy đồng thời không ảnh hưởng lẫn nhau.
    - **Durability:** Khi giao dịch hoàn tất, dữ liệu được ghi nhận vĩnh viễn (ngay cả khi mất điện).
- **Isolation Levels:** Mức độ cô lập của giao dịch nhằm giải quyết các hiện tượng như Dirty Read, Non-repeatable Read, Phantom Read. Các mức độ: Read Uncommitted, Read Committed (Mặc định trong Postgres), Repeatable Read, Serializable.
- **Locks (Khóa):** Cơ chế chặn các giao dịch khác tác động lên dòng/bảng đang được xử lý để tránh xung đột dữ liệu.
- **Deadlocks:** Tình trạng hai hoặc nhiều giao dịch chờ đợi lẫn nhau nhả khóa, dẫn đến "treo" vĩnh viễn (Database engine thường sẽ tự phát triện và kill một transaction để giải phóng deadlock).

---

## 2. Hướng dẫn sử dụng & Cú pháp quan trọng

### 2.1 CTE (Common Table Expression)
```sql
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
)
SELECT region, total_sales
FROM regional_sales
WHERE total_sales > (SELECT AVG(total_sales) FROM regional_sales);
```

### 2.2 Window Functions
```sql
-- Xem mức lương của nhân viên so với mức lương trung bình cùng phòng ban
SELECT 
    employee_id, 
    department_id, 
    salary,
    AVG(salary) OVER(PARTITION BY department_id) AS avg_dept_salary,
    RANK() OVER(PARTITION BY department_id ORDER BY salary DESC) as salary_rank
FROM employees;
```

### 2.3 Phân tích truy vấn với EXPLAIN ANALYZE
```sql
EXPLAIN ANALYZE 
SELECT * FROM users WHERE email = 'test@example.com';
```
*(Kết quả sẽ trả về Sequence Scan hay Index Scan, chi phí (cost) và thời gian chạy thực tế.)*

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Hiểu rõ Execution Plan:** Khi một câu SQL chạy chậm, phản xạ đầu tiên của DE phải là dùng `EXPLAIN ANALYZE` để xem nó bị scan toàn bảng (Seq Scan) ở đâu, có dùng index không, phép JOIN nào đang làm chậm hệ thống (Hash Join, Nested Loop...).
2. **Indexing Strategy:** Không phải báo bảng nào cũng gắn Index bừa bãi. Index làm tốn dung lượng ổ cứng và làm chậm tốc độ `INSERT`/`UPDATE`/`DELETE`. Phải chọn trường thường xuyên được filter (WHERE) hoặc dùng làm điều kiện JOIN.
3. **Tránh Deadlocks:** Khi viết các script ETL lưu lượng lớn hoặc cập nhật dữ liệu hàng loạt:
   - Hãy truy cập (hoặc cập nhật) các bảng cùng một thứ tự ở mọi transaction.
   - Cố gắng giữ cho các giao dịch (transaction) càng ngắn càng tốt.
4. **Partitioning:** Với các bảng dữ liệu khổng lồ (ví dụ Logs, Transactions hàng tháng), sử dụng Table Partitioning (theo ngày/tháng) của PostgreSQL để tăng hiệu năng truy xuất và dễ xóa/lưu trữ dữ liệu cũ.

---

## 4. Tài liệu & Bài tập thực hành

- **Bài tập:** 
  - HackerRank: SQL Domain (Giải quyết các bài mức độ Medium-Hard).
  - LeetCode: Database category. Hãy tập trung làm các bài có tag `Window Function` hoặc xử lý bài toán `Gaps inside data`.
- **Tài liệu đọc:**
  - [PostgreSQL Official Documentation](https://www.postgresql.org/docs/current/index.html) (Tập trung đọc phần Performance và Internals).
  - Use The Index, Luke! (`use-the-index-luke.com`): Cuốn sách online bắt buộc đọc để hiểu về cách hoạt động của SQL Index.
