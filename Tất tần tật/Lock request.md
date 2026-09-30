# Lock request khi user double-click gọi là gì?

Có **nhiều tên gọi** tuỳ theo ngữ cảnh và cách implement. Dưới đây là tổng hợp đầy đủ.

---

## 📛 Các tên gọi phổ biến

| Tên gọi | Ngữ cảnh | Mô tả |
|---------|----------|-------|
| **Idempotency** | API design, payment | Cùng 1 request gửi nhiều lần → kết quả giống nhau |
| **Idempotency Key** | Stripe, PayPal, Momo, VNPay | Client gửi key unique → server dedupe |
| **Request Deduplication** | General | Chặn request trùng lặp |
| **Distributed Lock** | Redis, DB | Lock resource giữa nhiều instance |
| **Mutex (Mutual Exclusion)** | Concurrency | Chỉ 1 request xử lý tại 1 thời điểm |
| **Debounce** | Frontend | Chờ user ngừng click mới gửi |
| **Throttle** | Frontend/Backend | Giới hạn số lần gọi trong khoảng thời gian |
| **Rate Limiting** | Backend | Giới hạn request theo IP/user |
| **Double-Submit Prevention** | Form submission | Chặn submit form 2 lần |
| **Optimistic Locking** | Database | Dùng version/timestamp để phát hiện conflict |
| **Pessimistic Locking** | Database | Lock row trước khi update |
| **Distributed Mutex** | Microservices | Mutex giữa nhiều instance |
| **Redlock** | Redis | Algorithm lock phân tán của Redis |
| **Semaphore** | Concurrency | Giới hạn N request đồng thời |

---

## 🎯 Phân biệt rõ 3 nhóm chính

### 1. **Idempotency** — Cùng request → cùng kết quả

Đây là khái niệm **quan trọng nhất** trong thanh toán.

**Định nghĩa:** 1 operation có thể gọi **N lần** nhưng kết quả **giống hệt** gọi 1 lần.

**Ví dụ:**
```
POST /payments/create
{
  "cartId": "abc",
  "provider": "vnpay",
  "idempotencyKey": "user-123-order-456-1704067200"   ← Client tự sinh
}
```

Server:
- Lần 1: Chưa có key → tạo payment → lưu key vào Redis/DB
- Lần 2 (double-click): Key đã có → **trả về kết quả cũ**, không tạo mới

**Ứng dụng:**
- Thanh toán: **BẮT BUỘC** (Stripe, PayPal, Momo, VNPay)
- Tạo order: nên có
- Gửi email: nên có

### 2. **Distributed Lock / Mutex** — Chỉ 1 request xử lý tại 1 thời điểm

**Định nghĩa:** Đảm bảo **chỉ 1 request** được xử lý resource tại 1 thời điểm. Request khác phải **chờ** hoặc **bị từ chối**.

**Ví dụ:**
```
POST /carts/items  (user double-click)
```

Server:
- Request 1: Acquire lock `cart:lock:user-123` → xử lý → release
- Request 2: Lock đang giữ → **chờ** hoặc **409 Conflict**

**Ứng dụng:**
- Trừ stock: **BẮT BUỘC**
- Update cart
- Merge cart
- Webhook payment

### 3. **Rate Limiting / Throttling** — Giới hạn số lần gọi

**Định nghĩa:** Giới hạn **N request / khoảng thời gian** cho mỗi user/IP.

**Ví dụ:**
```typescript
@Throttle({ default: { ttl: 60_000, limit: 30 } })  // 30 req/phút
```

**Ứng dụng:**
- Chống spam, DDoS
- Chống brute force password/OTP
- Bảo vệ API

**Lưu ý:** Rate limiting **không đủ** để chặn double-click vì:
- Nếu limit = 30/phút → user click 2 lần trong 100ms vẫn qua
- Rate limiting chỉ chặn **tần suất cao**, không chặn **trùng lặp**

---

## 🎯 Vậy double-click thì dùng cái gì?

**Tuỳ vào mục đích:**

| Mục đích | Giải pháp |
|----------|-----------|
| Chặn tạo 2 order/payment | **Idempotency Key** |
| Chặn 2 request update cùng lúc | **Distributed Lock** |
| Chặn spam API | **Rate Limiting** |
| UX mượt (không gửi 2 request) | **Debounce/Throttle ở frontend** |
| Chặn submit form 2 lần | **Double-Submit Prevention** |

**Thực tế:** Thường dùng **kết hợp**:
- Frontend: **Disable button** sau click + **debounce**
- Backend: **Idempotency Key** (cho payment/order) + **Distributed Lock** (cho stock/cart)

---

## 🎯 Cách implement từng loại

### 1. Idempotency Key (Redis SET NX)

**Client:**
```typescript
const idempotencyKey = `${userId}-${cartId}-${Date.now()}`;

fetch('/api/payments/create', {
  method: 'POST',
  headers: {
    'Idempotency-Key': idempotencyKey,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ cartId, provider: 'vnpay' }),
});
```

**Server:**
```typescript
async createPayment(userId, dto, idempotencyKey) {
  const key = `idempotency:payment:${idempotencyKey}`;
  
  // 1. Thử acquire lock (SET NX EX)
  const acquired = await this.redis.setNX(key, 'processing', 300);
  
  if (!acquired) {
    // 2. Đã có request đang xử lý hoặc đã xử lý xong
    const result = await this.redis.get(key);
    
    if (result === 'processing') {
      throw new ConflictException('Request đang được xử lý');
    }
    
    // 3. Đã xử lý xong → trả kết quả cũ
    return JSON.parse(result);
  }
  
  try {
    // 4. Xử lý
    const payment = await this.doCreatePayment(userId, dto);
    
    // 5. Lưu kết quả (TTL dài)
    await this.redis.set(key, JSON.stringify(payment), 86400);
    
    return payment;
  } catch (error) {
    // 6. Fail → release để retry
    await this.redis.del(key);
    throw error;
  }
}
```

**Redis commands:**
```
SET idempotency:payment:xxx "processing" EX 300 NX   ← Atomic
```

---

### 2. Distributed Lock (Redis SET NX)

**Pattern cơ bản:**

```typescript
async addItem(userId, deviceId, dto) {
  const lockKey = `lock:cart:${userId ?? deviceId}`;
  const lockValue = randomBytes(16).toString('hex');  // Unique value
  const lockTtl = 10;  // seconds
  
  // 1. Acquire lock
  const acquired = await this.redis.setNX(lockKey, lockValue, lockTtl);
  
  if (!acquired) {
    throw new ConflictException('Thao tác đang được xử lý, vui lòng thử lại');
  }
  
  try {
    // 2. Xử lý (đã có lock)
    return await this.doAddItem(userId, deviceId, dto);
  } finally {
    // 3. Release lock (chỉ xoá nếu value khớp — tránh xoá lock của request khác)
    await this.redis.releaseLock(lockKey, lockValue);
  }
}
```

**`releaseLock` script Lua (atomic):**

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

Code bạn đã có sẵn trong `RedisService`:

```typescript
async releaseLock(key: string, value: string): Promise<boolean> {
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `;
  const result = await this.redis.eval(script, 1, key, value);
  return result === 1;
}
```

**Tại sao cần `value` unique?** Tránh request A xoá lock của request B (nếu A bị timeout, lock hết hạn, B acquire, A release → xoá lock của B).

---

### 3. Double-Submit Prevention (Frontend + Backend)

**Frontend — Disable button:**

```typescript
const [submitting, setSubmitting] = useState(false);

const handleSubmit = async () => {
  if (submitting) return;   // ← Chặn click lần 2
  setSubmitting(true);
  
  try {
    await fetch('/api/orders', { ... });
  } finally {
    setSubmitting(false);
  }
};
```

**Frontend — Debounce:**

```typescript
import { debounce } from 'lodash';

const handleClick = debounce(async () => {
  await fetch('/api/orders', { ... });
}, 1000);  // Chờ 1s sau lần click cuối
```

**Frontend — Throttle:**

```typescript
import { throttle } from 'lodash';

const handleClick = throttle(async () => {
  await fetch('/api/orders', { ... });
}, 1000);  // Chỉ gọi 1 lần mỗi 1s
```

**Backend — CSRF token (đã có trong code bạn):**

CSRF token là 1 dạng double-submit prevention:
- Mỗi form có 1 token unique
- Submit lần 1 → token invalidated
- Submit lần 2 → token đã dùng → **403**

---

### 4. Optimistic Locking (Database)

**Entity có version column:**

```typescript
@Entity('carts')
export class Cart {
  @VersionColumn()
  version!: number;
}
```

**Update:**

```typescript
const cart = await this.cartRepository.findOne({ where: { id } });
cart.items.push(newItem);
await this.cartRepository.save(cart);
// TypeORM tự: UPDATE carts SET ..., version = version + 1 
//             WHERE id = ? AND version = ?
```

Nếu version đã thay đổi → **0 rows affected** → throw `OptimisticLockVersionMismatchError`.

**Ứng dụng:** Khi bạn muốn **detect conflict** thay vì **chặn**.

---

### 5. Pessimistic Locking (Database)

**Lock row trước khi update:**

```typescript
return this.dataSource.transaction(async (manager) => {
  const product = await manager.findOne(Product, {
    where: { id },
    lock: { mode: 'pessimistic_write' },   // ← SELECT ... FOR UPDATE
  });
  
  if (product.stock < quantity) throw new BadRequestException('Out of stock');
  
  product.stock -= quantity;
  return manager.save(product);
});
```

**SQL sinh ra:**
```sql
BEGIN;
SELECT * FROM products WHERE id = 'x' FOR UPDATE;   ← Lock row
UPDATE products SET stock = stock - 1 WHERE id = 'x';
COMMIT;
```

Request khác muốn `SELECT ... FOR UPDATE` cùng row → **chờ** đến khi transaction 1 commit/rollback.

**Ứng dụng:**
- Trừ stock: **BẮT BUỘC**
- Transfer tiền
- Update balance

---

## 🎯 So sánh các giải pháp

| Giải pháp | Chặn double-click? | Chống race condition? | Ứng dụng |
|-----------|:------------------:|:---------------------:|----------|
| **Debounce** | ✅ | ❌ | Frontend UX |
| **Throttle** | ✅ | ❌ | Frontend UX |
| **Disable button** | ✅ | ❌ | Frontend UX |
| **Rate Limiting** | ⚠️ Một phần | ❌ | Chống spam |
| **Idempotency Key** | ✅ | ✅ | Payment, Order |
| **Distributed Lock** | ✅ | ✅ | Cart, Stock |
| **Optimistic Lock** | ❌ | ✅ | Detect conflict |
| **Pessimistic Lock** | ✅ | ✅ | Stock, Balance |
| **Double-Submit Token** | ✅ | ✅ | Form submission |

---

## 🎯 Thực tế nên dùng gì?

### Cho **Payment / Order** → **Idempotency Key**

Vì:
- Client có thể retry nếu timeout
- Server cần trả **cùng kết quả** cho cùng key
- Tiêu chuẩn công nghiệp (Stripe, PayPal, Momo...)

### Cho **Cart / Stock** → **Distributed Lock** + **Pessimistic Lock**

Vì:
- Cần **chặn** request thứ 2, không phải trả kết quả cũ
- Cần **bảo vệ** resource trong khoảng thời gian ngắn
- Cart có thể merge, update... → cần lock

### Cho **Frontend** → **Disable button + Debounce**

Vì:
- UX tốt hơn (không gửi request thừa)
- Giảm tải server
- **Không thay thế** backend protection (vì client có thể bypass)

### Cho **API chung** → **Rate Limiting** (đã có `@Throttle`)

Vì:
- Chống DDoS, spam
- Bảo vệ server

---

## 🎯 Code mẫu hoàn chỉnh cho CartService.addItem

Kết hợp **Distributed Lock** + **Transaction**:

```typescript
async addItem(userId: string | null, deviceId: string | null, dto: AddToCartDto): Promise<CartSummary> {
  // 1. Xác định lock key
  const lockKey = `lock:cart:${userId ?? deviceId}`;
  const lockValue = randomBytes(16).toString('hex');
  const lockTtl = 10;

  // 2. Acquire lock
  const acquired = await this.redis.setNX(lockKey, lockValue, lockTtl);
  if (!acquired) {
    throw new ConflictException('Thao tác đang được xử lý, vui lòng thử lại sau');
  }

  try {
    // 3. Load + validate (ngoài transaction)
    const cart = await this.getOrCreateCart(userId, deviceId);
    const product = await this.productRepository.findOne({
      where: { id: dto.productId },
    });
    if (!product) throw new NotFoundException('Product not found');
    if (product.status !== ProductStatus.ACTIVE) {
      throw new BadRequestException('Product is not available');
    }

    const existingItem = await this.cartItemRepository.findOne({
      where: { cartId: cart.id, productId: dto.productId },
    });
    const newQuantity = (existingItem?.quantity ?? 0) + dto.quantity;
    if (newQuantity > product.stock) {
      throw new BadRequestException(`Insufficient stock. Available: ${product.stock}`);
    }

    const effectivePrice = product.salePrice ?? product.price;

    // 4. Transaction — chỉ writes
    await this.dataSource.transaction(async (manager) => {
      if (existingItem) {
        await manager.update(CartItem, existingItem.id, {
          quantity: newQuantity,
        });
      } else {
        await manager.insert(CartItem, {
          cartId: cart.id,
          productId: dto.productId,
          quantity: dto.quantity,
          priceSnapshot: effectivePrice,
        });
      }

      await manager.update(Cart, cart.id, { updatedAt: new Date() });
    });

    // 5. Sau commit
    await this.invalidateCache(cart.id);
    return this.getSummary(cart.id);
  } finally {
    // 6. Release lock (chỉ xoá nếu value khớp)
    await this.redis.releaseLock(lockKey, lockValue);
  }
}
```

**Luồng:**

```
User double-click
  ↓
Request 1: SET lock:cart:user-123 "abc" EX 10 NX → OK
Request 2: SET lock:cart:user-123 "xyz" EX 10 NX → FAIL
  ↓
Request 1: xử lý (validate + transaction)
Request 2: throw 409 Conflict → client hiển thị "Đang xử lý..."
  ↓
Request 1: releaseLock → OK
  ↓
User click lại → OK (lock đã release)
```

---

## 🎯 Trả lời trực tiếp

> **"Lock request khi người dùng double click gọi là gì?"**

**Trả lời:**

Có nhiều tên, tuỳ ngữ cảnh:

1. **Idempotency** (Idempotency Key) — nếu muốn **cùng request → cùng kết quả** (payment, order)

2. **Distributed Lock / Mutex** — nếu muốn **chặn request thứ 2** trong lúc request 1 đang xử lý (cart, stock)

3. **Request Deduplication** — tên chung cho việc chặn request trùng lặp

4. **Double-Submit Prevention** — ngữ cảnh form submission

5. **Debounce / Throttle** — nếu xử lý ở **frontend**

6. **Pessimistic Lock** — nếu xử lý ở **DB level** (SELECT ... FOR UPDATE)

7. **Optimistic Lock** — nếu dùng **version column** để detect conflict

**Trong thực tế:**
- **Payment/Order** → Idempotency Key
- **Cart/Stock** → Distributed Lock (Redis)
- **Frontend** → Disable button + Debounce
- **API** → Rate Limiting

**Code bạn đã có sẵn `RedisService.setNX` và `releaseLock`** — chỉ cần wrap vào method. 🎯