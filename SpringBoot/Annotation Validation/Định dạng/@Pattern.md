### @Pattern
Kiểm tra chuỗi khớp với biểu thức chính quy (Regex).

```java
@Pattern(regexp = "^[0-9]{10}$", message = "Số điện thoại phải có đúng 10 chữ số")
private String phone;
```
- `regexp`: Biểu thức chính quy
- `flags`: Các cờ regex (ví dụ `Pattern.Flag.CASE_INSENSITIVE`)

Một số regex thông dụng:
```java
// Mã bưu điện Việt Nam (6 chữ số)
@Pattern(regexp = "^[0-9]{6}$")

// Chỉ chứa chữ cái và khoảng trắng
@Pattern(regexp = "^[a-zA-Z ]+$")

// UUID format
@Pattern(regexp = "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$")
```

---