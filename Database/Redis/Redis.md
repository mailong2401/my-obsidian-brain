# Redis — Từ cơ bản đến ứng dụng Backend NestJS

## 1. Redis là gì?

**Redis** = **Remote Dictionary Server**.

Redis là một hệ thống lưu trữ dữ liệu dạng **key-value**, nổi bật vì phần lớn dữ liệu được thao tác trong **RAM**, nên tốc độ đọc/ghi rất cao.

Có thể hình dung:

```text
Redis
│
├── "user:1"        → {...}
├── "session:S1"    → {...}
├── "otp:user@email"→ "493821"
└── "product:100"   → {...}
```

Khác với PostgreSQL:

```text
PostgreSQL

users
┌────┬──────────┬─────────────────┐
│ id │ name     │ email           │
├────┼──────────┼─────────────────┤
│ 1  │ Long     │ long@gmail.com  │
│ 2  │ An       │ an@gmail.com    │
└────┴──────────┴─────────────────┘
```

Redis thường làm việc theo:

```text
KEY → VALUE
```

Ví dụ:

```text
"user:1:name" → "Long"
```

---

# 2. Tại sao Redis nhanh?

Redis thường giữ working dataset trong RAM.

So sánh đơn giản:

```text
Application
    │
    ├── PostgreSQL
    │       ↓
    │   database engine
    │       ↓
    │   storage / cache
    │
    └── Redis
            ↓
           RAM
```

RAM có độ trễ rất thấp, nhưng Redis nhanh không chỉ vì RAM.

Redis còn được thiết kế với:

- data structure tối ưu
    
- event-driven architecture
    
- command execution đơn giản
    
- networking hiệu quả
    
- tránh nhiều overhead của relational database
    

Redis rất phù hợp với:

```text
Cache
Session
Refresh Token
OTP
Rate Limiting
Distributed Lock
Queue
Pub/Sub
Leaderboard
Counter
Temporary Data
```

---

# 3. Redis không phải chỉ là Cache

Nhiều người mới học nghĩ:

```text
Redis = Cache
```

Không hoàn toàn đúng.

Redis là một **in-memory data store** hỗ trợ nhiều cấu trúc dữ liệu.

Cache chỉ là một trong những use case phổ biến nhất.

```text
                  Redis
                    │
      ┌─────────────┼──────────────┐
      │             │              │
    Cache        Session        Queue
      │             │              │
 Rate Limit       OTP          Pub/Sub
      │             │              │
 Counter         Token       Leaderboard
```

---

# 4. Redis hoạt động theo Key-Value

Ví dụ:

```bash
SET username Long
```

Redis:

```text
username
   ↓
"Long"
```

Lấy dữ liệu:

```bash
GET username
```

Kết quả:

```text
Long
```

Xóa:

```bash
DEL username
```

---

# 5. Redis Key

Key là tên dùng để tìm dữ liệu.

Ví dụ không nên đặt:

```text
abc
data
user
token
```

Vì khó quản lý.

Thường sử dụng convention:

```text
entity:id:property
```

Ví dụ:

```text
user:123
user:123:profile

session:123:device1

refresh:abc123

otp:email:long@gmail.com

product:100

cart:123
```

Nhìn key:

```text
session:user123:device456
```

là có thể hiểu:

```text
session
   │
   └── user123
           │
           └── device456
```

---

# 6. Namespace trong Redis

Redis không có folder thật như:

```text
session/
    user/
        device/
```

Dấu `:` chỉ là convention.

Ví dụ:

```text
session:user1:laptop
session:user1:iphone
session:user1:ipad
```

Redis vẫn coi chúng là ba string key độc lập.

Nhưng developer nhìn vào sẽ dễ quản lý hơn.

---

# 7. Các Data Type quan trọng

Redis không chỉ lưu string.

Các kiểu phổ biến:

```text
String
Hash
List
Set
Sorted Set
Stream
```

Ngoài ra Redis hiện đại còn có các khả năng/data structures khác tùy module và phiên bản.

---

# 8. String

Đây là kiểu cơ bản nhất.

```bash
SET name Long
```

```bash
GET name
```

```text
Long
```

String có thể dùng cho:

```text
text
number
JSON string
token
counter
flag
```

Ví dụ JSON:

```bash
SET user:1 '{"id":1,"name":"Long"}'
```

Lấy:

```bash
GET user:1
```

---

# 9. SET với TTL

Một tính năng cực kỳ quan trọng:

```bash
SET otp:user123 493821 EX 300
```

Nghĩa là:

```text
key:
otp:user123

value:
493821

TTL:
300 seconds
```

Sau 5 phút:

```text
otp:user123
    ↓
Redis tự động expire
    ↓
key biến mất
```

Rất phù hợp cho:

```text
OTP
Refresh Token metadata
Session
Cache
Password reset token
Email verification
Rate limit
```

---

# 10. TTL là gì?

TTL = **Time To Live**.

Nghĩa là thời gian key còn tồn tại.

Kiểm tra:

```bash
TTL otp:user123
```

Ví dụ:

```text
245
```

nghĩa là còn khoảng:

```text
245 seconds
```

Đặt expire:

```bash
EXPIRE otp:user123 300
```

---

# 11. SETEX

Có thể viết:

```bash
SETEX otp:user123 300 493821
```

Tương đương ý tưởng:

```text
SET value
+
EXPIRE 300 seconds
```

Tuy nhiên cú pháp `SET ... EX ...` thường linh hoạt hơn.

---

# 12. Hash

Redis Hash phù hợp khi một object có nhiều field.

Ví dụ user:

```text
user:123
│
├── name  → Long
├── email → long@gmail.com
└── age   → 20
```

Tạo:

```bash
HSET user:123 name Long email long@gmail.com age 20
```

Lấy một field:

```bash
HGET user:123 name
```

Kết quả:

```text
Long
```

Lấy toàn bộ:

```bash
HGETALL user:123
```

---

# 13. String JSON vs Redis Hash

Có hai cách lưu object.

### JSON String

```text
user:123
    ↓
"{\"name\":\"Long\",\"age\":20}"
```

Muốn đổi age:

```text
GET
↓
JSON.parse
↓
update
↓
JSON.stringify
↓
SET
```

Trong khi Hash:

```bash
HSET user:123 age 21
```

Có thể update trực tiếp field.

Redis Hash phù hợp khi:

```text
object có nhiều field
+
cần update từng field
```

---

# 14. List

Redis List là danh sách có thứ tự.

Ví dụ:

```text
tasks
│
├── task3
├── task2
└── task1
```

Thêm:

```bash
LPUSH tasks task1
```

hoặc:

```bash
RPUSH tasks task1
```

Lấy/xóa:

```bash
LPOP tasks
```

```bash
RPOP tasks
```

Có thể dùng cho một số queue đơn giản.

---

# 15. Set

Set chứa các phần tử **không trùng nhau**.

Ví dụ:

```text
online-users
│
├── user1
├── user2
└── user3
```

Thêm:

```bash
SADD online-users user1
```

Kiểm tra:

```bash
SISMEMBER online-users user1
```

Xóa:

```bash
SREM online-users user1
```

Lấy toàn bộ:

```bash
SMEMBERS online-users
```

---

# 16. Set rất hữu ích cho quản lý session

Ví dụ một user có nhiều session:

```text
user:sessions:U1
│
├── S1
├── S2
└── S3
```

Có thể:

```bash
SADD user:sessions:U1 S1
SADD user:sessions:U1 S2
SADD user:sessions:U1 S3
```

Khi muốn logout all devices:

```text
U1
│
├── S1 → revoke
├── S2 → revoke
└── S3 → revoke
```

---

# 17. Sorted Set

Sorted Set giống Set nhưng mỗi member có một **score**.

Ví dụ game leaderboard:

```text
player       score

Long         1000
An            850
Minh          700
```

Redis:

```bash
ZADD leaderboard 1000 Long
ZADD leaderboard 850 An
ZADD leaderboard 700 Minh
```

Lấy ranking:

```bash
ZREVRANGE leaderboard 0 9 WITHSCORES
```

Use case:

```text
Leaderboard
Ranking
Priority
Scheduling
Time-based ordering
```

---

# 18. Counter

Redis hỗ trợ tăng giảm số rất nhanh.

```bash
SET views:video:1 0
```

Mỗi lần xem:

```bash
INCR views:video:1
```

Kết quả:

```text
1
2
3
4
5
...
```

Có:

```bash
INCR
INCRBY
DECR
DECRBY
```

Ứng dụng:

```text
Page views
API usage
Login attempts
Rate limiting
Like counter
Download counter
```

---

# 19. Redis Cache

Giả sử API:

```text
GET /products/100
```

Không cache:

```text
Client
  ↓
NestJS
  ↓
PostgreSQL
  ↓
Product
```

Nếu request rất nhiều:

```text
1000 requests
      ↓
1000 database queries
```

Có Redis:

```text
Client
  ↓
NestJS
  ↓
Redis
  │
  ├── HIT → return
  │
  └── MISS
       ↓
    PostgreSQL
       ↓
    save Redis
       ↓
     return
```

---

# 20. Cache Hit

Redis có dữ liệu:

```text
GET product:100
        ↓
      FOUND
        ↓
    Cache HIT
```

Không cần query PostgreSQL.

---

# 21. Cache Miss

Redis không có:

```text
GET product:100
        ↓
       null
        ↓
   Cache MISS
        ↓
   PostgreSQL
        ↓
      Product
        ↓
   SET Redis
```

Pseudo-code:

```ts
async getProduct(id: number) {
  const key = `product:${id}`;

  const cached = await redis.get(key);

  if (cached) {
    return JSON.parse(cached);
  }

  const product = await database.findProduct(id);

  await redis.set(
    key,
    JSON.stringify(product),
    "EX",
    300,
  );

  return product;
}
```

---

# 22. Cache-Aside Pattern

Pattern trên gọi là:

```text
Cache-Aside
```

Flow:

```text
Application
    │
    │ GET cache
    ▼
 Redis
    │
    ├── HIT ─────────→ return
    │
    └── MISS
          │
          ▼
       Database
          │
          ▼
      SET cache
          │
          ▼
        return
```

Đây là pattern Redis rất phổ biến.

---

# 23. Cache Invalidation

Đây là phần khó.

Ví dụ cache:

```text
product:100

price = 100k
```

Database được update:

```text
price = 120k
```

Nhưng Redis vẫn:

```text
price = 100k
```

=> dữ liệu stale.

Khi update product có thể:

```text
UPDATE PostgreSQL
       ↓
DEL product:100
```

Request tiếp theo:

```text
Redis MISS
   ↓
PostgreSQL
   ↓
price = 120k
   ↓
cache lại
```

---

# 24. Redis trong Authentication

Đây là phần liên quan trực tiếp đến hệ thống bạn đang học.

Kiến trúc:

```text
             Authentication
                    │
          ┌─────────┴─────────┐
          │                   │
    Access Token        Refresh Token
        JWT                 Opaque
          │                   │
      short-lived           Redis
                              │
                         Session State
```

---

# 25. Session-per-device

User:

```text
U1
│
├── Laptop
│    └── S1
│
├── Phone
│    └── S2
│
└── iPad
     └── S3
```

Redis:

```text
session:U1:laptop
session:U1:phone
session:U1:ipad
```

Mỗi thiết bị có session riêng.

---

# 26. Session Object

Ví dụ:

```json
{
  "userId": "U1",
  "deviceId": "D1",
  "sessionId": "S1",
  "familyId": "F1",
  "refreshTokenHash": "H1",
  "createdAt": 1750000000,
  "expiresAt": 1750600000
}
```

Redis:

```text
session:U1:D1
        ↓
Session Object
```

---

# 27. Opaque Refresh Token + Redis

Server sinh:

```text
R1 = random secret
```

Browser:

```text
Cookie
│
└── refreshToken = R1
```

Server:

```text
hash(R1)
   ↓
H1
```

Redis:

```text
session:U1:D1
│
└── refreshTokenHash = H1
```

Không cần lưu raw `R1` trong Redis.

---

# 28. Verify Refresh Token

Browser gửi:

```text
R1
```

Server:

```text
R1
 ↓
SHA-256
 ↓
H1
```

Redis:

```text
stored H1
```

So sánh:

```text
hash(R1) === storedHash

H1 === H1

TRUE
```

Refresh token hợp lệ.

---

# 29. Refresh Token Rotation

Ban đầu:

```text
R1 ACTIVE
```

Client refresh:

```text
R1
 ↓
Server
 ↓
verify
 ↓
create R2
```

Kết quả:

```text
R1 ❌ used
R2 ✅ active
```

Chuỗi:

```text
R1 → R2 → R3 → R4
```

---

# 30. Reuse Detection

Giả sử:

```text
R1 → R2
```

R1 đã dùng.

Sau đó server lại nhận:

```text
R1
```

Server phát hiện:

```text
R1 status = USED
```

=> có khả năng token bị replay/copy.

```text
R1 xuất hiện lần 2
        ↓
Reuse Detection
        ↓
Revoke Session / Token Family
```

---

# 31. Redis cho Reuse Detection

Ví dụ:

```text
refresh:H1
{
  status: "used",
  familyId: "F1"
}

refresh:H2
{
  status: "active",
  familyId: "F1"
}
```

Nếu R1 xuất hiện:

```text
R1
 ↓
hash
 ↓
H1
 ↓
GET refresh:H1
 ↓
status = used
 ↓
REUSE DETECTED
```

---

# 32. Stateful Authentication

Khi authentication phụ thuộc vào dữ liệu server đang lưu:

```text
Client
  ↓
Refresh Token
  ↓
Server
  ↓
Redis
  ↓
Session
```

đây là:

```text
STATEFUL
```

Server nhớ trạng thái session.

---

# 33. Stateless Authentication

JWT Access Token có thể hoạt động:

```text
Client
 ↓
JWT
 ↓
Server
 ↓
verify signature
 ↓
check exp
 ↓
OK
```

không cần Redis cho mỗi request.

Đây có thể là:

```text
STATELESS
```

---

# 34. Hybrid Authentication

Một thiết kế phổ biến:

```text
Access Token
    │
    ├── JWT
    ├── short-lived
    └── stateless

Refresh Token
    │
    ├── opaque
    ├── rotation
    ├── Redis
    └── stateful
```

Bạn nhận được:

```text
JWT performance
+
Session control
```

---

# 35. Redis cho OTP

Ví dụ gửi OTP:

```text
OTP = 583291
```

Redis:

```bash
SET otp:email:user@example.com 583291 EX 300
```

Flow:

```text
User request OTP
      ↓
Generate OTP
      ↓
Redis
      ↓
TTL = 5 minutes
```

Sau 5 phút:

```text
key expired
```

OTP tự hết hạn.

---

# 36. Redis cho Rate Limiting

Ví dụ:

```text
POST /login
```

Muốn giới hạn:

```text
5 attempts / minute
```

Có thể dùng counter:

```text
login-attempt:IP
        ↓
        1
        2
        3
        4
        5
```

Nếu:

```text
> 5
```

thì:

```text
429 Too Many Requests
```

Sau TTL:

```text
counter expire
```

Có nhiều thuật toán rate limit tốt hơn counter đơn giản:

```text
Fixed Window
Sliding Window
Token Bucket
Leaky Bucket
```

---

# 37. Atomic Operation

Đây là khái niệm cực kỳ quan trọng.

Atomic nghĩa là operation được thực hiện như một đơn vị không bị chen ngang theo cách làm lộ trạng thái trung gian của chính command đó.

Ví dụ:

```bash
INCR counter
```

là atomic.

Nếu 100 request cùng:

```text
INCR counter
```

Redis xử lý việc tăng counter an toàn ở cấp command.

---

# 38. Race Condition

Giả sử refresh token:

```text
R1 = active
```

Hai request cùng lúc:

```text
Request A ── R1 ──┐
                  ├── Server
Request B ── R1 ──┘
```

Code ngây thơ:

```ts
const token = await redis.get("R1");

if (token.active) {
  await createNewToken();
}
```

Có thể xảy ra:

```text
A đọc R1 → active
B đọc R1 → active

A tạo R2
B tạo R3
```

Kết quả:

```text
      R1
     /  \
   R2    R3
```

Trong khi mong muốn:

```text
R1
 ↓
R2
```

Đây là **race condition**.

---

# 39. Vì sao Rotation là Security-Critical?

Refresh rotation cần đảm bảo:

```text
R1
 ↓
chỉ được consume thành công một lần
```

Không được:

```text
R1
├── R2
├── R3
└── R4
```

Do đó logic rotation nên atomic.

---

# 40. Redis Transaction

Redis hỗ trợ:

```text
MULTI
...
EXEC
```

Ví dụ:

```bash
MULTI
SET a 1
SET b 2
EXEC
```

Các command được queue rồi thực thi khi `EXEC`.

Nhưng Redis transaction không hoàn toàn giống transaction của PostgreSQL.

Đừng mặc định rằng:

```text
Redis MULTI/EXEC
=
SQL BEGIN/COMMIT/ROLLBACK
```

Hai mô hình có semantics khác nhau.

---

# 41. WATCH

Redis có:

```text
WATCH
MULTI
EXEC
```

`WATCH` có thể dùng optimistic locking.

Ý tưởng:

```text
WATCH token
   ↓
đọc token
   ↓
MULTI
   ↓
update
   ↓
EXEC
```

Nếu key bị người khác thay đổi trước `EXEC`, transaction có thể thất bại để application retry/xử lý.

---

# 42. Lua Script

Redis hỗ trợ chạy Lua script phía server.

Ý tưởng:

```text
CHECK R1
+
MARK R1 USED
+
CREATE R2
```

được gom vào logic atomic phía Redis.

Ví dụ conceptual:

```text
if R1 == active
    mark R1 used
    create R2
    return success
else
    return failure
end
```

Điều này rất hữu ích cho:

```text
Token Rotation
Rate Limiting
Distributed coordination
Conditional update
```

---

# 43. SET NX

Redis:

```bash
SET lock:payment:123 abc NX EX 10
```

`NX` nghĩa là:

```text
chỉ SET nếu key chưa tồn tại
```

Có thể hữu ích cho lock/idempotency-like patterns tùy thiết kế.

Ví dụ:

```text
Request A
   ↓
SET lock NX
   ↓
SUCCESS
```

Request B:

```text
SET lock NX
   ↓
FAIL
```

Nhưng distributed lock đúng chuẩn có nhiều chi tiết hơn chỉ một lệnh `SET NX`.

---

# 44. EX và PX

Expire theo giây:

```bash
SET key value EX 60
```

Expire theo millisecond:

```bash
SET key value PX 5000
```

---

# 45. NX và XX

```text
NX
→ chỉ set nếu key chưa tồn tại

XX
→ chỉ set nếu key đã tồn tại
```

Ví dụ:

```bash
SET user:1 active NX
```

---

# 46. Pub/Sub

Redis hỗ trợ publish/subscribe.

Ví dụ:

```text
Service A
   │
   │ PUBLISH
   ▼
Channel
   │
   ├── Service B
   ├── Service C
   └── Service D
```

Publisher:

```bash
PUBLISH notifications "hello"
```

Subscriber:

```bash
SUBSCRIBE notifications
```

Use case:

```text
Real-time events
Notifications
Simple service messaging
```

Nhưng Redis Pub/Sub không phải durable queue: subscriber offline có thể bỏ lỡ message.

---

# 47. Redis Streams

Nếu cần event/message có khả năng lưu lại tốt hơn Pub/Sub, Redis có:

```text
Streams
```

Concept:

```text
Producer
   ↓
Redis Stream
   ↓
Consumer Group
   │
   ├── Worker 1
   ├── Worker 2
   └── Worker 3
```

Redis Streams hỗ trợ các khái niệm như:

```text
Message IDs
Consumer Groups
Acknowledgement
Pending entries
```

---

# 48. Redis Persistence

Redis chủ yếu thao tác dữ liệu trong memory nhưng có cơ chế persistence xuống disk.

Hai cơ chế quan trọng:

```text
RDB
AOF
```

---

# 49. RDB

RDB tạo snapshot dữ liệu tại một thời điểm.

```text
RAM
 ↓
snapshot
 ↓
dump.rdb
```

Ưu điểm:

```text
compact
backup thuận tiện
restart/load có thể nhanh
```

Trade-off:

```text
có thể mất dữ liệu từ snapshot gần nhất
```

---

# 50. AOF

AOF = **Append Only File**.

Redis ghi lại các operation thay đổi dữ liệu.

Concept:

```text
SET user 1
INCR counter
DEL session
...
```

Khi restart có thể replay log để phục hồi state.

AOF thường giảm cửa sổ mất dữ liệu so với snapshot-only, tùy cấu hình fsync.

---

# 51. RDB + AOF

Có thể cấu hình persistence theo nhu cầu.

```text
Redis
│
├── RDB
│    └── snapshot
│
└── AOF
     └── operation log
```

Việc lựa chọn phụ thuộc:

```text
performance
durability
recovery requirements
```

---

# 52. Redis có phải Database không?

Có.

Redis là một data store/database.

Nhưng không nên suy nghĩ:

```text
Redis sẽ luôn thay thế PostgreSQL
```

Thông thường:

```text
PostgreSQL
→ persistent business data
→ relational queries
→ constraints
→ transactions

Redis
→ fast temporary/state data
→ cache
→ session
→ counters
→ rate limiting
→ coordination
```

Ví dụ e-commerce:

```text
PostgreSQL
├── users
├── products
├── orders
├── payments
└── inventory

Redis
├── cache
├── sessions
├── refresh tokens
├── OTP
├── rate limits
└── temporary state
```

---

# 53. Redis Memory

Vì Redis thường sử dụng RAM nên phải quan tâm:

```text
memory usage
```

Không thể lưu vô hạn.

Redis có:

```text
maxmemory
```

và các eviction policy.

---

# 54. Eviction

Khi Redis đạt giới hạn memory, tùy cấu hình nó có thể:

```text
reject write
```

hoặc loại bỏ một số key theo policy.

Các chiến lược có thể liên quan tới:

```text
LRU-like
LFU-like
TTL
random
```

Điều này đặc biệt quan trọng.

Nếu Redis của bạn lưu:

```text
CACHE
```

eviction thường có thể chấp nhận được.

Nhưng nếu Redis đang lưu:

```text
SECURITY SESSION STATE
```

mất key có thể làm user bị logout hoặc ảnh hưởng authentication flow.

Vì vậy cần thiết kế memory/policy cẩn thận.

---

# 55. Cache và Session không hoàn toàn giống nhau

Cache:

```text
Redis mất dữ liệu
     ↓
query PostgreSQL lại
     ↓
rebuild cache
```

Session:

```text
Redis mất dữ liệu
     ↓
session không còn
     ↓
user có thể bị logout
```

Vì vậy:

```text
Cache = disposable

Session = security/application state
```

nên yêu cầu reliability khác nhau.

---

# 56. Redis Security

Redis production không nên expose trực tiếp ra Internet.

Sai:

```text
Internet
   ↓
:6379
   ↓
Redis
```

Nên:

```text
Internet
   ↓
Nginx / API
   ↓
NestJS
   ↓
Private Network
   ↓
Redis
```

Redis nên được bảo vệ bằng:

```text
network isolation
authentication/ACL
TLS nếu cần
least privilege
firewall
monitoring
secure configuration
```

---

# 57. Không lưu Plain Refresh Token

Không nên:

```text
Redis

refreshToken = R1
```

Nếu Redis bị leak:

```text
Attacker
   ↓
R1
   ↓
POST /refresh
   ↓
Access Token
```

Nên lưu:

```text
hash(R1) = H1
```

Redis:

```text
refreshTokenHash = H1
```

Attacker lấy H1 không thể trực tiếp gửi H1 thay cho R1.

---

# 58. Hash không phải Encryption

```text
Encryption
plaintext
   ↓ key
ciphertext
   ↓ key
plaintext
```

Có thể giải mã với key.

Hash:

```text
R1
 ↓
SHA-256
 ↓
H1
```

Không có operation:

```text
H1
 ↓
"decrypt"
 ↓
R1
```

---

# 59. Brute Force

Attacker có thể thử:

```text
guess
 ↓
hash
 ↓
compare
```

Nếu token yếu:

```text
123456
```

có thể đoán được.

Nếu:

```ts
randomBytes(32)
```

thì có 256 bit entropy trước encoding, làm brute force toàn bộ không gian trở nên không khả thi trong thực tế.

---

# 60. Redis Key Design

Một hệ thống auth có thể dùng:

```text
auth:session:{userId}:{deviceId}

auth:refresh:{tokenId}

auth:family:{familyId}

auth:user-sessions:{userId}

auth:otp:{email}

rate-limit:login:{ip}
```

Điều này giúp Redis dễ quản lý.

---

# 61. Không dùng KEYS trong Production cho scan lớn

Command:

```bash
KEYS *
```

có thể rất tốn tài nguyên nếu database có rất nhiều key.

Ví dụ:

```text
millions of keys
```

Redis phải duyệt keyspace phù hợp với pattern.

Trong production thường ưu tiên:

```bash
SCAN
```

Ví dụ:

```bash
SCAN 0 MATCH session:* COUNT 100
```

`SCAN` trả cursor để tiếp tục iteration.

---

# 62. Không nên phụ thuộc vào scan để logout all

Một thiết kế không tốt:

```text
KEYS session:user123:*
```

rồi xóa.

Tốt hơn là có index:

```text
user:sessions:U1
    ↓
SET
├── S1
├── S2
└── S3
```

Khi logout all:

```text
SMEMBERS user:sessions:U1
        ↓
S1 S2 S3
        ↓
revoke từng session
```

---

# 63. Pipelining

Bình thường:

```text
Client → Redis command 1
Redis  → Client

Client → Redis command 2
Redis  → Client

Client → Redis command 3
Redis  → Client
```

Mỗi lần có network round trip.

Pipeline:

```text
Client
 │
 ├── command 1
 ├── command 2
 └── command 3
        ↓
      Redis
        ↓
   nhiều response
```

Giảm network round trips.

Pipeline không đồng nghĩa với transaction/atomicity.

---

# 64. Pipeline vs Transaction

```text
Pipeline
→ tối ưu network/performance

Transaction
→ nhóm command theo transaction semantics của Redis

Lua
→ chạy logic server-side atomically
```

Không nên nhầm ba thứ này.

---

# 65. Redis và NestJS

Ví dụ sử dụng `ioredis`:

```ts
import Redis from "ioredis";

const redis = new Redis({
  host: "localhost",
  port: 6379,
});
```

Set:

```ts
await redis.set("name", "Long");
```

Get:

```ts
const name = await redis.get("name");
```

Delete:

```ts
await redis.del("name");
```

Expire:

```ts
await redis.set(
  "otp:user1",
  "583291",
  "EX",
  300,
);
```

---

# 66. Redis Service trong NestJS

Có thể tạo:

```text
RedisModule
     ↓
RedisService
     ↓
Redis Client
```

Các module khác:

```text
AuthService ──────┐
                  │
OtpService ───────┼──→ RedisService
                  │
CacheService ─────┘
```

Nhờ vậy connection/config Redis được quản lý tập trung.

---

# 67. Ví dụ SessionService

```ts
@Injectable()
export class SessionService {
  constructor(
    private readonly redis: Redis,
  ) {}

  async getSession(
    userId: string,
    deviceId: string,
  ) {
    const key =
      `session:${userId}:${deviceId}`;

    const data = await this.redis.get(key);

    if (!data) {
      return null;
    }

    return JSON.parse(data);
  }
}
```

---

# 68. Tạo Session

```ts
async createSession(
  userId: string,
  deviceId: string,
  session: SessionData,
) {
  const key =
    `session:${userId}:${deviceId}`;

  await this.redis.set(
    key,
    JSON.stringify(session),
    "EX",
    60 * 60 * 24 * 7,
  );
}
```

TTL:

```text
7 days
```

---

# 69. Revoke Session

```ts
async revokeSession(
  userId: string,
  deviceId: string,
) {
  const key =
    `session:${userId}:${deviceId}`;

  await this.redis.del(key);
}
```

Sau đó:

```text
refresh request
      ↓
get session
      ↓
null
      ↓
401
```

Refresh token client còn giữ cũng không còn sử dụng được nếu verification phụ thuộc vào session đó.

---

# 70. Redis trong Docker

Ví dụ development:

```yaml
services:
  redis:
    image: redis:alpine
    ports:
      - "6379:6379"
```

NestJS chạy trên host:

```env
REDIS_HOST=localhost
REDIS_PORT=6379
```

Nếu NestJS cũng nằm trong Docker Compose:

```text
NestJS container
      ↓
redis:6379
      ↓
Redis container
```

thì thường:

```env
REDIS_HOST=redis
REDIS_PORT=6379
```

không phải:

```env
REDIS_HOST=localhost
```

vì `localhost` bên trong NestJS container là chính container NestJS.

---

# 71. Docker Volume

Nếu cần persistence:

```yaml
services:
  redis:
    image: redis:alpine

    volumes:
      - redis-data:/data

volumes:
  redis-data:
```

Concept:

```text
Redis Container
      │
      ▼
   /data
      │
      ▼
Docker Volume
```

Container bị recreate nhưng volume có thể vẫn giữ dữ liệu.

---

# 72. Redis CLI

Vào Redis:

```bash
redis-cli
```

Kiểm tra:

```bash
PING
```

Response:

```text
PONG
```

Set:

```bash
SET name Long
```

Get:

```bash
GET name
```

Delete:

```bash
DEL name
```

---

# 73. Một số Command cần nhớ

```text
SET
GET
DEL
EXISTS
EXPIRE
TTL

HSET
HGET
HGETALL
HDEL

SADD
SREM
SMEMBERS
SISMEMBER

LPUSH
RPUSH
LPOP
RPOP
LRANGE

ZADD
ZRANGE
ZREVRANGE

INCR
DECR

SCAN
```

---

# 74. EXISTS

Kiểm tra key:

```bash
EXISTS session:U1:D1
```

Có:

```text
1
```

Không có:

```text
0
```

---

# 75. Redis Null

Ví dụ:

```ts
const value =
  await redis.get("something");
```

Nếu key không tồn tại:

```ts
value === null
```

Do đó thường:

```ts
if (!value) {
  // handle not found
}
```

Nhưng khi dữ liệu có thể chứa các giá trị falsy theo cách parse của application, nên kiểm tra semantics cẩn thận.

---

# 76. Serialization

Redis String không tự hiểu TypeScript object.

Sai:

```ts
await redis.set("user", user);
```

Thường serialize:

```ts
await redis.set(
  "user",
  JSON.stringify(user),
);
```

Lấy:

```ts
const raw =
  await redis.get("user");

const user =
  JSON.parse(raw!);
```

Hoặc sử dụng Redis Hash nếu phù hợp.

---

# 77. Naming Strategy

Nên centralize key generation.

Ví dụ:

```ts
export const RedisKeys = {
  SESSION: (
    userId: string,
    deviceId: string,
  ) =>
    `auth:session:${userId}:${deviceId}`,

  REFRESH: (tokenHash: string) =>
    `auth:refresh:${tokenHash}`,

  FAMILY: (familyId: string) =>
    `auth:family:${familyId}`,
};
```

Thay vì rải:

```ts
`session:${userId}:${deviceId}`
```

khắp codebase.

Lợi ích:

```text
consistency
maintainability
refactoring
fewer typo bugs
```

---

# 78. Redis Failure

Redis có thể down.

Ví dụ:

```text
NestJS
  ↓
Redis
  ❌
```

Application phải xác định rõ:

```text
Redis dùng cache?
→ có thể fallback DB

Redis dùng auth session?
→ thường fail closed

Redis dùng rate limit?
→ cần quyết định security policy
```

Đặc biệt với authentication:

```text
Không kiểm tra được session
```

không nên tự động suy ra:

```text
"thôi cho qua"
```

vì đây là security-critical path.

---

# 79. Fail Open vs Fail Closed

**Fail Open**

```text
Security service lỗi
      ↓
cho request qua
```

Có thể tăng availability nhưng nguy hiểm với một số security controls.

**Fail Closed**

```text
Security service lỗi
      ↓
từ chối request
```

Ví dụ refresh-token verification phụ thuộc Redis:

```text
Redis unavailable
      ↓
không xác minh được refresh token
      ↓
reject refresh
```

thường an toàn hơn.

---

# 80. Redis không nên chứa mọi thứ

Không phải thấy Redis nhanh rồi:

```text
"Đưa toàn bộ database sang Redis!"
```

Redis có trade-off:

```text
RAM cost
persistence model
query capabilities
relationships
operational complexity
```

Dữ liệu business quan trọng thường vẫn phù hợp với PostgreSQL/MySQL.

---

# 81. Kiến trúc E-commerce thực tế

Ví dụ project NextJS + NestJS của bạn:

```text
                   CLIENT
                     │
                  Next.js
                     │
                     ▼
                   Nginx
                     │
                     ▼
                  NestJS
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
   PostgreSQL      Redis        Queue
        │            │
        │            ├── Cache
        │            ├── Session
        │            ├── Refresh Token
        │            ├── OTP
        │            └── Rate Limit
        │
        ├── Users
        ├── Products
        ├── Orders
        └── Payments
```

---

# 82. Authentication Architecture

Một architecture tương đối mạnh:

```text
Login
 │
 ▼
AuthService
 │
 ├── verify password
 │
 ├── create device session
 │
 ├── create access JWT
 │
 └── create opaque refresh token
           │
           ▼
         Redis
```

Client:

```text
Access Token
→ memory

Refresh Token
→ Secure + HttpOnly Cookie
```

Redis:

```text
Session
Token Hash
Token Family
Token Status
TTL
```

---

# 83. Refresh Flow

```text
Access Token expired
        ↓
Client receives 401
        ↓
POST /auth/refresh
        ↓
Cookie sends R1
        ↓
NestJS
        ↓
hash(R1)
        ↓
Redis
        ↓
validate session/token
        ↓
atomic rotation
        ↓
R1 → used
R2 → active
        ↓
return A2
Set-Cookie R2
```

---

# 84. Reuse Attack

```text
             R1
           /    \
        User    Hacker
```

Hacker dùng trước:

```text
R1
 ↓
R2
```

User dùng R1:

```text
R1
 ↓
Redis
 ↓
status = USED
 ↓
REUSE DETECTED
 ↓
REVOKE FAMILY
```

Kết quả:

```text
R1 ❌
R2 ❌
Session ❌
```

Đây là một trong những lý do Redis rất hữu ích trong auth state.

---

# 85. Mental Model quan trọng nhất

Đừng nhớ Redis đơn giản là:

```text
Redis = cache
```

Hãy nhớ:

```text
Redis
=
Fast server-side state/data-structure store
```

Nó có thể giữ:

```text
"Thứ gì đang xảy ra ngay lúc này?"
```

Ví dụ:

```text
Ai đang có session?
OTP nào còn sống?
Token nào đã dùng?
Request nào đang bị rate limit?
Cache nào còn valid?
Counter hiện tại bao nhiêu?
Lock nào đang tồn tại?
Job/event nào đang chờ?
```

Trong khi PostgreSQL thường giữ:

```text
"Business data lâu dài của hệ thống là gì?"
```

---

# 86. Redis vs PostgreSQL

|Redis|PostgreSQL|
|---|---|
|In-memory oriented|Disk-backed relational DB|
|Key-value/data structures|Tables/rows|
|Rất nhanh cho nhiều workload|Query relational mạnh|
|TTL native|Không dùng TTL theo kiểu Redis|
|Cache tốt|Persistent business data tốt|
|Session tốt|User/order/product tốt|
|Counter tốt|Complex reporting/query tốt|
|Rate limit tốt|Relationship/constraint tốt|

Không phải:

```text
Redis VS PostgreSQL
```

mà thường là:

```text
Redis + PostgreSQL
```

---

# 87. Redis vs Memory trong NestJS

Bạn có thể hỏi:

> "Sao không dùng Map?"

Ví dụ:

```ts
const sessions = new Map();
```

Vấn đề khi scale:

```text
NestJS Instance A
sessions = {...}

NestJS Instance B
sessions = {...}
```

Memory không chia sẻ.

Client request:

```text
Request 1 → Server A

Request 2 → Server B
```

B không biết state của A.

Redis giải quyết:

```text
        Redis
       /     \
      /       \
NestJS A     NestJS B
```

Cả hai cùng truy cập state chung.

---

# 88. Redis trong Horizontal Scaling

Ví dụ:

```text
             Load Balancer
            /      |      \
           /       |       \
      NestJS 1  NestJS 2  NestJS 3
           \       |       /
            \      |      /
                 Redis
```

Session nằm Redis nên request vào instance nào cũng có thể kiểm tra session.

Đây là lý do Redis rất phổ biến trong distributed backend.

---

# 89. Redis Cluster

Khi hệ thống lớn, Redis có thể chạy theo mô hình cluster/sharding.

Concept:

```text
          Redis Cluster
        /       |       \
     Node A   Node B   Node C
```

Keyspace được phân chia giữa các node.

Mục tiêu:

```text
scale
throughput
larger dataset
```

Redis Cluster có những constraint riêng đối với multi-key operation.

---

# 90. Redis Replication

Có thể có:

```text
Primary
   │
   ├── Replica 1
   └── Replica 2
```

Primary nhận write.

Replica nhận replication data.

Mục tiêu:

```text
availability
read scaling trong một số kiến trúc
failover support
```

---

# 91. Sentinel

Redis Sentinel được dùng với Redis deployment không phải Cluster để hỗ trợ:

```text
monitoring
failure detection
automatic failover
service discovery
```

Concept:

```text
Primary ❌
   ↓
Sentinel phát hiện
   ↓
promote Replica
   ↓
new Primary
```

---

# 92. Production Redis

Development:

```text
NestJS
 ↓
Redis container
```

Production có thể:

```text
Application
    ↓
Private Network
    ↓
Redis deployment
    │
    ├── replication
    ├── persistence
    ├── monitoring
    ├── backup
    ├── authentication
    └── failover
```

---

# 93. Monitoring

Một số thứ nên monitor:

```text
Memory usage
Connected clients
Operations/sec
Latency
Hit rate
Evictions
Expired keys
Replication health
Persistence errors
CPU
Network
```

Nếu Redis là auth/session store thì:

```text
availability
latency
memory
eviction
```

đặc biệt quan trọng.

---

# 94. Cache Hit Rate

Ví dụ:

```text
1000 cache requests

900 HIT
100 MISS
```

Hit rate:

```text
900 / 1000 = 90%
```

Nếu cache hit thấp, có thể cần xem:

```text
TTL
key design
cache strategy
workload
invalidation
```

---

# 95. Những lỗi người mới thường mắc

### Lỗi 1

```text
Redis = database thay thế mọi DB
```

Không đúng.

---

### Lỗi 2

Không đặt TTL cho temporary data.

```text
OTP
session
cache
temporary token
```

có thể tồn tại mãi.

---

### Lỗi 3

Lưu refresh token raw.

```text
Redis leak
↓
token leak
```

---

### Lỗi 4

Dùng:

```bash
KEYS *
```

trong production trên keyspace lớn.

---

### Lỗi 5

Không xử lý Redis down.

---

### Lỗi 6

Cho Redis public Internet access.

---

### Lỗi 7

Dùng:

```text
GET
↓
logic
↓
SET
```

cho operation cần atomicity.

---

### Lỗi 8

Nhầm pipeline với transaction.

---

### Lỗi 9

Không nghĩ đến race condition.

---

### Lỗi 10

Không quản lý memory/eviction.

---

# 96. Redis Learning Roadmap

Nên học theo thứ tự:

```text
Redis
 │
 ├── 1. Key / Value
 │
 ├── 2. SET / GET / DEL
 │
 ├── 3. TTL / EXPIRE
 │
 ├── 4. String
 │
 ├── 5. Hash
 │
 ├── 6. List
 │
 ├── 7. Set
 │
 ├── 8. Sorted Set
 │
 ├── 9. Counter
 │
 ├── 10. Cache
 │
 ├── 11. Cache invalidation
 │
 ├── 12. Session
 │
 ├── 13. OTP
 │
 ├── 14. Rate limiting
 │
 ├── 15. Transaction / WATCH
 │
 ├── 16. Lua / atomic operations
 │
 ├── 17. Pub/Sub
 │
 ├── 18. Streams
 │
 ├── 19. Persistence
 │
 ├── 20. Memory / Eviction
 │
 ├── 21. Security
 │
 ├── 22. Replication
 │
 ├── 23. Sentinel
 │
 ├── 24. Cluster
 │
 └── 25. Production monitoring
```

---

# 97. Roadmap riêng cho NestJS Backend

Với mục tiêu backend của bạn, ưu tiên:

```text
Redis Fundamentals
        ↓
TTL
        ↓
Caching
        ↓
NestJS Redis Service
        ↓
OTP
        ↓
Session-per-device
        ↓
Opaque Refresh Token
        ↓
Refresh Token Rotation
        ↓
Reuse Detection
        ↓
Rate Limiting
        ↓
Atomic Operations
        ↓
Lua / WATCH
        ↓
Queue / Streams
        ↓
Distributed Systems
```

---

# 98. Các khái niệm nên liên kết trong Obsidian

```text
[[Redis]]
│
├── [[Caching]]
├── [[Cache Invalidation]]
├── [[TTL]]
├── [[Session]]
├── [[Stateful Authentication]]
├── [[JWT]]
├── [[Opaque Refresh Token]]
├── [[Refresh Token Rotation]]
├── [[Refresh Token Reuse Detection]]
├── [[Session Per Device]]
├── [[Rate Limiting]]
├── [[Race Condition]]
├── [[Atomic Operation]]
├── [[Distributed Lock]]
├── [[Pub Sub]]
├── [[Redis Streams]]
├── [[Horizontal Scaling]]
└── [[NestJS]]
```

---

# 99. Cheat Sheet

```text
SET key value
→ lưu

GET key
→ lấy

DEL key
→ xóa

EXPIRE key seconds
→ đặt TTL

TTL key
→ xem TTL

EXISTS key
→ kiểm tra tồn tại

INCR key
→ +1

DECR key
→ -1
```

Hash:

```text
HSET
HGET
HGETALL
HDEL
```

Set:

```text
SADD
SREM
SMEMBERS
SISMEMBER
```

List:

```text
LPUSH
RPUSH
LPOP
RPOP
LRANGE
```

Sorted Set:

```text
ZADD
ZRANGE
ZREVRANGE
```

Production iteration:

```text
SCAN
```

---

# 100. Câu quan trọng nhất

Nếu chỉ nhớ một mental model về Redis, hãy nhớ:

```text
              PostgreSQL
                  │
                  ▼
        Business data lâu dài

User / Product / Order / Payment


                Redis
                  │
                  ▼
         Fast application state

Cache
Session
Refresh Token
OTP
Rate Limit
Counter
Lock
Temporary State
Queue/Event
```

Và đối với authentication mà bạn đang học:

```text
JWT Access Token
        ↓
short-lived
        ↓
có thể stateless


Opaque Refresh Token
        ↓
Redis
        ↓
stateful
        ↓
Session-per-device
        ↓
Rotation
        ↓
Reuse Detection
        ↓
Revocation
```

Đây là một trong những ứng dụng quan trọng nhất của Redis khi xây dựng backend authentication có khả năng quản lý session tốt.