### @DecimalMin và @DecimalMax
Tương tự `@Min/@Max` nhưng hỗ trợ số thập phân và có thuộc tính `inclusive` để xác định có bao gồm giá trị biên hay không.

```java
@DecimalMin(value = "0.01", message = "Giá phải lớn hơn 0")
@DecimalMax(value = "9999.99", inclusive = true, message = "Giá tối đa là 9999.99")
private BigDecimal price;
```
- `value`: Giá trị biên dạng String (hỗ trợ số thập phân)
- `inclusive`: Có bao gồm giá trị biên không (mặc định `true`)

---