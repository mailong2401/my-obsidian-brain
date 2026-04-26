### @NotBlank
Kiểm tra chuỗi không được `null` và phải chứa ít nhất một ký tự không phải khoảng trắng (whitespace). Dùng cho trường bắt buộc nhập.

```java
@NotBlank(message = "Tên không được để trống")
private String fullName;
```
- `null` → ❌ Không hợp lệ
- `""` → ❌ Không hợp lệ
- `"   "` → ❌ Không hợp lệ
- `"John"` → ✅ Hợp lệ

---