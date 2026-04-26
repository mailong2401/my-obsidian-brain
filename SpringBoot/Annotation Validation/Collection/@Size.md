### @Size
Giới hạn độ dài (với String) hoặc số phần tử (với Collection/Map/Array).

```java
@Size(min = 3, max = 50, message = "Username phải từ 3 đến 50 ký tự")
private String username;
```
- `min`: Độ dài tối thiểu (mặc định 0)
- `max`: Độ dài tối đa (mặc định Integer.MAX_VALUE)
- `"ab"` → ❌ (dưới min)
- `"abc"` → ✅

---