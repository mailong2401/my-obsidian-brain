Để có một commit trên GitHub chuẩn chỉnh, chuyên nghiệp và dễ quản lý (đặc biệt khi làm việc nhóm hoặc làm dự án lớn), bạn nên tuân theo các nguyên tắc về thông điệp commit (**Commit Message**), quy trình làm việc và kích thước của commit.

  

Dưới đây là hướng dẫn chi tiết từ A-Z:

  

### 1. Cấu trúc chuẩn của một Commit Message

Một thông điệp commit tốt thường tuân theo chuẩn **Conventional Commits**, gồm các phần sau:

  

Plaintext

```
<type>(<scope>): <short description>

[optional body]

[optional footer]
```

#### Trong đó:

- **`type` (Loại commit - Bắt buộc):** Cho biết commit này làm gì. Các loại phổ biến:
    
      
    - `feat`: Thêm một tính năng mới (feature).
        
          
        
    - `fix`: Sửa lỗi (bug fix).
        
          
        
    - `docs`: Thay đổi tài liệu (README, hướng dẫn,...).
        
          
        
    - `style`: Định dạng code, thêm dấu chấm phẩy, khoảng trắng,... (không ảnh hưởng đến logic).
        
          
        
    - `refactor`: Tái cấu trúc code (không sửa lỗi cũng không thêm tính năng).
        
          
        
    - `perf`: Cải thiện hiệu suất.
        
          
        
    - `test`: Thêm hoặc sửa các bài test.
        
          
        
    - `chore`: Cập nhật build task, file cấu hình, thư viện phụ thuộc,...
        
          
        
- **`scope` (Phạm vi - Tùy chọn):** Nơi mà thay đổi tác động tới, viết trong ngoặc đơn (ví dụ: `auth`, `ui`, `api`, `db`).
    
      
    
- **`description` (Mô tả ngắn - Bắt buộc):**
    
      
    - Viết ngắn gọn, súc tích (dưới 50-72 ký tự).
        
          
        
    - **Không** viết hoa chữ cái đầu tiên (trừ khi là danh từ riêng).
        
          
        
    - **Không** dùng dấu chấm (.) ở cuối câu.
        
          
        
    - Dùng động mệnh lệnh hoặc hiện tại đơn (ví dụ: _add_, _fix_, _update_ thay vì _added_, _fixed_, _updates_).
        
          
        

### 2. Các ví dụ thực tế chuẩn chỉnh

- **Thêm tính năng mới:**
    
      
    
    Bash
    
    ```
    git commit -m "feat(auth): add google login integration"
    ```
    
- **Sửa lỗi:**
    
      
    
    Bash
    
    ```
    git commit -m "fix(cart): resolve crash when checkout with empty items"
    ```
    
- **Cập nhật tài liệu:**
    
      
    
    Bash
    
    ```
    git commit -m "docs(readme): update installation instructions"
    ```
    
- **Chỉnh sửa nhỏ không đổi logic:**
    
      
    
    Bash
    
    ```
    git commit -m "style(home): format code with prettier"
    ```
    

### 3. Nguyên tắc vàng khi Commit code

1. **Commit nhỏ gọn (Atomic Commits):**
    
      
    - Mỗi commit chỉ nên giải quyết **một vấn đề duy nhất** (ví dụ: vừa sửa lỗi đăng nhập vừa sửa giao diện trang chủ thì nên tách làm 2 commit). Điều này giúp dễ dàng xem lại lịch sử (`git log`) hoặc quay lại phiên bản trước (`git revert`) khi có lỗi.
        
          
        
2. **Kiểm tra kỹ trước khi commit:**
    
      
    - Luôn dùng lệnh `git status` và `git diff` để kiểm tra lại các thay đổi xem có thừa file rác (như file log, file cấu hình cá nhân `.env`) không trước khi add.
        
          
        
    - Chỉ add những file liên quan: `git add <tên_file>` thay vì lạm dụng `git add .` (trừ khi bạn chắc chắn tất cả thay đổi đều cần thiết).
        
          
        
3. **Không commit code bị lỗi (broken code):**
    
      
    - Đảm bảo dự án vẫn chạy được (hoặc ít nhất không bị lỗi cú pháp/biên dịch nghiêm trọng) trước khi đưa lên lịch sử commit chung.
        
          
        

### 4. Quy trình thực hiện chuẩn trên Terminal

Bash

```
# 1. Kiểm tra trạng thái các file thay đổi
git status

# 2. Xem chi tiết các thay đổi (nếu cần)
git diff

# 3. Chọn file cần đưa vào staging area
git add src/components/Header.jsx

# 4. Tiến hành commit theo chuẩn
git commit -m "feat(header): add navigation menu items"

# 5. Đẩy code lên GitHub
git push origin main
```

> **Mẹo:** Bạn có thể sử dụng các extension trên VS Code như **GitLens** hoặc các công cụ hỗ trợ như **Commitizen** để tự động tạo khung commit chuẩn theo menu tương tác.
> 
>   

Bạn có đang gặp khó khăn ở phần quản lý nhánh (branch) hay xử lý xung đột (conflict) nào trên Git không?