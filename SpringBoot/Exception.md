
---

## 1. Exception là gì và tại sao cần xử lý tập trung?

**Exception (ngoại lệ)** là các tình huống bất thường xảy ra trong quá trình ứng dụng chạy, làm gián đoạn luồng xử lý bình thường.

Trong Spring Boot, việc xử lý exception một cách tập trung mang lại các lợi ích:

- **Phản hồi API nhất quán:** Client luôn nhận được response có cấu trúc JSON thống nhất, dù lỗi gì xảy ra.
- **Code sạch hơn:** Loại bỏ các khối `try-catch` lặp đi lặp lại trong Controller và Service.
- **Bảo mật:** Tránh để lộ thông tin nhạy cảm (stack trace, cấu trúc database) ra bên ngoài.
- **Dễ bảo trì:** Mọi logic xử lý lỗi được tập trung tại một nơi duy nhất.

---

## 2. Phân loại Exception trong Java/Spring

| Loại Exception | Đặc điểm | Ví dụ | Bắt buộc xử lý? |
|:---|:---|:---|:---|
| **Checked Exception** | Kế thừa từ `Exception` (không phải `RuntimeException`). Trình biên dịch bắt buộc phải khai báo `throws` hoặc bọc trong `try-catch`. | `IOException`, `SQLException` | Có |
| **Unchecked Exception** | Kế thừa từ `RuntimeException`. Xảy ra do lỗi logic, thường không thể khôi phục. | `NullPointerException`, `IllegalArgumentException` | Không |
| **Error** | Các lỗi nghiêm trọng từ JVM, thường không thể xử lý. | `OutOfMemoryError`, `StackOverflowError` | Không |

> **Trong thực tế Spring Boot, hầu hết các exception tự định nghĩa đều kế thừa `RuntimeException` (Unchecked)** để tận dụng cơ chế rollback tự động của `@Transactional`.

---

## 3. Ba cách xử lý Exception trong Spring Boot

Spring Boot cung cấp 3 cấp độ xử lý exception, từ cục bộ đến toàn cục:

### Cách 1: `try-catch` trực tiếp (Không khuyến khích)

Cách này xử lý cục bộ ngay trong từng phương thức. Nó chỉ phù hợp cho logic nghiệp vụ đặc biệt cần xử lý phục hồi, còn với API thì gây lặp code và khó duy trì.

```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public ResponseEntity<?> getUser(@PathVariable Long id) {
        try {
            User user = userService.findById(id);
            return ResponseEntity.ok(user);
        } catch (UserNotFoundException e) {
            // ❌ Lặp code, cấu trúc response không thống nhất
            return ResponseEntity.status(HttpStatus.NOT_FOUND)
                                 .body(Map.of("error", e.getMessage()));
        }
    }
}
```

### Cách 2: `@ExceptionHandler` trong Controller (Cục bộ)

Phương thức được đánh dấu `@ExceptionHandler` sẽ bắt exception ném ra từ chính Controller đó.

```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    // ✅ Chỉ có hiệu lực trong UserController này
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<String> handleNotFound(UserNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(e.getMessage());
    }
}
```

**Hạn chế:** Chỉ áp dụng cho một Controller, không giải quyết được vấn đề trùng lặp nếu có nhiều Controller cần xử lý cùng một loại exception.

### Cách 3: `@RestControllerAdvice` (Toàn cục – Tiêu chuẩn Production)

Đây là cách làm chuẩn cho ứng dụng thực tế. `@RestControllerAdvice` kết hợp `@ControllerAdvice` và `@ResponseBody`, cho phép bạn định nghĩa một class xử lý exception toàn cục cho tất cả Controller.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException e) {
        ErrorResponse error = new ErrorResponse(e.getMessage(), 404);
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

**Ưu điểm:**
- Một nơi duy nhất quản lý toàn bộ exception.
- Response format thống nhất.
- Dễ dàng mở rộng khi có thêm loại exception mới.

---

## 4. Xây dựng hệ thống Exception chuyên nghiệp

### 4.1. Tạo Response Object thống nhất

Client luôn mong đợi một cấu trúc JSON nhất quán. Hãy định nghĩa một class chuyên dụng cho error response:

```java
import com.fasterxml.jackson.annotation.JsonInclude;
import java.time.LocalDateTime;

@JsonInclude(JsonInclude.Include.NON_NULL)
public class ErrorResponse {
    private int status;
    private String message;
    private String errorCode;
    private String path;

    // Constructor đầy đủ
    public ErrorResponse(int status, String message, String errorCode, String path) {
        this.status = status;
        this.message = message;
        this.errorCode = errorCode;
        this.path = path;
    }

    // Constructor ngắn gọn
    public ErrorResponse(String message, int status) {
        this.status = status;
        this.message = message;
    }

    // Getters (không cần Setters để đảm bảo tính bất biến)
    public int getStatus() { return status; }
    public String getMessage() { return message; }
    public String getErrorCode() { return errorCode; }
    public String getPath() { return path; }
}
```

### 4.2. Hệ thống Custom Exception phân cấp

Thay vì một exception chung chung, hãy tạo hệ thống phân cấp để biểu thị rõ loại lỗi:

```java
// Exception gốc của ứng dụng
public abstract class AppException extends RuntimeException {
    protected final int status;
    protected final String errorCode;

    protected AppException(String message, int status, String errorCode) {
        super(message);
        this.status = status;
        this.errorCode = errorCode;
    }

    public int getStatus() { return status; }
    public String getErrorCode() { return errorCode; }
}

// 404 - Resource không tồn tại
public class ResourceNotFoundException extends AppException {
    public ResourceNotFoundException(String message) {
        super(message, 404, "RESOURCE_NOT_FOUND");
    }
}

// 400 - Dữ liệu đầu vào không hợp lệ
public class BadRequestException extends AppException {
    public BadRequestException(String message) {
        super(message, 400, "BAD_REQUEST");
    }
}

// 409 - Xung đột dữ liệu
public class ConflictException extends AppException {
    public ConflictException(String message) {
        super(message, 409, "CONFLICT");
    }
}

// 403 - Không có quyền
public class ForbiddenException extends AppException {
    public ForbiddenException(String message) {
        super(message, 403, "FORBIDDEN");
    }
}
```

### 4.3. GlobalExceptionHandler hoàn chỉnh

Dưới đây là một `GlobalExceptionHandler` toàn diện, sẵn sàng cho production:

```java
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    // ============ App Exception ============
    @ExceptionHandler(AppException.class)
    public ResponseEntity<ErrorResponse> handleAppException(
            AppException e, 
            HttpServletRequest request) {
        
        ErrorResponse error = new ErrorResponse(
            e.getStatus(),
            e.getMessage(),
            e.getErrorCode(),
            request.getRequestURI()
        );
        return new ResponseEntity<>(error, HttpStatus.valueOf(e.getStatus()));
    }

    // ============ Validation Error ============
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException e,
            HttpServletRequest request) {
        
        Map<String, String> fieldErrors = new HashMap<>();
        e.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            fieldErrors.put(fieldName, errorMessage);
        });

        String message = "Validation failed";
        if (!fieldErrors.isEmpty()) {
            message = fieldErrors.values().iterator().next(); // Trả về lỗi đầu tiên
        }

        ErrorResponse error = new ErrorResponse(
            400,
            message,
            "VALIDATION_FAILED",
            request.getRequestURI()
        );
        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // ============ Fallback: Các exception không mong đợi ============
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAllUncaughtException(
            Exception e,
            HttpServletRequest request) {
        
        // Log lỗi đầy đủ cho developer
        // log.error("Unhandled exception: ", e);
        
        ErrorResponse error = new ErrorResponse(
            500,
            "Internal server error. Please contact administrator.",
            "INTERNAL_ERROR",
            request.getRequestURI()
        );
        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

---

## 5. Tích hợp Validation Exception chi tiết

Khi sử dụng Bean Validation (`@Valid`), Spring sẽ ném `MethodArgumentNotValidException` nếu dữ liệu không hợp lệ. Ta có thể xử lý chi tiết từng trường lỗi:

```java
// DTO
public class UserRequest {
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    private String username;

    @Email(message = "Email must be valid")
    @NotBlank(message = "Email is required")
    private String email;
    
    // getters, setters
}
```

```java
// Controller
@PostMapping("/users")
public User createUser(@Valid @RequestBody UserRequest request) {
    return userService.create(request);
}
```

```java
// ExceptionHandler trong GlobalExceptionHandler (đã có ở trên)
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(...) { ... }
```

**Response mẫu khi validation fail:**
```json
{
  "status": 400,
  "message": "Username is required",
  "errorCode": "VALIDATION_FAILED",
  "path": "/api/users"
}
```

---

## 6. Xử lý các lỗi phổ biến khác

### 6.1. HttpMessageNotReadableException (Request Body sai format)
```java
@ExceptionHandler(HttpMessageNotReadableException.class)
public ResponseEntity<ErrorResponse> handleMalformedJson(HttpServletRequest request) {
    ErrorResponse error = new ErrorResponse(
        400,
        "Malformed JSON request. Please check your request body.",
        "MALFORMED_JSON",
        request.getRequestURI()
    );
    return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
}
```

### 6.2. MissingServletRequestParameterException (Thiếu param bắt buộc)
```java
@ExceptionHandler(MissingServletRequestParameterException.class)
public ResponseEntity<ErrorResponse> handleMissingParam(
        MissingServletRequestParameterException e,
        HttpServletRequest request) {
    ErrorResponse error = new ErrorResponse(
        400,
        String.format("Parameter '%s' is required", e.getParameterName()),
        "MISSING_PARAM",
        request.getRequestURI()
    );
    return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
}
```

### 6.3. AccessDeniedException (403 từ Spring Security)
```java
@ExceptionHandler(AccessDeniedException.class)
public ResponseEntity<ErrorResponse> handleAccessDenied(HttpServletRequest request) {
    ErrorResponse error = new ErrorResponse(
        403,
        "You do not have permission to access this resource.",
        "ACCESS_DENIED",
        request.getRequestURI()
    );
    return new ResponseEntity<>(error, HttpStatus.FORBIDDEN);
}
```

---

## 7. Best Practices & Lời khuyên cho Production

1.  **Luôn sử dụng `@RestControllerAdvice`** – Tập trung mọi xử lý lỗi về một chỗ.
2.  **Định nghĩa ErrorResponse thống nhất** – Đảm bảo client luôn parse được response.
3.  **Phân cấp Custom Exception** – Không dùng chung một `AppException` với status code linh hoạt quá mức. Tách thành `ResourceNotFoundException`, `BadRequestException`... để code rõ nghĩa hơn.
4.  **Sử dụng HTTP Status Code chuẩn xác:**
    - `400` cho lỗi validation, sai input.
    - `401` cho lỗi xác thực.
    - `403` cho lỗi phân quyền.
    - `404` cho resource không tồn tại.
    - `409` cho lỗi xung đột (ví dụ: username đã tồn tại).
    - `422` cho lỗi nghiệp vụ (thực thể không thể xử lý).
    - `500` cho lỗi server không mong đợi.
5.  **KHÔNG trả về stack trace cho client** – Ghi log chi tiết ở server bằng `log.error()`.
6.  **Cân nhắc sử dụng Error Code riêng** – Ngoài Status Code, một mã lỗi dạng `USER_NOT_FOUND`, `VALIDATION_FAILED` giúp client xử lý lỗi đa ngôn ngữ hoặc điều hướng logic dễ hơn.
7.  **Luôn bắt `Exception.class` làm Fallback cuối cùng** – Tránh để lộ lỗi server ra client và đảm bảo response vẫn là JSON.

---

## 8. Luồng xử lý lỗi hoàn chỉnh trong Spring Boot

```text
Client Request 
    → DispatcherServlet 
        → Controller 
            → Service (ném AppException)
        → ExceptionHandler (@RestControllerAdvice bắt AppException)
    → ErrorResponse (JSON) gửi về Client
```

---

Nếu bạn muốn nâng cấp lên level cao hơn nữa với các chủ đề như **Error Code System chuẩn Big Tech**, **Centralized Error Catalog**, hoặc **Logging + Distributed Tracing** trong môi trường Microservices, hãy cho tôi biết để tôi soạn tiếp nhé.