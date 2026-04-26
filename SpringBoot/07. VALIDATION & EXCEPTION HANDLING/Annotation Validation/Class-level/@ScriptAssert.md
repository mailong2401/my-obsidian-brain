### @ScriptAssert (Class-level)
Kiểm tra logic phức tạp liên quan đến nhiều trường.

```java
@ScriptAssert(lang = "javascript", 
              script = "_this.password.equals(_this.confirmPassword)",
              message = "Mật khẩu xác nhận không khớp")
public class RegisterRequest {
    private String password;
    private String confirmPassword;
}