# Session-per-device + Opaque Refresh Token + Rotation + Reuse Detection

> [!abstract] Mục tiêu  
> Xây dựng hệ thống authentication theo mô hình:
> 
> **Session-per-device + Opaque Refresh Token + Refresh Token Rotation + Reuse Detection**
> 
> Hiểu được:
> 
> - Access Token và Refresh Token
>     
> - Session-per-device
>     
> - Opaque Refresh Token
>     
> - Hash refresh token trong database
>     
> - Refresh Token Rotation
>     
> - Token Family
>     
> - Reuse Detection
>     
> - Race condition
>     
> - Logout từng thiết bị / logout all
>     
> - Attacker đánh cắp refresh token thì chuyện gì xảy ra?
>     

---

# 1. Bài toán Authentication

Giả sử user đăng nhập:

```text
POST /auth/login

email
password
```

Server xác thực thành công và trả:

```text
Access Token
Refresh Token
```

Ví dụ:

```text
Access Token
expires: 15 phút

Refresh Token
expires: 30 ngày
```

Access Token được dùng:

```text
Client
   │
   │ Authorization: Bearer ACCESS_TOKEN
   ▼
API
```

Ví dụ:

```http
GET /users/me
Authorization: Bearer eyJ...
```

---

# 2. Tại sao cần Refresh Token?

Nếu chỉ có Access Token sống 15 phút:

```text
Login
  │
  ▼
Access Token
  │
  ├──── 5 phút
  ├──── 10 phút
  └──── 15 phút
          │
          ▼
        expired
```

User sẽ phải login lại mỗi 15 phút.

Không thực tế.

Do đó có Refresh Token.

```text
Access Token expired
       │
       ▼
Refresh Token
       │
       ▼
POST /auth/refresh
       │
       ▼
New Access Token
```

User không cần nhập password lại.

---

# 3. Hai loại token có nhiệm vụ khác nhau

Có thể hình dung:

```text
Access Token
     │
     ├── dùng thường xuyên
     ├── sống ngắn
     └── truy cập API


Refresh Token
     │
     ├── dùng ít
     ├── sống dài
     └── xin Access Token mới
```

Một thiết kế phổ biến:

```text
Access Token: 5–15 phút

Refresh Token: vài ngày / vài tuần
```

Thời gian cụ thể phụ thuộc security requirement của hệ thống.

---

# 4. Session-per-device là gì?

Đây là phần cực kỳ quan trọng.

Giả sử user Long đăng nhập trên:

```text
Laptop
iPhone
iPad
```

Không nên coi cả 3 là cùng một session.

Thay vào đó:

```text
User: Long
│
├── Session A
│      device: Laptop
│
├── Session B
│      device: iPhone
│
└── Session C
       device: iPad
```

Mỗi lần login tạo một **session riêng**.

Đây chính là:

# Session-per-device

Tên gọi "per-device" mang tính mô hình UX. Thực tế một browser/profile/app installation có thể tạo nhiều session tùy chính sách login của hệ thống.

---

# 5. Database Session

Ví dụ bảng:

```sql
CREATE TABLE sessions (
    id UUID PRIMARY KEY,

    user_id UUID NOT NULL,

    refresh_token_hash TEXT NOT NULL,

    family_id UUID NOT NULL,

    expires_at TIMESTAMP NOT NULL,

    revoked_at TIMESTAMP NULL,

    created_at TIMESTAMP NOT NULL,

    last_used_at TIMESTAMP NULL,

    device_name TEXT NULL,

    ip_address TEXT NULL,

    user_agent TEXT NULL
);
```

Ví dụ:

```text
sessions

┌────────┬─────────┬────────────┬────────────┐
│ id     │ user    │ device     │ revoked    │
├────────┼─────────┼────────────┼────────────┤
│ S1     │ Long    │ Laptop     │ false      │
│ S2     │ Long    │ iPhone     │ false      │
│ S3     │ Long    │ iPad       │ false      │
└────────┴─────────┴────────────┴────────────┘
```

Nhờ vậy:

```text
Logout Laptop
```

chỉ revoke:

```text
S1
```

iPhone và iPad vẫn hoạt động.

---

# 6. Tại sao không chỉ lưu refresh token theo User?

Thiết kế đơn giản:

```text
users

id
email
password_hash
refresh_token
```

Có vấn đề khi user có nhiều thiết bị.

Ví dụ:

```text
Laptop login
   ↓
refresh_token = R1
```

Sau đó:

```text
iPhone login
   ↓
refresh_token = R2
```

Nếu chỉ có một field:

```text
users.refresh_token
```

R2 sẽ thay R1.

Laptop có thể mất session.

Hoặc nếu dùng chung một token cho mọi thiết bị:

```text
Laptop ─┐
iPhone ─┼── R1
iPad ───┘
```

thì khó:

- logout riêng thiết bị
    
- revoke riêng thiết bị
    
- quản lý thiết bị
    
- phát hiện session bị compromise
    

Do đó:

```text
User
  │
  ├── Session
  ├── Session
  └── Session
```

thường linh hoạt hơn.

---

# 7. Opaque Refresh Token là gì?

Có hai cách phổ biến để thiết kế Refresh Token.

## Cách 1 — JWT Refresh Token

Refresh Token chứa claims:

```json
{
  "sub": "user-123",
  "sessionId": "session-456",
  "exp": 1790000000
}
```

Token có dạng:

```text
eyJhbGciOi...
```

Server verify signature.

---

## Cách 2 — Opaque Refresh Token

Token chỉ là một chuỗi random.

Ví dụ:

```text
B3m2Y0VxL9...random...8KpA
```

Nó không chứa thông tin business có ý nghĩa đối với client.

Client không cần biết:

```text
userId
email
role
sessionId
```

từ token đó.

Nó chỉ giống như một secret credential.

```text
Random Secret
     │
     ▼
Server lookup / verify
     │
     ▼
Session
     │
     ▼
User
```

Đó là **Opaque Token**.

---

# 8. Opaque token nên random mạnh

Ví dụ Node.js:

```ts
import { randomBytes } from 'crypto';

const refreshToken = randomBytes(32).toString('base64url');
```

32 bytes:

```text
256 bits randomness
```

Attacker gần như không thể brute-force nếu RNG được dùng đúng.

---

# 9. Không nên lưu raw Refresh Token

Giả sử server phát:

```text
R1 = 8Jd92ks...
```

Không nên database:

```text
refresh_token = "8Jd92ks..."
```

Nếu database bị leak:

```text
Attacker
   │
   ▼
database dump
   │
   ▼
raw refresh token
   │
   ▼
POST /auth/refresh
```

Attacker có credential sử dụng được ngay.

---

# 10. Hash Refresh Token

Thay vì lưu:

```text
R1
```

server lưu:

```text
hash(R1)
```

Ví dụ:

```ts
import { createHash } from 'crypto';

function hashToken(token: string): string {
  return createHash('sha256')
    .update(token)
    .digest('hex');
}
```

Khi tạo token:

```ts
const refreshToken = randomBytes(32).toString('base64url');

const tokenHash = hashToken(refreshToken);
```

Database:

```text
refresh_token_hash = SHA256(R1)
```

Client:

```text
R1
```

Server không cần lưu raw `R1`.

> [!note]  
> Với token ngẫu nhiên có entropy cao, SHA-256 thường phù hợp để tạo fingerprint/hash tra cứu. Password thì khác: password có entropy thấp nên cần password hashing như Argon2id/bcrypt/scrypt.

---

# 11. Login hoàn chỉnh

User:

```text
POST /auth/login
```

Server:

```text
1. Verify email/password

2. Create Session S1

3. Generate Refresh Token R1

4. hash(R1)

5. Store hash(R1) vào S1

6. Generate Access Token A1

7. Return:
      A1
      R1
```

Sơ đồ:

```text
User
 │
 │ email + password
 ▼
Server
 │
 ├── verify password
 │
 ├── create S1
 │
 ├── generate A1
 │
 └── generate R1
        │
        ├──────────────► Client receives R1
        │
        ▼
     hash(R1)
        │
        ▼
      Database
```

Database:

```text
Session S1

user_id = U1
refresh_token_hash = hash(R1)
```

---

# 12. Refresh Token Rotation

Đây là phần quan trọng nhất.

Thiết kế đơn giản có thể làm:

```text
R1
 │
 ├── refresh → A2
 ├── refresh → A3
 ├── refresh → A4
 └── refresh → A5
```

R1 dùng được nhiều lần cho đến khi expire.

Nếu attacker lấy được R1:

```text
User ─────── R1 ───────► Server

Attacker ─── R1 ───────► Server
```

Cả hai đều có thể refresh.

Không tốt.

---

# 13. Rotation giải quyết thế nào?

Mỗi lần Refresh Token được sử dụng thành công:

```text
R1
 │
 │ refresh
 ▼
A2 + R2
```

R1 lập tức bị thay thế.

Lần tiếp theo:

```text
R2
 │
 ▼
A3 + R3
```

Tiếp:

```text
R3
 │
 ▼
A4 + R4
```

Tạo thành:

```text
R1 → R2 → R3 → R4 → R5
```

Đây gọi là:

# Refresh Token Rotation

---

# 14. Refresh request thực tế

Client đang có:

```text
Access Token = A1
Refresh Token = R1
```

A1 hết hạn.

Client gửi:

```http
POST /auth/refresh
Cookie: refresh_token=R1
```

Server:

```text
hash(R1)
   │
   ▼
find Session
   │
   ▼
valid?
   │
   ├── NO → reject
   │
   └── YES
        │
        ├── create A2
        │
        ├── create R2
        │
        ├── invalidate R1
        │
        └── store hash(R2)
```

Response:

```text
Access Token = A2
Refresh Token = R2
```

Client phải thay:

```text
R1 ❌

R2 ✅
```

---

# 15. Điều cực kỳ quan trọng

Khi:

```text
R1 → refresh
```

server **không chỉ tạo A2**.

Nếu dùng Rotation thì server tạo:

```text
A2 + R2
```

Sau đó:

```text
R1 invalid
R2 valid
```

Đây chính là điểm rất hay bị nhầm.

---

# 16. Nhưng Rotation vẫn còn một vấn đề

Giả sử attacker đánh cắp:

```text
R1
```

User refresh trước:

```text
User:

R1
 ↓
A2 + R2
```

Bây giờ:

```text
R1 = old token
R2 = current token
```

Attacker sau đó thử:

```text
R1
 ↓
POST /auth/refresh
```

Server thấy:

```text
R1 đã được sử dụng rồi.
```

Đây là tín hiệu rất đáng ngờ.

Vì Refresh Token Rotation quy định:

```text
Một refresh token chỉ được sử dụng thành công một lần.
```

---

# 17. Reuse Detection

Server phát hiện một refresh token cũ đã được sử dụng lại.

Đây gọi là:

# Refresh Token Reuse Detection

Ví dụ:

```text
R1
 │
 │ legitimate use
 ▼
R2

R1 marked USED
```

Sau đó:

```text
Attacker
   │
   │ R1
   ▼
Server
   │
   ▼
R1 already USED
   │
   ▼
REUSE DETECTED
```

Server có bằng chứng rằng cùng credential đã xuất hiện nhiều hơn một lần.

---

# 18. Tại sao reuse cho thấy có vấn đề?

Trong flow bình thường:

```text
R1 → R2
```

Client phải bỏ R1.

Sau đó chỉ sử dụng:

```text
R2
```

Nếu server lại nhận:

```text
R1
```

có thể xảy ra:

```text
1. Token bị đánh cắp

2. Client gửi request duplicate

3. Race condition

4. Bug phía client

5. Response refresh trước đó bị mất
```

Vì vậy không nên đơn giản kết luận:

```text
reuse = hacker 100%
```

Nhưng nó là security signal quan trọng.

---

# 19. Token Family

Để quản lý chuỗi:

```text
R1 → R2 → R3 → R4
```

ta có khái niệm:

# Token Family

Ví dụ:

```text
family_id = F100
```

Các token:

```text
R1 ─┐
R2 ─┤
R3 ─┼── family F100
R4 ─┤
R5 ─┘
```

Family thường gắn với một session/login chain.

---

# 20. Session và Token Family

Có thể thiết kế:

```text
User U1
│
├── Session S1
│      Laptop
│
│      Family F1
│
│      R1 → R2 → R3
│
├── Session S2
│      iPhone
│
│      Family F2
│
│      R1' → R2' → R3'
│
└── Session S3
       iPad

       Family F3

       R1'' → R2''
```

Laptop bị compromise:

```text
revoke S1 / F1
```

không nhất thiết phải logout:

```text
S2
S3
```

---

# 21. Attacker lấy R1 — Case 1

Giả sử:

```text
User has R1
Attacker steals R1
```

Attacker sử dụng trước.

```text
Attacker

R1
 │
 ▼
Server
 │
 ▼
A2 + R2
```

Server nghĩ đây là request hợp lệ vì R1 chưa từng được dùng.

Lúc này attacker có:

```text
A2
R2
```

User vẫn giữ:

```text
R1
```

Sau đó user refresh:

```text
User

R1
 │
 ▼
Server
```

Server thấy:

```text
R1 already used
```

=> REUSE DETECTED.

---

# 22. Lúc đó server làm gì?

Một chính sách bảo mật phổ biến:

```text
REUSE DETECTED
       │
       ▼
revoke session/token family
       │
       ▼
R2 invalid
R3 invalid
...
```

Tức là:

```text
Attacker:
R2 ❌

User:
R1 ❌
```

Cả hai mất refresh capability của session đó.

User phải authentication lại.

Đây là một trade-off:

```text
Security > convenience
```

---

# 23. Attacker lấy R1 — Case 2

User sử dụng R1 trước.

```text
User

R1
 ↓
A2 + R2
```

R1 đã used.

Attacker:

```text
R1
 ↓
Server
 ↓
REUSE DETECTED
 ↓
revoke session
```

Kết quả:

```text
R1 ❌
R2 ❌
```

Attacker không thể tiếp tục bằng refresh-token family đó.

User cũng cần login lại.

---

# 24. Nhưng Access Token thì sao?

Điểm rất quan trọng.

Giả sử attacker đã nhận:

```text
A2
```

Sau đó session bị revoke.

Nếu Access Token là JWT stateless:

```text
Server normally không query session mỗi request.
```

Thì:

```text
A2
```

có thể vẫn hoạt động cho đến khi:

```text
exp
```

Ví dụ:

```text
Access Token lifetime = 10 phút
```

Attacker có thể còn tối đa khoảng 10 phút truy cập tùy thời điểm token được cấp.

Đó là lý do Access Token nên có lifetime ngắn.

---

# 25. Muốn revoke Access Token ngay lập tức thì sao?

Có thể kiểm tra session mỗi request.

Access Token chứa:

```json
{
  "sub": "U1",
  "sid": "S1"
}
```

API:

```text
Access Token
    │
    ▼
verify JWT
    │
    ▼
sid = S1
    │
    ▼
check Session S1
    │
    ├── active → allow
    │
    └── revoked → deny
```

Nhưng đổi lại:

```text
request
   ↓
JWT verify
   ↓
Redis/DB lookup
```

Tốn thêm I/O.

Có thể dùng Redis:

```text
sid → active/revoked
```

để giảm cost.

Đây là trade-off giữa:

```text
Stateless performance

vs

Immediate revocation
```

---

# 26. Thiết kế Database cho Rotation

Có hai hướng chính.

## Hướng A — Chỉ lưu current token

Session:

```text
sessions

id
user_id
current_refresh_token_hash
expires_at
revoked_at
```

Rotation:

```text
R1
 ↓
verify hash
 ↓
replace hash(R1)
with hash(R2)
```

Database chỉ biết:

```text
current = R2
```

Thiết kế đơn giản nhưng reuse detection/history hạn chế hơn.

---

# 27. Hướng B — Lưu từng Refresh Token

Ví dụ:

```sql
refresh_tokens
--------------
id
session_id
family_id
token_hash
parent_token_id
created_at
expires_at
used_at
revoked_at
replaced_by_token_id
```

Sau một vài lần refresh:

```text
R1
used_at = 10:00
replaced_by = R2

R2
used_at = 10:15
replaced_by = R3

R3
used_at = NULL
```

Ta nhìn thấy:

```text
R1 → R2 → R3
```

R3 là token hiện tại.

---

# 28. Ví dụ database

```text
refresh_tokens

┌────┬─────────┬────────┬──────────┬─────────────┐
│ id │ session │ family │ used_at  │ replaced_by │
├────┼─────────┼────────┼──────────┼─────────────┤
│ R1 │ S1      │ F1     │ 10:00    │ R2          │
│ R2 │ S1      │ F1     │ 10:15    │ R3          │
│ R3 │ S1      │ F1     │ NULL     │ NULL        │
└────┴─────────┴────────┴──────────┴─────────────┘
```

Current valid token:

```text
R3
```

Nếu request gửi:

```text
R1
```

lookup thấy:

```text
used_at != NULL
```

=> reuse.

---

# 29. Hash lookup

Client gửi:

```text
R1
```

Server:

```ts
const hash = hashToken(refreshToken);
```

Sau đó:

```sql
SELECT *
FROM refresh_tokens
WHERE token_hash = ?;
```

Server không cần biết raw token cũ.

---

# 30. Một cách tối ưu token format

Opaque token có thể gồm:

```text
tokenId.secret
```

Ví dụ:

```text
a8f12c.X2Km9Q...random...
```

Trong đó:

```text
a8f12c
   │
   └── public lookup ID

X2Km9Q...
   │
   └── secret
```

Server:

```text
tokenId
   ↓
lookup database nhanh
   ↓
get token record
   ↓
hash(secret)
   ↓
compare stored hash
```

Tương tự tư tưởng:

```text
identifier + secret
```

Public identifier không phải credential; secret mới là phần cần bảo vệ.

---

# 31. Race Condition nguy hiểm

Giả sử browser vô tình gửi cùng lúc:

```text
Request A → R1
Request B → R1
```

Nếu server xử lý không atomic:

```text
Request A:
R1 valid

Request B:
R1 valid
```

Sau đó:

```text
A → R2
B → R3
```

Bây giờ có hai nhánh:

```text
       ┌── R2
R1 ────┤
       └── R3
```

Rotation bị phá vỡ.

---

# 32. Phải đảm bảo atomic

Logic cần giống:

```text
BEGIN TRANSACTION

lock R1

if R1.used_at != null
    reuse detected

mark R1 used

create R2

COMMIT
```

Ví dụ PostgreSQL:

```sql
SELECT *
FROM refresh_tokens
WHERE token_hash = $1
FOR UPDATE;
```

Request A lock row.

Request B phải chờ.

Sau khi A:

```text
R1.used_at = now()
```

commit.

B đọc lại:

```text
used_at != NULL
```

và không được rotate R1 lần thứ hai.

---

# 33. Nhưng duplicate request có thể gây false positive

Ví dụ network:

```text
Client
 │
 ├──── R1 ─────► Server
 │
 │              R1 → R2
 │
 │
 │   response bị mất
 │       X
 │
 └──── retry R1 ─────► Server
```

Server nhìn thấy:

```text
R1 reused
```

nhưng không có hacker.

Chỉ là network retry.

Vì vậy production implementation phải suy nghĩ kỹ về:

- concurrent refresh
    
- retry
    
- multiple browser tabs
    
- mobile networking
    
- response loss
    

Có hệ thống chọn strict revoke ngay; có hệ thống sử dụng grace/idempotency strategy rất ngắn để xử lý duplicate hợp lệ.

Không nên thêm grace period tùy tiện vì nó có thể làm yếu reuse detection.

---

# 34. Multiple tabs

Browser:

```text
Tab A
Tab B
Tab C
```

Cả ba thấy Access Token expired.

```text
Tab A → refresh R1
Tab B → refresh R1
Tab C → refresh R1
```

Có thể gây reuse.

Frontend nên có cơ chế:

```text
             ┌── API 1
             │
expired ─────┼── API 2
             │
             └── API 3
                  │
                  ▼
           ONE refresh request
                  │
                  ▼
                R2
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
        retry   retry   retry
```

Thường gọi là:

```text
single-flight refresh
```

hoặc refresh mutex/lock phía client.

---

# 35. Session-per-device + Rotation

Bây giờ ghép lại.

Login Laptop:

```text
Session S1
Family F1

R1 → R2 → R3
```

Login iPhone:

```text
Session S2
Family F2

X1 → X2
```

Database:

```text
User Long
│
├── S1 Laptop
│      │
│      └── F1
│           R1 → R2 → R3
│
└── S2 iPhone
       │
       └── F2
            X1 → X2
```

---

# 36. Hacker tấn công Laptop

Attacker lấy:

```text
R2
```

Laptop refresh:

```text
R2 → R3
```

Attacker dùng R2:

```text
R2
 ↓
reuse detected
 ↓
revoke S1/F1
```

Kết quả:

```text
Laptop S1 ❌
```

Nhưng:

```text
iPhone S2 ✅
```

Đây là lợi ích rất lớn của session-per-device.

---

# 37. Logout một thiết bị

User chọn:

```text
Logout Laptop
```

Backend:

```sql
UPDATE sessions
SET revoked_at = NOW()
WHERE id = 'S1';
```

Sau đó toàn bộ refresh-token chain thuộc S1 không được sử dụng nữa.

```text
S1 ❌ Laptop

S2 ✅ iPhone

S3 ✅ iPad
```

---

# 38. Logout All Devices

```sql
UPDATE sessions
SET revoked_at = NOW()
WHERE user_id = 'U1';
```

Kết quả:

```text
User U1
│
├── Laptop ❌
├── iPhone ❌
└── iPad ❌
```

Refresh token của tất cả session không còn hợp lệ.

---

# 39. Change Password

Tùy security policy.

Ví dụ sau khi đổi password:

```text
revoke all sessions
```

hoặc:

```text
revoke all sessions except current session
```

Một số hệ thống yêu cầu reauthentication cho hành động nhạy cảm thay vì phụ thuộc hoàn toàn vào tuổi session.

---

# 40. Access Token chứa gì?

Ví dụ:

```json
{
  "sub": "user-id",
  "sid": "session-id",
  "role": "USER",
  "iat": 1780000000,
  "exp": 1780000900
}
```

Quan trọng:

```text
sub
```

= subject/user.

```text
sid
```

= session.

Nhờ `sid`, backend có thể liên hệ Access Token với session nếu kiến trúc cần.

---

# 41. Refresh Token chứa gì?

Nếu dùng opaque:

```text
nothing meaningful
```

Chỉ là:

```text
random secret
```

Ví dụ:

```text
Vf87NnJ2lX....
```

Server dùng token để tìm/xác minh server-side state.

Đây là điểm khác biệt quan trọng với JWT.

---

# 42. Cookie cho Refresh Token

Web application thường có thể đặt Refresh Token trong cookie:

```http
Set-Cookie:
refresh_token=R1;
HttpOnly;
Secure;
SameSite=Lax;
Path=/auth
```

### HttpOnly

JavaScript phía browser không đọc cookie trực tiếp:

```js
document.cookie
```

không truy cập được cookie `HttpOnly`.

Điều này giúp giảm khả năng token bị lấy trực tiếp qua XSS.

Nhưng:

> [!warning]  
> HttpOnly không làm XSS trở nên vô hại. Script độc hại vẫn có thể thực hiện request dưới context của người dùng tùy kiến trúc.

---

### Secure

Cookie chỉ được gửi qua HTTPS.

---

### SameSite

Giúp kiểm soát cross-site cookie behavior và là một phần của phòng chống CSRF.

Nếu kiến trúc dùng cookie authentication, cần thiết kế CSRF protection phù hợp với deployment thực tế.

---

# 43. Access Token lưu ở đâu?

Một mô hình web có thể là:

```text
Refresh Token
    ↓
HttpOnly Secure Cookie

Access Token
    ↓
memory
```

Thay vì cố ý lưu credential dài hạn vào:

```text
localStorage
```

Lý do chính là JavaScript chạy trong origin có thể đọc `localStorage`; nếu có XSS thì credential lưu ở đó có nguy cơ bị exfiltrate.

Tuy nhiên lựa chọn storage phụ thuộc kiến trúc frontend/backend và threat model.

---

# 44. Toàn bộ Login Flow

```text
CLIENT
  │
  │ email/password
  ▼
POST /auth/login
  │
  ▼
SERVER
  │
  ├── verify password
  │
  ├── create Session S1
  │
  ├── create Family F1
  │
  ├── generate A1
  │
  └── generate R1
          │
          ├── hash(R1) → DB
          │
          └── R1 → HttpOnly cookie
```

Client nhận:

```text
A1
R1(cookie)
```

---

# 45. Toàn bộ Refresh Flow

```text
CLIENT

A1 expired
    │
    ▼
POST /auth/refresh
Cookie: R1
    │
    ▼
SERVER
    │
    ├── hash(R1)
    │
    ├── lookup token
    │
    ├── check session
    │
    ├── check expiration
    │
    ├── check revoked
    │
    └── check used
            │
            ▼
         VALID
            │
            ├── R1.used = true
            │
            ├── generate R2
            │
            ├── store hash(R2)
            │
            └── generate A2
```

Response:

```text
A2
Set-Cookie: R2
```

Client:

```text
A1 ❌
R1 ❌

A2 ✅
R2 ✅
```

---

# 46. Reuse Attack Flow

```text
              R1 stolen
                  │
          ┌───────┴────────┐
          │                │
        USER            ATTACKER
          │                │
          │ R1             │ R1
          ▼                │
        SERVER             │
          │                │
          ▼                │
       R1 → R2             │
                           │
                           ▼
                         SERVER
                           │
                           ▼
                   R1 already used
                           │
                           ▼
                    REUSE DETECTED
                           │
                           ▼
                    revoke Session
```

Kết quả:

```text
R1 ❌
R2 ❌
Session ❌
```

Attacker mất khả năng duy trì quyền truy cập lâu dài bằng refresh-token chain đó.

---

# 47. Vì sao Rotation mạnh?

Không Rotation:

```text
stolen R1
   │
   ├── attacker refresh
   ├── attacker refresh
   ├── attacker refresh
   ├── attacker refresh
   └── ...
```

cho đến khi expire/revoke.

Rotation:

```text
R1 → R2 → R3
```

Mỗi token về nguyên tắc chỉ được sử dụng thành công một lần.

Copy cũ xuất hiện lại:

```text
reuse signal
```

---

# 48. Nhưng Rotation không phải phép màu

Rotation không cứu được mọi trường hợp.

Nếu attacker có khả năng liên tục lấy **current token**:

```text
R1 stolen
R2 stolen
R3 stolen
R4 stolen
```

thì vấn đề nằm sâu hơn:

```text
device compromise
malware
XSS
browser compromise
server compromise
```

Rotation chủ yếu giúp hạn chế và phát hiện **replay của refresh token đã bị copy**.

---

# 49. Kiến trúc tổng thể

```text
                    ┌──────────────────┐
                    │       USER       │
                    └────────┬─────────┘
                             │
                           Login
                             │
                             ▼
                    ┌──────────────────┐
                    │   Auth Server    │
                    └────────┬─────────┘
                             │
                 ┌───────────┴───────────┐
                 │                       │
                A1                      R1
          Access Token           Refresh Token
                 │                       │
                 │                   hash(R1)
                 │                       │
                 ▼                       ▼
             Client                 Database
                                         │
                                         ▼
                                   Session S1
                                         │
                                         ▼
                                   Token Family
                                         │
                                         ▼
                              R1 → R2 → R3 → R4
```

---

# 50. NestJS Architecture

Có thể tổ chức:

```text
auth/
├── auth.controller.ts
├── auth.service.ts
│
├── guards/
│   └── jwt-auth.guard.ts
│
├── strategies/
│   └── jwt.strategy.ts
│
├── entities/
│   ├── session.entity.ts
│   └── refresh-token.entity.ts
│
└── dto/
    ├── login.dto.ts
    └── register.dto.ts
```

Service:

```text
AuthService
│
├── login()
├── refresh()
├── logout()
├── logoutAll()
├── revokeSession()
├── createSession()
├── rotateRefreshToken()
└── detectReuse()
```

---

# 51. Pseudo-code Login

```ts
async login(user: User, device: DeviceInfo) {
  const session = await this.createSession(
    user.id,
    device,
  );

  const refreshToken = generateSecureToken();

  await this.saveRefreshToken({
    sessionId: session.id,
    tokenHash: hashToken(refreshToken),
  });

  const accessToken = await this.generateAccessToken({
    userId: user.id,
    sessionId: session.id,
  });

  return {
    accessToken,
    refreshToken,
  };
}
```

---

# 52. Pseudo-code Rotation

Ý tưởng:

```ts
async refresh(rawToken: string) {
  const tokenHash = hashToken(rawToken);

  return this.dataSource.transaction(async (manager) => {
    const token = await manager.findOne(RefreshToken, {
      where: {
        tokenHash,
      },

      lock: {
        mode: "pessimistic_write",
      },
    });

    if (!token) {
      throw new UnauthorizedException();
    }

    if (token.revokedAt) {
      throw new UnauthorizedException();
    }

    if (token.usedAt) {
      await this.handleReuse(token, manager);

      throw new UnauthorizedException();
    }

    const session = await this.getSession(
      token.sessionId,
      manager,
    );

    if (session.revokedAt) {
      throw new UnauthorizedException();
    }

    if (token.expiresAt < new Date()) {
      throw new UnauthorizedException();
    }

    token.usedAt = new Date();

    const newRefreshToken = generateSecureToken();

    const newToken = manager.create(RefreshToken, {
      sessionId: session.id,
      familyId: token.familyId,
      tokenHash: hashToken(newRefreshToken),
      parentTokenId: token.id,
    });

    await manager.save(newToken);

    token.replacedByTokenId = newToken.id;

    await manager.save(token);

    const accessToken = await this.generateAccessToken({
      userId: session.userId,
      sessionId: session.id,
    });

    return {
      accessToken,
      refreshToken: newRefreshToken,
    };
  });
}
```

Đây chỉ là kiến trúc minh họa; production implementation còn phải xử lý transaction boundaries, unique constraints, expiry, cookie policy và concurrency thật cẩn thận.

---

# 53. Security Model cuối cùng

Hệ thống hoàn chỉnh:

```text
                     USER
                      │
                      ▼
                    LOGIN
                      │
                      ▼
                  Session S1
                      │
                      ▼
              ┌───────────────┐
              │               │
              ▼               ▼
        Access Token A1    Refresh R1
          short-lived        opaque
                              │
                              ▼
                           hash(R1)
                              │
                              ▼
                              DB

Refresh:

R1
 │
 ▼
validate
 │
 ▼
mark R1 used
 │
 ├───────────────┐
 ▼               ▼
A2              R2
                 │
                 ▼
              hash(R2)

Next:

R2 → A3 + R3

Next:

R3 → A4 + R4
```

Nếu token cũ xuất hiện:

```text
R1
 │
 ▼
already used
 │
 ▼
REUSE DETECTED
 │
 ▼
REVOKE SESSION / FAMILY
 │
 ├── R1 ❌
 ├── R2 ❌
 ├── R3 ❌
 └── future refresh ❌
```

---

# 54. Mental Model cần nhớ

Đừng nghĩ Refresh Token là:

```text
"Token sống 30 ngày để lấy access token."
```

Hãy nghĩ:

```text
Refresh Token
      │
      ▼
one-time credential
      │
      ▼
R1 → R2 → R3 → R4
```

Session mới là đối tượng quản lý dài hạn:

```text
User
 │
 ├── Session Laptop
 │      └── token family
 │
 ├── Session Phone
 │      └── token family
 │
 └── Session Tablet
        └── token family
```

---

# 55. Công thức tổng kết

```text
Session-per-device
+
Opaque Refresh Token
+
Secure random generation
+
Hash token at rest
+
Short-lived Access Token
+
Refresh Token Rotation
+
Token Family
+
Reuse Detection
+
Atomic Rotation
+
Session Revocation
+
Secure/HttpOnly cookie
=
Robust session architecture
```

> [!important] Câu quan trọng nhất  
> Khi dùng **Refresh Token Rotation**:
> 
> ```text
> R1 --refresh--> A2 + R2
> ```
> 
> chứ không phải:
> 
> ```text
> R1 --refresh--> A2
> ```
> 
> Sau đó:
> 
> ```text
> R1 = USED
> R2 = CURRENT
> ```
> 
> Nếu `R1` xuất hiện lần nữa:
> 
> ```text
> REUSE DETECTED
> ```
> 
> và hệ thống có thể revoke toàn bộ session/token family đó.

---

## 🔗 Liên kết Obsidian

- [[Authentication]]
    
- [[Authorization]]
    
- [[JWT]]
    
- [[Access Token]]
    
- [[Refresh Token]]
    
- [[Opaque Token]]
    
- [[Session Management]]
    
- [[Refresh Token Rotation]]
    
- [[Token Reuse Detection]]
    
- [[OAuth 2.0]]
    
- [[NestJS Authentication]]
    
- [[NestJS Guards]]
    
- [[Redis]]
    
- [[PostgreSQL Transactions]]
    
- [[Database Race Condition]]
    
- [[CSRF]]
    
- [[XSS]]