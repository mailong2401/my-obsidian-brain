### @UniqueElements (Hibernate Validator)
Kiểm tra tất cả phần tử trong Collection là duy nhất (không trùng lặp).

```java
@UniqueElements(message = "Danh sách email không được trùng lặp")
private List<String> emails;
```

---