# Tài liệu Obsidian: Product, Cart & Payment Modules

> **Dự án:** E-Commerce Backend (NestJS + TypeORM + PostgreSQL + Redis)
> **Phạm vi:** 3 module `product`, `cart`, `payment`
> **Mục đích:** Giải thích luồng hoạt động và phân tích code

---

## 📑 Mục lục

- [[#1. Tổng quan kiến trúc]]
- [[#2. Module Product]]
- [[#3. Module Cart]]
- [[#4. Module Payment]]
- [[#5. Luồng end-to-end (User Journey)]]
- [[#6. Điểm mạnh & Rủi ro]]
- [[#7. Sơ đồ phụ thuộc giữa các module]]

---

## 1. Tổng quan kiến trúc

### 1.1. Sơ đồ layer

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT (HTTP Request)                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  Controller Layer (Product / Cart / Payment)                │
│  - @Throttle rate limit                                     │
│  - @UseGuards (JwtAuthGuard, RolesGuard, WebhookIpGuard)    │
│  - Validation Pipe (DTO)                                    │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│  Service Layer                                              │
│  - Business logic                                           │
│  - Cache invalidation (Redis)                               │
│  - Transaction (DataSource)                                 │
│  - Gateway dispatch (Payment)                               │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  PostgreSQL  │  │    Redis     │  │   Gateways   │
│  (TypeORM)   │  │  (Cache/Rate)│  │ VNPay / Momo │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 1.2. Vai trò từng module

| Module | Vai trò | Entity chính |
|---|---|---|
| **Product** | Quản lý danh mục sản phẩm, tồn kho, tìm kiếm | `Product` |
| **Cart** | Giỏ hàng cho cả guest (deviceId) và user (userId), merge khi login | `Cart`, `CartItem` |
| **Payment** | Tạo giao dịch, xử lý webhook từ VNPay/Momo, cập nhật trạng thái thanh toán | `PaymentTransaction` |

---

## 2. Module Product

### 2.1. Cấu trúc file

```
src/modules/product/
├── dto/
│   ├── create-product.dto.ts   # Validate tạo mới
│   ├── update-product.dto.ts   # PartialType(CreateProductDto)
│   └── query-product.dto.ts    # Filter, search, sort, pagination
├── product.controller.ts       # Public + Admin routes
├── product.entity.ts           # Bảng products
├── product.module.ts
└── product.service.ts          # Business logic
```

### 2.2. Entity `Product`

```typescript
@Entity('products')
@Index(['status', 'createdAt'])
@Index(['category'])
export class Product {
  id: uuid;
  name: string;              // 255
  slug: string;              // unique, SEO-friendly
  description: string | null;
  price: decimal(12,2);
  salePrice: decimal(12,2) | null;
  stock: int;                // default 0
  category: string;
  brand: string | null;
  images: string[];          // jsonb
  status: ProductStatus;     // draft | active | inactive | out_of_stock
  rating: decimal(3,2);
  reviewCount: int;
  createdAt, updatedAt;
}
```

**Index chiến lược:**
- `(status, createdAt)` → phục vụ list sản phẩm active mới nhất
- `(category)` → filter theo danh mục

### 2.3. Luồng hoạt động chính

#### 🔹 Tạo sản phẩm (Admin)

```
POST /api/products
  ├─ JwtAuthGuard → verify token
  ├─ RolesGuard   → role === 'admin'
  ├─ ValidationPipe → CreateProductDto
  └─ ProductService.create(dto)
       ├─ slugify(name)         # bỏ dấu tiếng Việt
       ├─ ensureUniqueSlug()    # thêm -1, -2 nếu trùng
       ├─ validate salePrice < price
       └─ save() → DB
```

**Hàm `slugify` xử lý tiếng Việt:**
```typescript
name.toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')  // bỏ dấu
    .replace(/đ/g, 'd')
    .replace(/[^a-z0-9\s-]/g, '')
    .replace(/\s+/g, '-')
```

#### 🔹 List sản phẩm (Public)

```
GET /api/products?search=&category=&minPrice=&maxPrice=&sortBy=&page=&limit=
  └─ ProductService.findAll(query)
       ├─ QueryBuilder với các điều kiện optional
       ├─ search: ILIKE trên name + description
       ├─ sort: whitelist ProductSortBy (chống SQL injection qua sortBy)
       ├─ pagination: skip/take
       └─ getManyAndCount() → { data, meta }
```

**Bảo mật:** `sortField` được whitelist qua `Object.values(ProductSortBy)` — tránh inject qua `ORDER BY`.

#### 🔹 Update sản phẩm

```
PATCH /api/products/:id
  └─ update(id, dto)
       ├─ findOne(id)
       ├─ validate salePrice mới < price mới
       ├─ nếu đổi name → slugify lại + ensureUniqueSlug(excludeId)
       └─ Object.assign + save
```

#### 🔹 Stock management (dùng cho Order sau này)

| Method | Mục đích | Side effect |
|---|---|---|
| `decreaseStock(id, qty)` | Trừ kho khi đặt hàng | Nếu stock = 0 → status = `OUT_OF_STOCK` |
| `increaseStock(id, qty)` | Hoàn kho khi hủy đơn | Nếu stock > 0 → status = `ACTIVE` |

### 2.4. Rate limiting

| Route | Limit |
|---|---|
| `GET /products` | 100/phút |
| `GET /products/slug/:slug` | 100/phút |
| `POST /products` | 30/phút |
| `PATCH /products/:id` | 30/phút |
| `DELETE /products/:id` | 20/phút |

---

## 3. Module Cart

### 3.1. Cấu trúc file

```
src/modules/cart/
├── dto/
│   ├── add-to-cart.dto.ts       # productId + quantity
│   ├── query-cart.dto.ts
│   └── update-cart-item.dto.ts  # quantity (0 = xóa)
├── cart.entity.ts               # Bảng carts
├── cart-item.entity.ts          # Bảng cart_items
├── cart.controller.ts
├── cart.module.ts
└── cart.service.ts
```

### 3.2. Entities

#### `Cart` — giỏ hàng

```typescript
@Entity('carts')
@Index(['userId'])
@Index(['deviceId'])
@Index(['status', 'updatedAt'])   // cho cronjob abandon cart sau này
export class Cart {
  id: uuid;
  userId: uuid | null;      // null nếu là guest
  deviceId: varchar(100) | null;
  status: CartStatus;       // active | checked_out | abandoned
  paymentStatus: CartPaymentStatus;  // unpaid | paid | refunded
  totalAmount: decimal;
  paidAt: Date | null;
  items: CartItem[];        // eager + cascade
  createdAt, updatedAt;
}
```

#### `CartItem` — item trong giỏ

```typescript
@Entity('cart_items')
@Unique(['cartId', 'productId'])   // 1 cart chỉ có 1 row/product
@Index(['cartId'])
export class CartItem {
  id: uuid;
  cartId: uuid;
  productId: uuid;
  product: Product;         // eager
  quantity: int;
  priceSnapshot: decimal;   // ⭐ snapshot giá lúc thêm vào giỏ
  createdAt, updatedAt;
}
```

> ⭐ **`priceSnapshot`** là điểm thiết kế quan trọng — giữ giá cũ để so sánh với giá hiện tại, phát hiện giá thay đổi.

### 3.3. Luồng hoạt động chính

#### 🔹 Xác định cart (guest vs user)

```
getOrCreateCart(userId, deviceId)
  ├─ Nếu có userId    → tìm cart có userId + status = ACTIVE
  ├─ Nếu chỉ deviceId → tìm cart có deviceId + userId IS NULL + ACTIVE
  └─ Không có → tạo mới
```

**Bảo vệ:** throw `BadRequestException` nếu cả userId và deviceId đều null.

#### 🔹 Thêm item vào giỏ

```
POST /api/carts/items  (body: { productId, quantity })
  └─ addItem(userId, deviceId, dto)
       ├─ getOrCreateCart()
       ├─ Tìm Product, validate status = ACTIVE
       ├─ Tìm CartItem có sẵn
       ├─ newQuantity = existing + dto.quantity
       ├─ if newQuantity > product.stock → throw
       ├─ effectivePrice = salePrice ?? price
       ├─ Nếu có item → update qty
       ├─ Nếu chưa → create với priceSnapshot
       ├─ Touch cart.updatedAt
       ├─ invalidateCache(cart.id)
       └─ getSummary(cart.id)
```

#### 🔹 Cập nhật item

```
PATCH /api/carts/items/:itemId  (body: { quantity })
  ├─ Nếu quantity === 0 → xóa item
  ├─ Validate tồn kho
  ├─ Cập nhật lại priceSnapshot (vì user chủ động thay đổi)
  └─ invalidateCache
```

#### 🔹 Summary với Redis cache

```
getSummary(cartId)
  ├─ Try Redis GET cart:summary:{cartId}
  │   └─ Hit → return luôn (TTL 5 phút)
  ├─ Load DB với relations { items: { product: true } }
  ├─ buildSummary(cart)
  │    ├─ Với mỗi item:
  │    │    ├─ currentPrice = salePrice ?? price
  │    │    ├─ snapshotPrice = priceSnapshot
  │    │    ├─ priceChanged = currentPrice !== snapshotPrice
  │    │    ├─ subtotalItem = currentPrice * quantity
  │    │    └─ discountItem = (price - salePrice) * quantity
  │    └─ Tổng: subtotal, discount, total, totalQuantity
  └─ Redis SETEX cart:summary:{cartId} 300s
```

**Kết quả `CartSummary`:**
```typescript
{
  id, items: [{
    id, productId, name, image, price, salePrice,
    quantity, subtotal, priceChanged,
    stockAvailable, inStock
  }],
  totalItems, totalQuantity,
  subtotal, discount, total
}
```

#### 🔹 Merge guest cart khi login

```
POST /api/carts/merge   (sau khi login)
  └─ mergeGuestCart(userId, deviceId)
       ├─ Tìm guestCart (deviceId, userId IS NULL, ACTIVE)
       ├─ Nếu không có → return user cart hiện tại
       ├─ getOrCreateCart(userId) → userCart
       ├─ Với mỗi guestItem:
       │    ├─ Nếu đã có trong userCart → cộng qty (cap tại stock)
       │    └─ Nếu chưa → chuyển cartId sang userCart
       ├─ Xóa guestCart
       └─ invalidateCache(userCart.id)
```

#### 🔹 Checkout hooks (dùng cho Order module)

| Method | Mục đích |
|---|---|
| `validateForCheckout(cartId)` | Trả `{ valid, errors[] }` — check rỗng, product inactive, thiếu stock |
| `markAsCheckedOut(cartId)` | Set `status = CHECKED_OUT`, `paymentStatus = UNPAID`, ghi `totalAmount` |

### 3.4. Rate limiting

| Route | Limit |
|---|---|
| `GET /carts/me` | 100/phút |
| `POST /carts/items` | 30/phút |
| `PATCH /carts/items/:id` | 30/phút |
| `DELETE /carts/items/:id` | 30/phút |
| `DELETE /carts/me` | 10/phút |
| `POST /carts/merge` | 5/phút |

---

## 4. Module Payment

### 4.1. Cấu trúc file

```
src/modules/payment/
├── constants/
│   └── payment.constant.ts     # Enums + TTL
├── dto/
│   └── create-payment.dto.ts
├── entities/
│   └── payment-transaction.entity.ts
├── gateways/
│   ├── momo.gateway.ts         # HMAC SHA256
│   └── vnpay.gateway.ts        # HMAC SHA512
├── guards/
│   └── webhook-ip.guard.ts     # IP whitelist
├── interfaces/
│   └── payment-gateway.interface.ts  # Strategy pattern
├── payment.controller.ts
├── payment.module.ts
└── payment.service.ts
```

### 4.2. Constants

```typescript
enum PaymentProvider { VNPAY, MOMO, COD }
enum PaymentStatus {
  PENDING, PROCESSING, SUCCESS, FAILED, REFUNDED, CANCELLED
}
const PAYMENT_CACHE_TTL = 300;              // 5 phút
const WEBHOOK_IDEMPOTENCY_TTL = 7 ngày;
```

### 4.3. Entity `PaymentTransaction`

```typescript
@Entity('payment_transactions')
@Index(['cartId'])
@Index(['provider', 'providerTransactionId'], { unique: true })
@Index(['status', 'createdAt'])
export class PaymentTransaction {
  id: uuid;
  cartId: uuid;
  userId: uuid;
  provider: PaymentProvider;
  providerTransactionId: string | null;
  amount: decimal(12,2);
  status: PaymentStatus;
  paymentUrl: string | null;
  providerResponse: jsonb;
  errorMessage: string | null;
  paidAt: Date | null;
  createdAt, updatedAt;
}
```

### 4.4. Strategy Pattern — Payment Gateway

```
interface PaymentGateway {
  createPayment(params): Promise<CreatePaymentResult>;
  verifyWebhook(data): WebhookVerifyResult;
  buildWebhookResponse(success, message): any;
}
       ▲                          ▲
       │                          │
  VnpayGateway               MomoGateway
```

**PaymentService** giữ `Map<PaymentProvider, PaymentGateway>` để dispatch:

```typescript
this.gateways = new Map([
  [PaymentProvider.VNPAY, this.vnpayGateway],
  [PaymentProvider.MOMO,  this.momoGateway],
]);
```

### 4.5. Luồng tạo thanh toán

```
POST /api/payments/create  (JwtAuthGuard)
  body: { cartId, provider }
  └─ createPayment(userId, dto, ip)
       ├─ 1. Verify cart: id + userId + status = CHECKED_OUT
       ├─ 2. if cart.paymentStatus === PAID → Conflict
       ├─ 3. Check existing pending tx có paymentUrl → return luôn
       ├─ 4. gateway = gateways.get(dto.provider)
       ├─ 5. Tạo PaymentTransaction (status = PENDING)
       ├─ 6. gateway.createPayment({ orderId: cartId, amount, ... })
       │    ├─ VNPay: build query string + HMAC SHA512
       │    └─ Momo:  build JSON + HMAC SHA256
       ├─ 7. Update transaction: paymentUrl, providerTxId, status = PROCESSING
       └─ 8. Return { paymentUrl, transactionId }
```

**Nếu gateway fail:**
```typescript
transaction.status = FAILED;
transaction.errorMessage = error.message;
await save();
throw error;
```

### 4.6. Luồng xử lý Webhook (phức tạp nhất)

```
GET /api/payments/vnpay/webhook   (WebhookIpGuard)
POST /api/payments/momo/webhook   (WebhookIpGuard)
  └─ handleWebhook(provider, data, ip)
```

#### Các bước xử lý:

```
BƯỚC 1: VERIFY SIGNATURE
  ├─ gateway.verifyWebhook(data)
  ├─ Nếu !isValid → log warn + return response(false, 'Invalid signature')
  └─ (Không throw — gateway cần response để không retry vô hạn)

BƯỚC 2: IDEMPOTENCY (chống xử lý trùng)
  ├─ key = `webhook:{provider}:{providerTransactionId}`
  ├─ redis.setNX(key, 'processing', 7 ngày)
  ├─ Nếu KHÔNG acquire được:
  │    ├─ get(key) === 'done' → return 'Already processed'
  │    └─ get(key) === 'processing' → return 'Processing'
  └─ Nếu acquire OK → tiếp tục

BƯỚC 3: FIND TRANSACTION
  ├─ Tìm tx theo { cartId: verifyResult.orderId, provider }
  ├─ Nếu không thấy → del key (release) + return false
  └─ (Release để gateway retry vì có thể race condition)

BƯỚC 4: VERIFY AMOUNT
  ├─ if transaction.amount !== verifyResult.amount → del key + return false
  └─ (Chống tấn công thay đổi số tiền)

BƯỚC 5: ATOMIC UPDATE (trong transaction)
  ├─ newStatus = isSuccess ? SUCCESS : FAILED
  ├─ UPDATE payment_transactions
  │    SET status = newStatus, paid_at, provider_response
  │    WHERE id = ? AND status IN (PENDING, PROCESSING)   ⭐
  ├─ Nếu affected === 0 → return { alreadyProcessed: true }
  ├─ Nếu isSuccess → UPDATE carts
  │    SET payment_status = 'paid', paid_at = now()
  │    WHERE id = ? AND payment_status = 'unpaid'          ⭐
  └─ return { alreadyProcessed: false }

BƯỚC 6: MARK DONE
  └─ redis.set(key, 'done', 7 ngày)

BƯỚC 7: LOG + RESPONSE
  └─ gateway.buildWebhookResponse(true, 'Confirm Success')
```

#### ⚠️ State Machine bảo vệ 2 lớp

| Lớp | Cơ chế | Mục đích |
|---|---|---|
| Redis `SET NX` | Atomic, TTL 7 ngày | Chặn request đồng thời |
| SQL `WHERE status IN (...)` | Atomic ở DB | Chặn update nếu status đã đổi |

> 💡 **Tại sao cần cả 2?** Nếu Redis mất data (restart, flush), SQL vẫn bảo vệ. Nếu SQL race condition, Redis đã chặn trước.

#### 🔹 Signature verification khác nhau giữa 2 gateway

**VNPay (SHA512):**
```typescript
1. Loại bỏ vnp_SecureHash, vnp_SecureHashType
2. Sort keys alphabet
3. signData = keys.map(k => `${k}=${encodeURIComponent(v).replace(/%20/g,'+')}`).join('&')
4. HMAC-SHA512(signData, hashSecret)
5. So sánh với vnp_SecureHash
6. Success khi vnp_ResponseCode === '00'
```

**Momo (SHA256):**
```typescript
1. Build rawSignature theo thứ tự CỐ ĐỊNH (accessKey, amount, extraData, ...)
2. HMAC-SHA256(rawSignature, secretKey)
3. So sánh với data.signature
4. Success khi resultCode === 0
```

### 4.7. WebhookIpGuard

```typescript
canActivate(context):
  ├─ provider = extractProvider(req.path)   # /api/payments/vnpay/webhook → 'vnpay'
  ├─ ip = normalizeIp(req.ip)               # bỏ ::ffff:
  ├─ whitelist = allowedIps[provider]
  ├─ Nếu whitelist rỗng → return true (dựa vào signature)
  └─ Nếu ip không có trong whitelist → throw ForbiddenException
```

**Lưu ý:** Đọc provider từ **URL path** (không phải `req.params`) vì webhook route là `@Get('vnpay/webhook')` không có param.

### 4.8. Rate limiting

| Route | Limit | Guard |
|---|---|---|
| `POST /payments/create` | 10/phút | JwtAuthGuard |
| `GET /payments/vnpay/webhook` | default 100/phút | WebhookIpGuard |
| `POST /payments/momo/webhook` | default 100/phút | WebhookIpGuard |
| `GET /payments/transactions/:id` | 60/phút | JwtAuthGuard |
| `GET /payments/carts/:cartId/transactions` | 60/phút | JwtAuthGuard |

---

## 5. Luồng end-to-end (User Journey)

```
┌──────────────────────────────────────────────────────────────────┐
│  1. GUEST duyệt sản phẩm                                         │
│     GET /api/products?category=...&sortBy=price                  │
│     → ProductService.findAll()                                   │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  2. GUEST thêm vào giỏ                                           │
│     POST /api/carts/items                                        │
│     Headers: X-Device-Id: abc12345...                            │
│     → CartService.addItem(null, deviceId, dto)                   │
│     → Tạo cart với userId=null, deviceId=abc12345                │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  3. USER đăng nhập                                               │
│     POST /api/auth/login  → nhận accessToken + cookie            │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  4. MERGE guest cart                                             │
│     POST /api/carts/merge  (JwtAuthGuard)                        │
│     → CartService.mergeGuestCart(userId, deviceId)               │
│     → Gộp items, xóa guest cart                                  │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  5. XEM giỏ hàng                                                 │
│     GET /api/carts/me  → CartSummary (có cache Redis 5 phút)     │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  6. CHECKOUT (Order module — chưa code)                          │
│     - CartService.validateForCheckout(cartId)                    │
│     - Tạo Order (snapshot items)                                 │
│     - ProductService.decreaseStock()                             │
│     - CartService.markAsCheckedOut(cartId)                       │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  7. TẠO THANH TOÁN                                               │
│     POST /api/payments/create                                    │
│     body: { cartId, provider: 'vnpay' }                          │
│     → PaymentService.createPayment(userId, dto, ip)              │
│     → Return { paymentUrl, transactionId }                       │
│     → FE redirect user sang paymentUrl                           │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│  8. USER thanh toán trên VNPay/Momo                              │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
        ┌──────────────────┴──────────────────┐
        ▼                                     ▼
┌─────────────────────────┐    ┌─────────────────────────────────┐
│ 9a. VNPay gọi webhook   │    │ 9b. USER redirect về return URL │
│ GET /payments/vnpay/    │    │ GET /payments/vnpay/return      │
│     webhook             │    │ (cùng xử lý như webhook)        │
│ → handleWebhook()       │    │                                 │
└───────────┬─────────────┘    └──────────────┬──────────────────┘
            │                                 │
            └────────────┬────────────────────┘
                         ▼
┌──────────────────────────────────────────────────────────────────┐
│ 10. UPDATE trạng thái                                            │
│     - PaymentTransaction: SUCCESS + paid_at                      │
│     - Cart: payment_status = 'paid', paid_at = now()             │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. Điểm mạnh & Rủi ro

### ✅ Điểm mạnh

| # | Điểm mạnh | Module |
|---|---|---|
| 1 | **Price snapshot** trong `CartItem` — phát hiện giá thay đổi | Cart |
| 2 | **Idempotency 2 lớp** (Redis NX + SQL WHERE status) | Payment |
| 3 | **Strategy pattern** cho gateway — dễ thêm provider mới | Payment |
| 4 | **Webhook IP whitelist** trước khi verify signature | Payment |
| 5 | **Merge guest cart** tự động khi login | Cart |
| 6 | **Slug SEO-friendly** + xử lý tiếng Việt | Product |
| 7 | **Whitelist sortBy** chống SQL injection | Product |
| 8 | **Redis cache** cho `CartSummary` giảm load DB | Cart |
| 9 | **Atomic SET NX EX** trong Redis service | Infra |
| 10 | Rate limiting theo user (không chỉ IP) | Global |

### ⚠️ Rủi ro & cải thiện

| # | Vấn đề | Đề xuất |
|---|---|---|
| 1 | `CartService.mergeGuestCart` không dùng transaction → có thể mất item nếu fail giữa chừng | Bọc trong `dataSource.transaction()` |
| 2 | `PaymentService.createPayment` không lock cart → 2 request tạo 2 transaction | Distributed lock qua Redis hoặc unique constraint |
| 3 | Webhook `handleWebhook` không verify IP trong service, chỉ dựa vào guard | Thêm check IP trong service để defense-in-depth |
| 4 | `ProductService.decreaseStock` không atomic → race condition khi nhiều order | Dùng `UPDATE ... SET stock = stock - X WHERE stock >= X` |
| 5 | Cache `CartSummary` không invalidate khi product thay đổi giá | Invalidate theo productId hoặc giảm TTL |
| 6 | `Cart.markAsCheckedOut` không clear `items` → rác DB | Cronjob cleanup sau N ngày |
| 7 | Không có `Order` entity → `Cart` đang gánh vai trò Order | Tách Order module riêng |
| 8 | Webhook Momo return `NO_CONTENT` nhưng service vẫn trả body | Kiểm tra lại — có thể Momo cần body |
| 9 | `PaymentService.handleWebhook` không log `providerResponse` khi fail | Thêm structured log |
| 10 | Không có refund flow | Thêm `POST /payments/:id/refund` |

### 🔒 Security checklist

- [x] HMAC signature verification (SHA256/SHA512)
- [x] IP whitelist cho webhook
- [x] Idempotency chống replay
- [x] Amount verification chống tampering
- [x] JWT guard cho route nhạy cảm
- [x] Role guard cho admin route
- [x] Rate limiting per-user
- [x] httpOnly cookie cho refresh token
- [ ] CSRF token (hiện dựa vào `sameSite: strict`)
- [ ] Encryption at rest cho `providerResponse`

---

## 7. Sơ đồ phụ thuộc giữa các module

```
                    ┌─────────────┐
                    │  AppModule  │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        ▼                  ▼                  ▼
   ┌─────────┐       ┌──────────┐       ┌──────────┐
   │ Product │       │   Cart   │       │ Payment  │
   └────┬────┘       └────┬─────┘       └────┬─────┘
        │                 │                   │
        │                 │ imports           │ imports
        │                 ▼                   ▼
        │            ┌─────────┐        ┌──────────┐
        │            │ Product │        │   Cart   │
        │            │ (repo)  │        │  (repo)  │
        │            └─────────┘        └──────────┘
        │
        ▼
   ┌──────────────────────────┐
   │  Infrastructure (global) │
   │  - RedisModule           │
   │  - MailModule            │
   │  - TypeOrmModule         │
   └──────────────────────────┘
```

### Dependency graph (TypeScript imports)

```
ProductModule ──────────────► (độc lập, chỉ dùng Product repo)

CartModule ────┬────────────► Product repo (validate sản phẩm)
               └────────────► RedisService (cache summary)

PaymentModule ─┬────────────► Cart repo (verify cart CHECKED_OUT)
               ├────────────► RedisService (idempotency)
               └────────────► VnpayGateway, MomoGateway
```

---

## 8. Biến môi trường liên quan

```bash
# Product
# (không có env riêng)

# Cart
# (không có env riêng — dùng Redis chung)

# Payment
VNPAY_TMN_CODE=YOUR_TMN_CODE
VNPAY_HASH_SECRET=YOUR_HASH_SECRET
VNPAY_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html
VNPAY_WHITELIST_IPS=   # comma-separated

MOMO_PARTNER_CODE=MOMO
MOMO_ACCESS_KEY=YOUR_ACCESS_KEY
MOMO_SECRET_KEY=YOUR_SECRET_KEY
MOMO_ENDPOINT=https://test-payment.momo.vn/v2/gateway/api/create
MOMO_REDIRECT_URL=http://localhost:3000/payment/return
MOMO_IPN_URL=http://localhost:8080/api/payments/momo/webhook
MOMO_WHITELIST_IPS=

APP_URL=                 # dùng build returnUrl cho gateway
```

---

## 9. Ghi chú khi mở rộng

### Thêm payment provider mới (vd: ZaloPay)

1. Tạo `zalopay.gateway.ts` implements `PaymentGateway`
2. Thêm `ZALOPAY` vào `PaymentProvider` enum
3. Đăng ký trong `PaymentModule.providers`
4. Thêm vào `Map` trong `PaymentService` constructor
5. Thêm env `ZALOPAY_*` + `ZALOPAY_WHITELIST_IPS`
6. Thêm routes `@Get('zalopay/webhook')` và `@Get('zalopay/return')`

### Thêm Order module

```typescript
OrderService.createFromCart(cartId, userId) {
  1. validateForCheckout(cartId)
  2. Tạo Order + OrderItems (snapshot từ cart items)
  3. decreaseStock() cho từng product
  4. markAsCheckedOut(cartId)
  5. Tạo payment intent (gọi PaymentService)
}
```

### Cronjob đề xuất

| Job | Schedule | Mục đích |
|---|---|---|
| Abandon cart | Mỗi giờ | `Cart` có `status=ACTIVE` + `updatedAt < now - 30d` → `ABANDONED` |
| Cleanup transactions | Hàng ngày | `PaymentTransaction` có `status=FAILED` + `createdAt < now - 90d` |
| Stock alert | Mỗi 6h | Product có `stock < threshold` → notify admin |

---

## 10. Tags

#nestjs #typeorm #postgresql #redis #ecommerce #product #cart #payment #vnpay #momo #webhook #idempotency #strategy-pattern #obsidian-doc

---

> 📌 **Ghi chú cuối:** Tài liệu này mô tả trạng thái codebase tại thời điểm đọc `repomix-output.xml`. Khi code thay đổi, cập nhật lại các mục tương ứng (đặc biệt là **§6 Rủi ro** và **§9 Ghi chú mở rộng**).