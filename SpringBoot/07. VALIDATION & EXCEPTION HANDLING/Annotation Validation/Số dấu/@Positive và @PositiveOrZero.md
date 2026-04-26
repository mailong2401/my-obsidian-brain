
### @Positive và @PositiveOrZero
```java
@Positive(message = "Số lượng phải lớn hơn 0")
private int quantity;

@PositiveOrZero(message = "Tồn kho không được âm")
private int stock;
```
- `@Positive`: Phải > 0
- `@PositiveOrZero`: Phải >= 0

---

