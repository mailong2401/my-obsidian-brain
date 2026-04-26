### @Digits
Kiểm soát số chữ số phần nguyên (integer) và phần thập phân (fraction).

```java
@Digits(integer = 10, fraction = 2, 
        message = "Số tiền tối đa 10 chữ số và 2 chữ số thập phân")
private BigDecimal amount;
```
- `1234567890.50` → ✅ (10 chữ số nguyên, 2 thập phân)
- `12345678901.50` → ❌ (vượt quá 10 chữ số nguyên)
- `100.123` → ❌ (vượt quá 2 chữ số thập phân)

---