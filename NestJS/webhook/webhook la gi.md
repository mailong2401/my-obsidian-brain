# Tài liệu Obsidian: Webhook trong hệ thống Payment

> **Phạm vi:** Toàn bộ cơ chế webhook của module Payment (VNPay + Momo)
> **Liên quan:** [[Payment Module]] · [[Idempotency]] · [[Security]]
> **Mục đích:** Hiểu sâu cách hệ thống nhận, xác thực và xử lý webhook từ payment gateway

---

## 📑 Mục lục

- [[#1. Webhook là gì và tại sao cần]]
- [[#2. Kiến trúc tổng thể]]
- [[#3. Luồng webhook VNPay]]
- [[#4. Luồng webhook Momo]]
- [[#5. Bảo mật webhook (5 lớp)]]
- [[#6. Idempotency — chống xử lý trùng]]
- [[#7. State Machine & Race Condition]]
- [[#8. So sánh VNPay vs Momo]]
- [[#9. Testing & Debugging]]
- [[#10. Runbook vận hành]]
- [[#11. Anti-patterns cần tránh]]
- [[#12. Checklist triển khai provider mới]]

---

## 1. Webhook là gì và tại sao cần

### 1.1. Định nghĩa

**Webhook** = HTTP callback mà payment gateway (VNPay, Momo, Stripe...) gọi **ngược về server của bạn** để thông báo kết quả giao dịch.

```
User ──► Payment Gateway ──► User thanh toán
                │
                │ (bất đồng bộ, vài giây sau)
                ▼
        Webhook ──► Server của bạn
```

### 1.2. Tại sao không dùng polling?

| Cách | Vấn đề |
|---|---|
| **Polling** (`GET /query` mỗi 5s) | Tốn request, latency cao, gateway rate-limit |
| **Webhook** | Real-time, gateway chủ động push, ít request |

### 1.3. Hai loại callback trong hệ thống

Hệ thống này xử lý **2 loại** callback song song:

| Loại | Trigger | Mục đích | Endpoint |
|---|---|---|---|
| **Webhook (IPN)** | Gateway gọi server-to-server | Cập nhật DB chính thức | `/payments/{provider}/webhook` |
| **Return URL** | Browser user redirect về | Hiển thị kết quả cho user | `/payments/{provider}/return` |

> ⚠️ **Quan trọng:** Return URL **KHÔNG đáng tin** — user có thể tự gõ URL. Luôn xác thực signature. Hệ thống này xử lý **cả hai qua cùng 1 hàm** `handleWebhook` để đảm bảo idempotency.

---

## 2. Kiến trúc tổng thể

### 2.1. Sơ đồ component

```
                    ┌─────────────────────────────────┐
                    │      Payment Controller          │
                    │  @Get('vnpay/webhook')           │
                    │  @Post('momo/webhook')           │
                    │  @Get('vnpay/return')            │
                    │  @Get('momo/return')             │
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │      WebhookIpGuard              │
                    │  - Đọc provider từ URL path      │
                    │  - Check IP whitelist            │
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │      PaymentService              │
                    │      handleWebhook()             │
                    └────────────┬────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
┌───────────────┐      ┌─────────────────┐      ┌────────────────┐
│  Gateway      │      │  RedisService   │      │  DataSource    │
│  verifyWebhook│      │  (idempotency)  │      │  (transaction) │
└───────────────┘      └─────────────────┘      └────────────────┘
```

### 2.2. Files liên quan

```
src/modules/payment/
├── payment.controller.ts          # 4 webhook routes
├── payment.service.ts             # handleWebhook() — core logic
├── guards/
│   └── webhook-ip.guard.ts        # Lớp bảo vệ IP
├── gateways/
│   ├── vnpay.gateway.ts           # verifyWebhook() SHA512
│   └── momo.gateway.ts            # verifyWebhook() SHA256
└── interfaces/
    └── payment-gateway.interface.ts  # WebhookVerifyResult

src/infrastructure/redis/
└── redis.service.ts               # setNX() — atomic lock
```

---

## 3. Luồng webhook VNPay

### 3.1. Endpoint

```typescript
@Get('vnpay/webhook')
@UseGuards(WebhookIpGuard)
@HttpCode(HttpStatus.OK)
async vnpayWebhook(@Query() query, @Ip() ip) {
  return this.paymentService.handleWebhook(PaymentProvider.VNPAY, query, ip);
}

@Get('vnpay/return')
@HttpCode(HttpStatus.OK)
async vnpayReturn(@Query() query, @Ip() ip) {
  return this.paymentService.handleWebhook(PaymentProvider.VNPAY, query, ip);
}
```

**Đặc điểm VNPay:**
- Dùng **GET** (query string) cho cả webhook và return
- Params có prefix `vnp_`
- Return URL **cũng phải verify signature** (không tin browser)

### 3.2. Payload mẫu

```
GET /api/payments/vnpay/webhook
  ?vnp_Amount=10000000
  &vnp_BankCode=NCB
  &vnp_OrderInfo=Thanh+toan+don+hang+abc12345
  &vnp_ResponseCode=00
  &vnp_TmnCode=YOUR_TMN_CODE
  &vnp_TransactionNo=13123456
  &vnp_TxnRef=<cartId>
  &vnp_SecureHash=abc123...
  &vnp_SecureHashType=SHA512
```

### 3.3. Verify signature (SHA512)

```typescript
verifyWebhook(data) {
  const secureHash = data['vnp_SecureHash'];
  if (!secureHash) return this.invalidResult('Missing vnp_SecureHash');

  // 1. Copy params, bỏ hash fields
  const params = { ...data };
  delete params['vnp_SecureHash'];
  delete params['vnp_SecureHashType'];

  // 2. Sort keys alphabet
  const sortedKeys = Object.keys(params).sort();

  // 3. Build signData — encode value, space → '+'
  const signData = sortedKeys
    .map(k => `${k}=${encodeURIComponent(params[k]).replace(/%20/g, '+')}`)
    .join('&');

  // 4. HMAC-SHA512
  const hmac = createHmac('sha512', vnp_HashSecret);
  const signed = hmac.update(Buffer.from(signData, 'utf-8')).digest('hex');

  // 5. So sánh
  const isValid = secureHash === signed;
  const isSuccess = isValid && data['vnp_ResponseCode'] === '00';

  return {
    isValid, isSuccess,
    orderId: data['vnp_TxnRef'],
    providerTransactionId: data['vnp_TransactionNo'],
    amount: Number(data['vnp_Amount']) / 100,   // ⚠️ VNPay * 100
    responseCode: data['vnp_ResponseCode'],
    message: data['vnp_Message'] || '',
    rawData: data,
  };
}
```

### 3.4. Response format

```typescript
buildWebhookResponse(success, message) {
  return {
    RspCode: success ? '00' : '99',
    Message: message,
  };
}
```

| RspCode | Ý nghĩa | VNPay sẽ làm gì |
|---|---|---|
| `00` | Thành công / đã xử lý | Không retry |
| `99` | Lỗi / chưa xử lý | Retry theo schedule |

### 3.5. Diagram luồng VNPay

```
VNPay Server                              Backend
     │                                       │
     │  GET /vnpay/webhook?vnp_...           │
     ├──────────────────────────────────────►│
     │                                       │
     │                          ┌────────────▼─────────────┐
     │                          │ WebhookIpGuard           │
     │                          │ - Check IP whitelist     │
     │                          └────────────┬─────────────┘
     │                                       │
     │                          ┌────────────▼─────────────┐
     │                          │ handleWebhook()          │
     │                          │ 1. verifyWebhook SHA512  │
     │                          │ 2. SET NX idempotency    │
     │                          │ 3. Find transaction      │
     │                          │ 4. Verify amount         │
     │                          │ 5. DB transaction        │
     │                          └────────────┬─────────────┘
     │                                       │
     │  { RspCode: '00', Message: '...' }    │
     │◄──────────────────────────────────────┤
     │                                       │
```

---

## 4. Luồng webhook Momo

### 4.1. Endpoint

```typescript
@Post('momo/webhook')
@UseGuards(WebhookIpGuard)
@HttpCode(HttpStatus.NO_CONTENT)
async momoWebhook(@Body() body, @Ip() ip) {
  return this.paymentService.handleWebhook(PaymentProvider.MOMO, body, ip);
}

@Get('momo/return')
@HttpCode(HttpStatus.OK)
async momoReturn(@Query() query, @Ip() ip) {
  return this.paymentService.handleWebhook(PaymentProvider.MOMO, query, ip);
}
```

**Đặc điểm Momo:**
- Webhook (IPN) dùng **POST** với JSON body
- Return dùng **GET** với query
- Khác method → phải xử lý cả 2

### 4.2. Payload mẫu (IPN)

```json
{
  "partnerCode": "MOMO",
  "orderId": "cart-uuid-here",
  "requestId": "request-uuid",
  "amount": 100000,
  "orderInfo": "Thanh toan don hang abc12345",
  "orderType": "momo_wallet",
  "transId": 2345678901,
  "resultCode": 0,
  "message": "Successful.",
  "payType": "qr",
  "responseTime": 1698765432000,
  "extraData": "",
  "signature": "abc123..."
}
```

### 4.3. Verify signature (SHA256)

```typescript
verifyWebhook(data) {
  // 1. Build rawSignature theo THỨ TỰ CỐ ĐỊNH (KHÔNG sort)
  const rawSignature =
    `accessKey=${accessKey}` +
    `&amount=${data.amount}` +
    `&extraData=${data.extraData || ''}` +
    `&message=${data.message}` +
    `&orderId=${data.orderId}` +
    `&orderInfo=${data.orderInfo}` +
    `&orderType=${data.orderType}` +
    `&partnerCode=${data.partnerCode}` +
    `&payType=${data.payType}` +
    `&requestId=${data.requestId}` +
    `&responseTime=${data.responseTime}` +
    `&resultCode=${data.resultCode}` +
    `&transId=${data.transId}`;

  // 2. HMAC-SHA256
  const signature = createHmac('sha256', secretKey)
    .update(rawSignature).digest('hex');

  const isValid = signature === data.signature;
  const isSuccess = isValid && data.resultCode === 0;

  return {
    isValid, isSuccess,
    orderId: data.orderId,
    providerTransactionId: String(data.transId),
    amount: Number(data.amount),
    responseCode: String(data.resultCode),
    message: data.message || '',
    rawData: data,
  };
}
```

### 4.4. ⚠️ Khác biệt quan trọng với VNPay

| Điểm | VNPay | Momo |
|---|---|---|
| Method build signature | **Sort keys alphabet** | **Thứ tự cố định** |
| Thuật toán | SHA512 | SHA256 |
| Đơn vị amount | **× 100** | Đơn vị gốc |
| Success code | `vnp_ResponseCode === '00'` | `resultCode === 0` |

### 4.5. Momo retry behavior

Momo retry **7 lần** trong 24h nếu không nhận `2xx`. Vì vậy:
- `@HttpCode(HttpStatus.NO_CONTENT)` (204)
- Nếu lỗi xử lý → phải `throw` để Momo retry
- Nếu invalid signature → trả 204 (không retry vô ích)

---

## 5. Bảo mật webhook (5 lớp)

### Lớp 1: IP Whitelist (`WebhookIpGuard`)

```typescript
canActivate(context) {
  const req = context.switchToHttp().getRequest();
  const provider = this.extractProvider(req.path);  // 'vnpay' | 'momo'
  const ip = this.normalizeIp(req.ip);              // bỏ ::ffff:

  const whitelist = this.allowedIps[provider];
  if (!whitelist || whitelist.length === 0) {
    return true;  // Không config → dựa vào signature
  }

  if (!whitelist.includes(ip)) {
    this.logger.warn(`🚫 Webhook from unauthorized IP: ${ip}`);
    throw new ForbiddenException('IP not allowed');
  }
  return true;
}

private extractProvider(path: string): string {
  // /api/payments/vnpay/webhook → 'vnpay'
  const parts = path.split('/').filter(Boolean);
  const idx = parts.indexOf('payments');
  return idx !== -1 && parts[idx + 1] ? parts[idx + 1] : '';
}

private normalizeIp(ip: string): string {
  return ip.replace(/^::ffff:/, '');  // IPv4-mapped IPv6
}
```

**Env config:**
```bash
VNPAY_WHITELIST_IPS=113.160.92.0,113.160.93.0
MOMO_WHITELIST_IPS=118.69.0.0,118.69.1.0
```

### Lớp 2: HMAC Signature Verification

Chống attacker giả mạo payload. Xem §3.3 và §4.3.

**Nguyên tắc:**
- Dùng constant-time compare (lý tưởng) — hiện tại code dùng `===` (có thể timing attack)
- Secret key **KHÔNG BAO GIỜ** log ra
- Reject ngay nếu thiếu signature field

### Lớp 3: Amount Verification

```typescript
if (Number(transaction.amount) !== verifyResult.amount) {
  this.logger.error(`Amount mismatch: expected=${transaction.amount} got=${verifyResult.amount}`);
  await this.redis.del(idempotencyKey);
  return gateway.buildWebhookResponse(false, 'Invalid amount');
}
```

**Tại sao cần?** Attacker có thể replay webhook với amount nhỏ hơn.

### Lớp 4: Idempotency (Redis SET NX)

Xem §6.

### Lớp 5: State Machine (SQL WHERE)

Xem §7.

---

## 6. Idempotency — chống xử lý trùng

### 6.1. Vấn đề

Gateway có thể gọi webhook **nhiều lần** cho cùng 1 giao dịch:
- User refresh trang return URL
- Gateway retry vì timeout
- Network issue

Nếu không chống → user bị **trừ tiền 2 lần** hoặc **update DB 2 lần**.

### 6.2. Giải pháp: Redis `SET NX EX`

```typescript
const idempotencyKey = `webhook:${provider}:${verifyResult.providerTransactionId}`;

const acquired = await this.redis.setNX(
  idempotencyKey,
  'processing',
  WEBHOOK_IDEMPOTENCY_TTL,  // 7 ngày
);

if (!acquired) {
  const status = await this.redis.get(idempotencyKey);
  if (status === 'done') {
    return gateway.buildWebhookResponse(true, 'Already processed');
  }
  // status === 'processing' → request khác đang xử lý
  return gateway.buildWebhookResponse(true, 'Processing');
}
```

### 6.3. Redis command thực thi

```typescript
async setNX(key, value, ttlSeconds): Promise<boolean> {
  const result = await this.redis.set(key, value, 'EX', ttlSeconds, 'NX');
  return result === 'OK';
}
```

Redis `SET key value EX 604800 NX` là **atomic** — đảm bảo chỉ 1 client acquire được.

### 6.4. State machine của idempotency key

```
        ┌──────────────┐
        │   (không có) │
        └───────┬──────┘
                │ SET NX 'processing'
                ▼
        ┌──────────────┐
        │  processing  │◄──── request đến đồng thời → return 'Processing'
        └───────┬──────┘
                │ xử lý xong
                ▼
        ┌──────────────┐
        │     done     │◄──── request đến sau → return 'Already processed'
        └───────┬──────┘
                │ TTL 7 ngày hết hạn
                ▼
        ┌──────────────┐
        │   (không có) │
        └──────────────┘
```

### 6.5. Khi nào release key?

```typescript
// ❌ Transaction not found
await this.redis.del(idempotencyKey);
return gateway.buildWebhookResponse(false, 'Transaction not found');

// ❌ Amount mismatch
await this.redis.del(idempotencyKey);
return gateway.buildWebhookResponse(false, 'Invalid amount');

// ❌ Exception trong DB transaction
catch (error) {
  await this.redis.del(idempotencyKey);
  throw error;
}
```

**Nguyên tắc:** Chỉ `del` khi **chắc chắn chưa xử lý xong** — để gateway retry.

### 6.6. Tại sao TTL 7 ngày?

```typescript
export const WEBHOOK_IDEMPOTENCY_TTL = 7 * 24 * 60 * 60;
```

- Gateway thường retry trong 24-72h
- 7 ngày là buffer an toàn
- Sau 7 ngày, key hết hạn → có thể xử lý lại (nhưng transaction đã `SUCCESS` → SQL WHERE chặn)

---

## 7. State Machine & Race Condition

### 7.1. Vấn đề race condition

```
Timeline:
T1: Webhook A đến, SET NX OK → bắt đầu xử lý
T2: Webhook B đến, SET NX fail → return 'Processing' (đúng)
T3: Webhook A update DB xong → set 'done'
T4: Webhook C đến, get = 'done' → return 'Already processed' (đúng)
```

Nhưng nếu Redis bị **flush** giữa chừng?

```
T1: Webhook A, SET NX OK, bắt đầu xử lý
T2: Redis restart → mất key
T3: Webhook B, SET NX OK (vì key đã mất) → xử lý lần 2!
```

→ Cần lớp bảo vệ thứ 2 ở **DB level**.

### 7.2. State Machine ở SQL

```typescript
const updateResult = await manager
  .createQueryBuilder()
  .update(PaymentTransaction)
  .set({ status: newStatus, paidAt, providerResponse, ... })
  .where('id = :id', { id: transaction.id })
  .andWhere('status IN (:...statuses)', {
    statuses: [PaymentStatus.PENDING, PaymentStatus.PROCESSING],  // ⭐
  })
  .execute();

if (updateResult.affected === 0) {
  return { alreadyProcessed: true };  // Status đã đổi → không update nữa
}
```

**Điều kiện `WHERE status IN (PENDING, PROCESSING)`** đảm bảo:
- Chỉ update khi transaction **chưa** ở trạng thái cuối
- Nếu đã `SUCCESS`/`FAILED` → `affected = 0` → bỏ qua

### 7.3. State machine của `PaymentTransaction`

```
       ┌─────────┐
       │ PENDING │  (mới tạo)
       └────┬────┘
            │ createPayment thành công
            ▼
       ┌────────────┐
       │ PROCESSING │  (đã gọi gateway)
       └─────┬──────┘
             │
      ┌──────┼──────┐
      │             │
      ▼             ▼
 ┌─────────┐   ┌────────┐
 │ SUCCESS │   │ FAILED │  (webhook đến)
 └─────────┘   └────────┘
      │
      │ refund
      ▼
 ┌──────────┐
 │ REFUNDED │
 └──────────┘
```

**Terminal states:** `SUCCESS`, `FAILED`, `REFUNDED`, `CANCELLED` — không update nữa.

### 7.4. State machine của `Cart.paymentStatus`

```
       ┌────────┐
       │ UNPAID │
       └───┬────┘
           │ webhook SUCCESS
           ▼
       ┌──────┐
       │ PAID │
       └───┬──┘
           │ refund
           ▼
       ┌──────────┐
       │ REFUNDED │
       └──────────┘
```

Update có guard:
```typescript
.andWhere('payment_status = :status', { status: CartPaymentStatus.UNPAID })
```

### 7.5. Tổng kết bảo vệ 2 lớp

| Lớp | Cơ chế | Bảo vệ khi |
|---|---|---|
| **Redis NX** | Atomic lock + TTL | 2 request đồng thời |
| **SQL WHERE** | Conditional update | Redis mất data, race condition |

> 💡 **Nguyên tắc vàng:** *"Defense in depth"* — không bao giờ tin 1 lớp bảo vệ duy nhất.

---

## 8. So sánh VNPay vs Momo

### 8.1. Bảng so sánh chi tiết

| Tiêu chí | VNPay | Momo |
|---|---|---|
| **Webhook method** | GET | POST |
| **Return method** | GET | GET |
| **Payload format** | Query string | JSON body |
| **Signature algo** | HMAC-SHA512 | HMAC-SHA256 |
| **Signature order** | Sort alphabet | Thứ tự cố định |
| **Amount unit** | × 100 | Đơn vị gốc |
| **Success code** | `vnp_ResponseCode === '00'` | `resultCode === 0` |
| **Response format** | `{ RspCode, Message }` | `{ message }` |
| **Response HTTP** | 200 | 204 |
| **Retry policy** | N lần / N giờ | 7 lần / 24h |
| **Env vars** | `VNPAY_*` | `MOMO_*` |
| **Whitelist IP** | `VNPAY_WHITELIST_IPS` | `MOMO_WHITELIST_IPS` |

### 8.2. Signature build — điểm khác biệt lớn nhất

**VNPay (sort):**
```typescript
const sortedKeys = Object.keys(params).sort();
const signData = sortedKeys.map(k => `${k}=${encodeURIComponent(params[k])}`).join('&');
```

**Momo (cố định):**
```typescript
const rawSignature =
  `accessKey=${accessKey}` +
  `&amount=${data.amount}` +
  `&extraData=${data.extraData || ''}` +
  // ... theo đúng thứ tự Momo quy định
```

> ⚠️ **Sai thứ tự ở Momo = signature sai = webhook rejected**

### 8.3. Response format — tại sao khác?

**VNPay:**
```json
{ "RspCode": "00", "Message": "Confirm Success" }
```
VNPay đọc `RspCode` để biết thành công.

**Momo:**
```json
{ "message": "Confirm Success" }
```
Momo chỉ cần HTTP `204` để biết OK — body không quan trọng.

---

## 9. Testing & Debugging

### 9.1. Test local với Postman/Curl

#### VNPay webhook

```bash
# 1. Tạo transaction
curl -X POST http://localhost:8080/api/payments/create \
  -H "Authorization: Bearer <access_token>" \
  -H "Content-Type: application/json" \
  -d '{"cartId":"<cart-uuid>","provider":"vnpay"}'

# 2. Giả lập webhook với signature hợp lệ
# (cần script tính HMAC — xem test helper bên dưới)
curl "http://localhost:8080/api/payments/vnpay/webhook?\
vnp_Amount=10000000&\
vnp_ResponseCode=00&\
vnp_TxnRef=<cart-uuid>&\
vnp_TransactionNo=12345&\
vnp_SecureHash=<computed_hmac>"
```

#### Momo webhook

```bash
curl -X POST http://localhost:8080/api/payments/momo/webhook \
  -H "Content-Type: application/json" \
  -d '{
    "partnerCode": "MOMO",
    "orderId": "<cart-uuid>",
    "amount": 100000,
    "resultCode": 0,
    "transId": 12345,
    "signature": "<computed_hmac>"
  }'
```

### 9.2. Helper script tính signature (Node.js)

```typescript
// scripts/sign-vnpay.ts
import { createHmac } from 'crypto';

const HASH_SECRET = process.env.VNPAY_HASH_SECRET!;

function signVnpay(params: Record<string, string>): string {
  const sortedKeys = Object.keys(params).sort();
  const signData = sortedKeys
    .map(k => `${k}=${encodeURIComponent(params[k]).replace(/%20/g, '+')}`)
    .join('&');
  return createHmac('sha512', HASH_SECRET)
    .update(Buffer.from(signData, 'utf-8'))
    .digest('hex');
}

// Usage
const params = {
  vnp_Amount: '10000000',
  vnp_ResponseCode: '00',
  vnp_TxnRef: 'cart-uuid',
  vnp_TransactionNo: '12345',
  // ...
};
console.log(signVnpay(params));
```

### 9.3. Debug checklist

Khi webhook không hoạt động, kiểm tra theo thứ tự:

```
1. ✅ Webhook có đến server không?
   → Check access log / NestJS log: "VNPay webhook received from {ip}"

2. ✅ WebhookIpGuard có pass không?
   → Nếu 403: IP không nằm trong whitelist
   → Fix: thêm IP vào VNPAY_WHITELIST_IPS hoặc để rỗng

3. ✅ Signature có hợp lệ không?
   → Check log: "⚠️ Invalid {provider} signature"
   → Debug: in rawSignature + signed + expected
   → Common issues:
     - Sai secret key
     - Sai thứ tự (Momo)
     - Quên encode URL
     - Sai đơn vị amount (VNPay × 100)

4. ✅ Transaction có tồn tại không?
   → Check log: "Transaction not found: cart=..."
   → Verify: orderId mapping đúng chưa?

5. ✅ Amount có khớp không?
   → Check log: "Amount mismatch: expected=X got=Y"
   → Fix: verify lại số tiền ở 2 phía

6. ✅ DB update có affected > 0 không?
   → Nếu affected = 0 → status đã ở terminal state
   → Check DB: SELECT status FROM payment_transactions WHERE id = '...'

7. ✅ Idempotency key có bị stuck 'processing' không?
   → redis-cli GET webhook:vnpay:12345
   → Nếu stuck > 5 phút → có thể có exception, check log
```

### 9.4. Log quan trọng cần theo dõi

```typescript
// ✅ Success
this.logger.log(`✅ Webhook processed: ${provider} cart=${cartId} status=${newStatus}`, {
  transactionId, providerTxId, amount, ip,
});

// ⚠️ Invalid signature
this.logger.warn(`⚠️ Invalid ${provider} signature from IP ${ipAddress}`, { orderId });

// ❌ Transaction not found
this.logger.error(`Transaction not found: cart=${orderId} provider=${provider}`);

// ❌ Amount mismatch
this.logger.error(`Amount mismatch: expected=${transaction.amount} got=${verifyResult.amount}`);

// 🔁 Already processed
this.logger.log(`Webhook already processed: ${provider} tx=${providerTransactionId}`);

// 💥 Processing failed
this.logger.error(`❌ Webhook processing failed: ${provider} cart=${cartId}`, error);
```

### 9.5. Redis commands hữu ích

```bash
# Xem idempotency key
redis-cli -a redis123 GET "webhook:vnpay:13123456"
# → 'processing' hoặc 'done'

# Xem TTL còn lại
redis-cli -a redis123 TTL "webhook:vnpay:13123456"
# → số giây

# Xóa key để retry (chỉ dùng khi debug)
redis-cli -a redis123 DEL "webhook:vnpay:13123456"

# Scan tất cả webhook keys
redis-cli -a redis123 --scan --pattern "webhook:*"
```

---

## 10. Runbook vận hành

### 10.1. Sự cố: "Webhook không đến"

**Triệu chứng:** User thanh toán thành công nhưng cart vẫn `unpaid`.

**Điều tra:**
```
1. Check gateway dashboard → transaction có success không?
2. Check webhook URL config trên gateway dashboard
   → VNPay: Merchant Portal → IPN URL
   → Momo:  Đối tác portal → IPN URL
3. Check firewall / reverse proxy → webhook route có bị chặn?
4. Check server log → có request nào từ gateway IP không?
```

**Fix tạm:**
```bash
# Query gateway API để lấy kết quả
GET /api/payments/{provider}/query?vnp_TxnRef=<cartId>
# (cần implement thêm)
```

### 10.2. Sự cố: "Signature invalid"

**Nguyên nhân thường gặp:**
| # | Nguyên nhân | Fix |
|---|---|---|
| 1 | Sai secret key | Verify env `VNPAY_HASH_SECRET` / `MOMO_SECRET_KEY` |
| 2 | Sai thứ tự (Momo) | Đối chiếu với docs Momo |
| 3 | Thiếu field trong signature | Đảm bảo có tất cả field bắt buộc |
| 4 | Encode URL sai | Dùng `encodeURIComponent` + replace `%20` → `+` |
| 5 | Sai đơn vị amount | VNPay dùng × 100 |

### 10.3. Sự cố: "Webhook processed nhưng DB không update"

**Điều tra:**
```sql
-- Check transaction status
SELECT id, cart_id, status, paid_at, provider_response
FROM payment_transactions
WHERE cart_id = '<cartId>'
ORDER BY created_at DESC;

-- Check cart status
SELECT id, status, payment_status, paid_at
FROM carts WHERE id = '<cartId>';
```

**Nếu transaction `SUCCESS` nhưng cart `UNPAID`:**
- Có thể update cart bị fail trong transaction → check log
- Fix: manual update hoặc thêm job reconciliation

### 10.4. Sự cố: "Idempotency key stuck"

**Triệu chứng:** Webhook retry nhưng luôn nhận `Processing`.

**Điều tra:**
```bash
redis-cli -a redis123 GET "webhook:vnpay:<txId>"
# → 'processing'

# Check TTL
redis-cli -a redis123 TTL "webhook:vnpay:<txId>"
# → còn nhiều giây (chưa hết hạn)
```

**Nguyên nhân:** Request đầu tiên crash giữa chừng, không `del` key.

**Fix:**
```bash
# Release thủ công
redis-cli -a redis123 DEL "webhook:vnpay:<txId>"

# Sau đó gateway retry sẽ xử lý lại
```

**Fix lâu dài:** Thêm cronjob release key có `processing` > 5 phút.

### 10.5. Reconciliation job (đề xuất)

```typescript
@Cron('0 */6 * * *')  // Mỗi 6h
async reconcilePayments() {
  // 1. Tìm transaction PENDING/PROCESSING > 30 phút
  const stuck = await this.paymentRepo.find({
    where: {
      status: In([PaymentStatus.PENDING, PaymentStatus.PROCESSING]),
      createdAt: LessThan(new Date(Date.now() - 30 * 60 * 1000)),
    },
  });

  // 2. Với mỗi transaction → query gateway API
  for (const tx of stuck) {
    const gateway = this.gateways.get(tx.provider);
    // const result = await gateway.queryTransaction(tx.providerTransactionId);
    // Update status nếu cần
  }
}
```

---

## 11. Anti-patterns cần tránh

### ❌ 1. Tin tưởng return URL

```typescript
// ❌ SAI — user có thể tự gõ URL
@Get('return')
async handleReturn(@Query('status') status: string) {
  if (status === 'success') {
    await this.markAsPaid();  // BUG: ai cũng gọi được
  }
}
```

```typescript
// ✅ ĐÚNG — verify signature
@Get('return')
async handleReturn(@Query() query) {
  return this.paymentService.handleWebhook(PaymentProvider.VNPAY, query, ip);
  // → verify signature bên trong
}
```

### ❌ 2. Không verify amount

```typescript
// ❌ SAI
if (verifyResult.isSuccess) {
  await this.markAsPaid(transaction.id);
}
```

```typescript
// ✅ ĐÚNG
if (verifyResult.isSuccess && Number(transaction.amount) === verifyResult.amount) {
  await this.markAsPaid(transaction.id);
}
```

### ❌ 3. Update không guard status

```typescript
// ❌ SAI — có thể overwrite SUCCESS thành FAILED
await this.paymentRepo.update(id, { status: newStatus });
```

```typescript
// ✅ ĐÚNG — conditional update
await manager.createQueryBuilder()
  .update(PaymentTransaction)
  .set({ status: newStatus })
  .where('id = :id', { id })
  .andWhere('status IN (:...statuses)', { statuses: [PENDING, PROCESSING] })
  .execute();
```

### ❌ 4. Throw khi invalid signature

```typescript
// ❌ SAI — gateway sẽ retry vô ích
if (!isValid) {
  throw new UnauthorizedException('Invalid signature');
}
```

```typescript
// ✅ ĐÚNG — trả response để gateway không retry
if (!isValid) {
  return gateway.buildWebhookResponse(false, 'Invalid signature');
}
```

### ❌ 5. Đồng bộ xử lý nặng

```typescript
// ❌ SAI — webhook phải trả nhanh (<5s)
async handleWebhook(data) {
  await this.sendEmail();
  await this.updateInventory();
  await this.notifyShipping();
  await this.generateInvoice();
  return { RspCode: '00' };
}
```

```typescript
// ✅ ĐÚNG — đẩy vào queue
async handleWebhook(data) {
  await this.markAsPaid();
  await this.queue.add('post-payment', { transactionId });  // async
  return { RspCode: '00' };
}
```

### ❌ 6. Log secret key

```typescript
// ❌ SAI
this.logger.debug(`Signature: ${rawSignature}, secret: ${secretKey}`);
```

```typescript
// ✅ ĐÚNG
this.logger.debug(`Signature verification failed for order ${orderId}`);
```

### ❌ 7. Dùng `==` thay vì constant-time compare

```typescript
// ⚠️ CÓ THỂ BỊ TIMING ATTACK
const isValid = secureHash === signed;
```

```typescript
// ✅ AN TOÀN HƠN
import { timingSafeEqual } from 'crypto';
const isValid = timingSafeEqual(
  Buffer.from(secureHash, 'hex'),
  Buffer.from(signed, 'hex'),
);
```

---

## 12. Checklist triển khai provider mới

Khi thêm provider (vd: ZaloPay, Stripe):

### Code

- [ ] Tạo `zalopay.gateway.ts` implements `PaymentGateway`
  - [ ] `createPayment()` — build request + signature
  - [ ] `verifyWebhook()` — verify signature
  - [ ] `buildWebhookResponse()` — format response
- [ ] Thêm `ZALOPAY` vào `PaymentProvider` enum
- [ ] Đăng ký trong `PaymentModule.providers`
- [ ] Thêm vào `Map` trong `PaymentService` constructor
- [ ] Thêm routes trong `PaymentController`:
  - [ ] `@Post('zalopay/webhook')` hoặc `@Get` tùy provider
  - [ ] `@Get('zalopay/return')`
- [ ] Thêm `WebhookIpGuard` cho webhook route
- [ ] Thêm case trong `WebhookIpGuard.allowedIps`

### Config

- [ ] Thêm env vars: `ZALOPAY_*`
- [ ] Thêm `ZALOPAY_WHITELIST_IPS` vào `.env.example`
- [ ] Document trong README

### Testing

- [ ] Unit test cho `verifyWebhook()`
- [ ] Integration test cho `handleWebhook()`:
  - [ ] Valid signature → SUCCESS
  - [ ] Invalid signature → rejected
  - [ ] Duplicate webhook → idempotent
  - [ ] Amount mismatch → rejected
  - [ ] Transaction not found → released lock
- [ ] Test với sandbox của provider

### Ops

- [ ] Setup webhook URL trên provider dashboard
- [ ] Verify IP whitelist với provider docs
- [ ] Setup alert khi webhook fail rate > X%
- [ ] Thêm vào reconciliation job

### Docs

- [ ] Thêm section vào tài liệu này
- [ ] Update bảng so sánh §8
- [ ] Thêm debug checklist cho provider mới

---

## 13. Tóm tắt 1 trang (TL;DR)

```
┌─────────────────────────────────────────────────────────────────┐
│                    WEBHOOK FLOW — TL;DR                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Gateway gọi webhook ──► WebhookIpGuard (check IP)           │
│                                                                  │
│  2. PaymentService.handleWebhook()                              │
│     │                                                            │
│     ├─ 3. gateway.verifyWebhook() → check HMAC signature        │
│     │     └─ Invalid → return response (KHÔNG throw)            │
│     │                                                            │
│     ├─ 4. redis.setNX(idempotencyKey) → chống trùng             │
│     │     └─ Fail → return 'Already processed' / 'Processing'   │
│     │                                                            │
│     ├─ 5. Find transaction → không có → del key + return        │
│     │                                                            │
│     ├─ 6. Verify amount → mismatch → del key + return           │
│     │                                                            │
│     ├─ 7. DB transaction:                                       │
│     │     ├─ UPDATE payment_transactions                        │
│     │     │   WHERE id = ? AND status IN (PENDING, PROCESSING)  │
│     │     └─ UPDATE carts                                       │
│     │         WHERE id = ? AND payment_status = 'unpaid'        │
│     │                                                            │
│     ├─ 8. redis.set(key, 'done', 7d)                            │
│     │                                                            │
│     └─ 9. return gateway.buildWebhookResponse(true, 'Success')  │
│                                                                  │
│  🛡️ 3 LỚP BẢO VỆ:                                               │
│     L1: Signature verification (HMAC)                           │
│     L2: Redis idempotency (SET NX EX)                           │
│     L3: SQL conditional update (WHERE status IN ...)            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 14. Tags

#webhook #payment #vnpay #momo #idempotency #security #hmac #signature #race-condition #state-machine #nestjs #redis #postgresql #reconciliation #obsidian-doc

---

> 📌 **Ghi chú cuối:**
> - Tài liệu này mô tả webhook flow tại thời điểm đọc `repomix-output.xml`
> - Khi thêm provider mới → cập nhật §8, §10, §12
> - Khi gặp sự cố production → ghi lại vào §10 (Runbook)
> - **Nguyên tắc vàng:** *Không bao giờ tin webhook — luôn verify signature + amount + idempotency*