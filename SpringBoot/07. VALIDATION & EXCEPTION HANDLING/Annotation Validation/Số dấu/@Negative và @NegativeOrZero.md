### @Negative và @NegativeOrZero
```java
@Negative(message = "Lợi nhuận phải là số âm (lỗ)")
private BigDecimal loss;

@NegativeOrZero(message = "Điểm điều chỉnh không được dương")
private int adjustmentScore;
```

---