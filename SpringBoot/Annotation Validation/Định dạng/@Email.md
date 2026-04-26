### @Email
Kiểm tra địa chỉ email hợp lệ. Hibernate Validator cung cấp thêm thuộc tính `regexp` để tùy chỉnh.

```java
@Email(message = "Email không đúng định dạng")
private String email;
```
- `user@domain` → ✅
- `user@domain.com` → ✅
- `user@` → ❌
- `@domain.com` → ❌

Tùy chỉnh regex riêng:
```java
@Email(regexp = "^[a-zA-Z0-9._%+-]+@gmail\\.com$", 
       message = "Chỉ chấp nhận email Gmail")
private String gmailOnly;
```

---