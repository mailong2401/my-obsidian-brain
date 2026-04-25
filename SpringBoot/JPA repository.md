JPA Repository là một phần cực kỳ quan trọng khi bạn dùng **Spring Boot + Spring Data JPA** để làm backend. Nó giúp bạn **truy cập database mà gần như không cần viết SQL**.

---

# 1. JPA Repository là gì?

**JPA Repository** là một interface giúp bạn thao tác với database (CRUD) dựa trên JPA mà **Spring tự generate code phía sau**.

👉 Bạn chỉ cần viết interface, Spring lo phần còn lại.

Ví dụ:

```java
public interface UserRepository extends JpaRepository<User, Long> {
}
```

➡️ Chỉ vậy thôi là bạn đã có sẵn:

- save()
    
- findById()
    
- findAll()
    
- delete()
    
- count()
    

---

# 2. Cấu trúc cơ bản

```java
JpaRepository<Entity, ID>
```

Ví dụ:

```java
JpaRepository<User, Long>
```

- `User` → Entity (table trong DB)
    
- `Long` → kiểu của primary key
    

---

# 3. Các method có sẵn (rất quan trọng)

## CRUD cơ bản

```java
userRepository.save(user);        // insert hoặc update
userRepository.findById(1L);      // tìm theo id
userRepository.findAll();         // lấy tất cả
userRepository.deleteById(1L);    // xoá theo id
userRepository.count();           // đếm số lượng
```

---

## Pagination + Sorting

```java
Page<User> users = userRepository.findAll(PageRequest.of(0, 10));
```

```java
List<User> users = userRepository.findAll(Sort.by("name").ascending());
```

---

# 4. Query Method (cực kỳ mạnh)

Spring cho phép bạn viết query bằng **tên hàm**

## Ví dụ:

```java
List<User> findByName(String name);
```

➡️ Tự động generate:

```sql
SELECT * FROM user WHERE name = ?
```

---

## Các keyword thường dùng

|Method|Ý nghĩa|
|---|---|
|findByName|tìm theo name|
|findByAgeGreaterThan|age >|
|findByEmailContaining|LIKE %email%|
|findByNameAndAge|AND|
|findByNameOrAge|OR|
|findByCreatedAtBetween|BETWEEN|

---

## Ví dụ nâng cao:

```java
List<User> findByAgeGreaterThan(int age);
List<User> findByNameContaining(String keyword);
List<User> findByEmailAndStatus(String email, String status);
```

---

# 5. @Query (viết JPQL hoặc SQL)

Khi query phức tạp → dùng `@Query`

## JPQL

```java
@Query("SELECT u FROM User u WHERE u.name = :name")
List<User> findUserByName(@Param("name") String name);
```

---

## Native SQL

```java
@Query(value = "SELECT * FROM user WHERE name = :name", nativeQuery = true)
List<User> findUserNative(@Param("name") String name);
```

---

# 6. Custom Repository (nâng cao)

Khi logic phức tạp:

### B1: Tạo interface custom

```java
public interface UserRepositoryCustom {
    List<User> findUserCustom();
}
```

### B2: Implement

```java
public class UserRepositoryImpl implements UserRepositoryCustom {
    @PersistenceContext
    private EntityManager em;

    public List<User> findUserCustom() {
        return em.createQuery("SELECT u FROM User u", User.class).getResultList();
    }
}
```

### B3: Kết hợp

```java
public interface UserRepository extends JpaRepository<User, Long>, UserRepositoryCustom {
}
```

---

# 7. Các loại Repository trong Spring

|Interface|Mô tả|
|---|---|
|Repository|cơ bản|
|CrudRepository|CRUD|
|PagingAndSortingRepository|phân trang|
|**JpaRepository**|full tính năng (dùng nhiều nhất)|

👉 Thực tế: **luôn dùng JpaRepository**

---

# 8. Lifecycle (cách nó hoạt động bên trong)

Khi bạn gọi:

```java
userRepository.findByName("Naga")
```

Spring sẽ:

1. Parse tên method
    
2. Convert thành JPQL
    
3. Gọi EntityManager
    
4. Mapping result → object
    

➡️ Bạn không cần viết SQL

---

# 9. Ưu điểm

- Không cần viết SQL
    
- Code nhanh, clean
    
- Tích hợp tốt với Spring Boot
    
- Hỗ trợ pagination, sorting
    
- Query dynamic cực mạnh
    

---

# 10. Nhược điểm

- Query phức tạp → khó debug
    
- Method name dài
    
- Performance đôi khi kém hơn SQL custom
    

---

# 11. Best Practice (quan trọng khi đi làm)

✔ Không viết query quá dài bằng method name  
✔ Dùng `@Query` cho query phức tạp  
✔ Tách service layer riêng  
✔ Dùng DTO projection khi cần tối ưu

---

# 12. Ví dụ thực tế chuẩn

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    List<User> findByEmail(String email);

    @Query("SELECT u FROM User u WHERE u.age > :age")
    List<User> findUserOlderThan(@Param("age") int age);
}
```

---

# 13. Luồng thực tế trong project

```
Controller → Service → Repository → Database
```

---

# 14. Khi nào nên dùng JPA Repository?

✔ CRUD app (99% project)  
✔ REST API  
✔ Microservices  
✔ Backend thông thường

---

# Kết luận

JPA Repository giúp bạn:

- Viết backend nhanh hơn rất nhiều
    
- Không cần viết SQL thủ công
    
- Tập trung vào business logic
    

---

Nếu bạn muốn, mình có thể:

- Vẽ sơ đồ kiến trúc Spring Boot + JPA
    
- So sánh JPA vs MyBatis vs JDBC
    
- Hoặc cho bạn bài tập thực hành CRUD chuẩn đi làm 🚀