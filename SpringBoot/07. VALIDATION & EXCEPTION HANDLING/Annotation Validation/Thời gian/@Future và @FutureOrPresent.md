### @Future và @FutureOrPresent
```java
@Future(message = "Ngày hẹn phải trong tương lai")
private LocalDateTime appointmentDate;

@FutureOrPresent(message = "Ngày hiệu lực không được trong quá khứ")
private LocalDate effectiveDate;
```

---