
### @Past và @PastOrPresent
```java
@Past(message = "Ngày sinh phải trong quá khứ")
private LocalDate birthDate;

@PastOrPresent(message = "Ngày tạo không được trong tương lai")
private LocalDateTime createdAt;