# 🐧 Tuần 4: Linux, Bash & Git Pro

## 🎯 Mục tiêu
- Thành thạo môi trường hệ điều hành Linux/Unix để deploy và vận hành hệ thống dữ liệu.
- Tự động hóa các tác vụ quản trị, monitor bằng Bash Script.
- Làm chủ Git nâng cao để làm việc nhóm hiệu quả, không gây xung đột (conflict) code và tuân thủ quy trình CI/CD.

## 🛠 Công cụ & Môi trường cần thiết
- **Hệ điều hành:** Linux (Ubuntu/CentOS), WSL2 trên Windows, hoặc macOS Terminal.
- **Git:** Git CLI cài đặt trên máy.
- **Tài khoản:** GitHub, GitLab hoặc Bitbucket.

---

## 1. Khái niệm & Lý thuyết chi tiết

### 1.1 Khám phá Linux Core
- **Kiến trúc Linux:** Kernel, Shell, User space, File system hierarchy (`/etc`, `/var`, `/home`, `/usr`).
- **Phân quyền (Permissions):** User, Group, Others (`chmod`, `chown`). Khái niệm siêu người dùng (root/sudo).
- **Process Management:** Quản lý tiến trình đang chạy (`ps`, `top`, `htop`, `kill`). Background vs Foreground processes (`&`, `bg`, `fg`).
- **Networking cơ bản:** Kiểm tra kết nối mạng và port (`ping`, `curl`, `netstat`, `ss`, `lsof`).

### 1.2 CLI Mastery (Các lệnh nâng cao cho dòng lệnh)
- **`grep`:** Tìm kiếm nội dung trong file theo chuỗi hoặc Regex. (Cực hữu ích để debug log).
- **`awk` & `sed`:** Công cụ thao tác, xử lý text mạnh mẽ thay thế một kịch bản code phức tạp.
- **`find`:** Tìm file/thư mục dựa theo tên, kích thước, ngày sửa đổi, v.v.
- **Piping & Redirection (`|`, `>`, `>>`):** Chuyển output dữ liệu của lệnh này làm input cho lệnh kia (ví dụ lấy 10 lỗi đầu tiên trong file log).

### 1.3 Tự động hóa bằng Bash Script
- **Biến & Mảng:** Khai báo, tính toán giá trị mảng.
- **Cấu trúc điều khiển:** `if-else`, vòng lặp `for`/`while`, `case`.
- **Hàm & Thu số (Arguments):** Truyền dữ liệu `$1`, `$2` vào script.
- **Cronjob:** Tiện ích lập trình tác vụ theo thời gian (ví dụ: Chạy backup database lúc 2h sáng).

### 1.4 Làm việc nhóm với Git Pro
- **Chiến lược phân nhánh (Branching Strategy):** Git Flow, GitHub Flow. Chuyên biệt hóa các nhánh: `main` (Production), `develop` (Staging), `feature/abc` (Tính năng mới).
- **Code Review:** Quy trình tạo Pull Request (PR) hoặc Merge Request (MR).
- **Xử lý Conflicts (Xung đột):** Đọc Diff, thống nhất code khi hai người cùng sửa một dòng.
- **Các lệnh Nâng cao:**
  - `git rebase`: Gộp lịch sử commit thẳng gọn thay vì tạo "mạng nhện" merge commits. Cảnh báo: KHÔNG REBASE trên nhánh public!
  - `git cherry-pick`: Lấy một commit duy nhất từ nhánh khác dán qua nhánh hiện tại (Hữu ích khi hotfix).
  - `git stash`: Cất tạm thay đổi đang viết dang dở vào môt bộ đệm để đổi nhánh khác làm việc gấp.

---

## 2. Hướng dẫn sử dụng & Cú pháp quan trọng

### 2.1 Lọc file log tìm Error lớn nhất (Linux Command)
```bash
# Tìm tất cả file log .txt, đếm số chữ "ERROR", sau đó sắp xếp.
cat /var/log/application/*.txt | grep -c "ERROR" | sort -nr
```

### 2.2 Viết Bash script backup Database tự động
```bash
#!/bin/bash
# Script backup CSDL PostgreSQL
DB_NAME="my_database"
BACKUP_DIR="/backups/db"
DATE=$(date +%Y-%m-%d)
OUTPUT_FILE="$BACKUP_DIR/${DB_NAME}_$DATE.sql"

# Khởi tạo thư mục nếu chưa có
mkdir -p $BACKUP_DIR

# Tiến hành dump CSDL
pg_dump -U postgres -d $DB_NAME -F p -f $OUTPUT_FILE

if [ $? -eq 0 ]; then
  echo "Backup successfully completed to $OUTPUT_FILE"
  # Gửi slack notification tại đây qua curl API (Bài tập tự làm)
else
  echo "Backup failed. Please check the logs." >&2
fi
```
Lên lịch bằng cron (`crontab -e`): Dòng dưới biểu thị chạy vào lúc 02:00 sáng mỗi ngày.
`0 2 * * * /path/to/backup_script.sh`

### 2.3 Resolving Git Conflicts & Rebase nâng cao
```bash
# Tình huống: Bạn ở nhánh feature/data_pipeline và muốn cập nhật code mới nhất từ team ở nhánh main.
git checkout main
git pull origin main       # Kéo thay đổi mới nhất
git checkout feature/data_pipeline

# Rebase feature nhánh lên đầu nhánh main:
git rebase main

# Nếu có Error báo conflict (xung đột):
# Mở Editor/VS Code để sửa các phần bao quanh bằng <<<< và >>>>
git add file_da_sua.py     # Đánh dấu đã sửa xung đột
git rebase --continue      # Tiếp tục rebase cho đến khi hoàn tất

# Đẩy code lên nhưng ghi đè lịch sử (DO CHÚNG TA ĐÃ REBASE):
git push origin feature/data_pipeline --force-with-lease
```

---

## 3. Best Practices & Lưu ý (Dành cho Data Engineer)

1. **Automation-first Mindset:** Hãy tập thói quen tự động hóa (Automate) mọi thao tác bạn phải lặp lại tay bộ quá 3 lần. Dùng bash script cho môi trường server, cronjob để lặp tác vụ nhỏ lẻ trước khi dùng Airflow.
2. **Commit rõ nghĩa:** Cách đọc commit message cũng phản ánh độ chuyên nghiệp (ví dụ: Thay vì rác như "Fix bug", hãy viết chuẩn "fix(db): Update batch insert chunk size to prevent memory leak"). Tham khảo chuẩn **Conventional Commits**.
3. **Cẩn trọng lệnh Root/Sudo:** Không bao giờ chạy các lệnh dạng `rm -rf /` hay cấp quyền `chmod 777` (Full Control) bừa bãi.
4. **Git stash thần thánh:** Data Engineer hay phải chuyển "context" để vá lỗi (hotfix) pipeline chạy sai dù đang viết code tính năng dở. Lệnh `git stash` lưu trạng thái chưa commit của bạn để nhảy nhánh thoải mái mà không lo mất dữ liệu.

---

## 4. Tài liệu tham khảo
- [Bash Scripting Cheat Sheet](https://devhints.io/bash) - Bảng cheat sheet tuyệt vời để copy lệnh nhanh.
- [Pro Git Book (Free)](https://git-scm.com/book/en/v2) - Sách miễn phí duy nhất để hiểu mọi thứ nội bộ về Git.
- Game thực hành git: [Learn Git Branching](https://learngitbranching.js.org/) (Rất thú vị, dễ tiếp thu hơn đọc lệnh khô khan).
