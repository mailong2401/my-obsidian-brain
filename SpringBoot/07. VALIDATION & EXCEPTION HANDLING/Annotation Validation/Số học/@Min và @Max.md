### @Min và @Max
Giới hạn giá trị số trong khoảng [min, max] (bao gồm cả 2 đầu).

```java
@Min(value = 18, message = "Tuổi phải từ 18 trở lên")
@Max(value = 65, message = "Tuổi không được vượt quá 65")
private int age;
```

---