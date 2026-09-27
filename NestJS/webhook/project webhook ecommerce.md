Mình sẽ giải thích đúng theo code `PaymentService` của bạn. Đây là một flow webhook khá chuẩn cho hệ thống thanh toán production.

Tổng quan:

```
User
 |
 | 1. Click thanh toán
 |
 v
Frontend
 |
 | POST /api/payments/create
 |
 v
PaymentService.createPayment()
 |
 | tạo transaction PENDING
 |
 v
VNPay/Momo
 |
 | User thanh toán
 |
 | Gateway xử lý
 |
 | gọi ngược
 |
 v
POST /api/payments/webhook/vnpay
 |
 v
PaymentService.handleWebhook()
 |
 | verify
 | chống gọi trùng
 | update DB
 |
 v
Cart = PAID
PaymentTransaction = SUCCESS
```

---

# 1. Khi tạo thanh toán

Đầu tiên user gọi:

```
POST /api/payments/create

{
 cartId: "abc",
 provider: "VNPAY"
}
```

Đi vào:

```ts
createPayment()
```

---

## Bước 1: Kiểm tra cart

```ts
const cart = await this.cartRepo.findOne({
 where:{
   id:dto.cartId,
   userId,
   status: CartStatus.CHECKED_OUT
 }
});
```

Ý nghĩa:

Không cho thanh toán:

- cart không tồn tại
    
- cart của user khác
    
- cart chưa checkout
    

Ví dụ:

Database:

### cart

|id|user|status|payment|
|---|---|---|---|
|abc|Long|CHECKED_OUT|UNPAID|

OK.

---

# 2. Tạo PaymentTransaction

Code:

```ts
const transaction = this.paymentRepo.create({
 cartId: cart.id,
 userId,
 provider:dto.provider,
 amount:Number(cart.totalAmount),
 status:PaymentStatus.PENDING
});
```

Lúc này DB:

## payment_transaction

```
id: tx001
cart_id: abc
amount:500000
provider:VNPAY
status:PENDING
```

Chưa có tiền.

Chỉ là:

> "Tôi đang chờ thanh toán"

---

# 3. Gọi API VNPay/Momo

```ts
const result = await gateway.createPayment()
```

Ví dụ VNPay trả:

```json
{
 paymentUrl:
 "https://vnpay.vn/pay?id=123",

 providerTransactionId:
 "VNP123"
}
```

Update:

```ts
transaction.paymentUrl=result.paymentUrl;

transaction.providerTransactionId="VNP123";

transaction.status=PROCESSING;
```

Database:

```
payment_transaction

id       tx001
cart     abc
amount   500000
status   PROCESSING
```

---

# 4. User thanh toán

User mở:

```
https://vnpay.vn/pay?id=123
```

Nhập:

```
ATM
OTP
```

VNPay xử lý.

Sau đó có 2 việc:

---

## Return URL

Dành cho user:

```
VNPay
 |
 |
 v
https://your-app.com/payment/success
```

Cái này chỉ để frontend hiển thị:

```
Thanh toán thành công
```

Không dùng để update DB.

Vì user có thể:

- đóng tab
    
- fake request
    
- sửa URL
    

---

## Webhook

Dành cho server:

```
VNPay
 |
 |
 POST
 |
 v
/api/payment/webhook/vnpay
```

Đây mới là nguồn sự thật.

---

# 5. Webhook chạy

Controller có thể kiểu:

```ts
@Post('/webhook/:provider')
webhook(
 @Param('provider') provider,
 @Body() data,
 @Ip() ip
){
 return paymentService.handleWebhook(
   provider,
   data,
   ip
 )
}
```

---

# 6. Verify chữ ký

Đầu tiên:

```ts
const verifyResult =
 gateway.verifyWebhook(data);
```

Ví dụ VNPay gửi:

```json
{
 orderId:"abc",
 amount:500000,
 transactionId:"VNP123",
 signature:"xxxxx"
}
```

Server tự tính:

```
hash(secret + data)
```

So sánh:

```
VNPay signature
        |
        |
        v
Server generate signature
```

Nếu khác:

```
STOP
```

Vì có thể hacker tự POST:

```
POST /webhook

{
status:"SUCCESS"
}
```

---

# 7. Idempotency (phần rất quan trọng)

Webhook có thể bị gọi nhiều lần.

Ví dụ:

VNPay:

```
POST webhook
       |
       v
server timeout

VNPay nghĩ:
"chưa nhận được"

gửi lại:

POST webhook
```

Bạn sẽ nhận:

```
webhook #1

webhook #2

webhook #3
```

Nếu không xử lý:

```
Cart PAID
Cart PAID
Cart PAID
```

Có thể lỗi.

---

Code của bạn:

```ts
const idempotencyKey =
`webhook:${provider}:${verifyResult.providerTransactionId}`;
```

Ví dụ:

```
webhook:VNPAY:VNP123
```

Redis:

```
webhook:VNPAY:VNP123 = processing
```

---

Bạn dùng:

```ts
setNX()
```

NX nghĩa là:

> Chỉ tạo nếu chưa tồn tại.

Request 1:

```
setNX(key)

OK
```

Request 2:

```
setNX(key)

FAIL
```

=> chỉ một request được xử lý.

---

# 8. Tìm transaction

```ts
const transaction =
 await paymentRepo.findOne({
 where:{
  cartId:verifyResult.orderId,
  provider
 }
});
```

Ví dụ:

Webhook:

```
cartId=abc
provider=VNPAY
```

Tìm:

```
payment_transaction

tx001
cart abc
```

---

# 9. Check số tiền

Cực kỳ quan trọng:

```ts
if(transaction.amount !== verifyResult.amount)
```

Ví dụ hacker sửa:

```
VNPay gửi:

amount=10000
```

nhưng DB:

```
500000
```

Reject.

---

# 10. Update transaction bằng transaction DB

Đây là phần mạnh nhất.

```ts
this.dataSource.transaction()
```

Nó đảm bảo:

Hoặc:

```
Payment SUCCESS
+
Cart PAID
```

cùng thành công

Hoặc:

```
rollback toàn bộ
```

---

## Update payment

```ts
.update(PaymentTransaction)
.set({
 status:SUCCESS
})
```

Trước:

```
payment_transaction

PROCESSING
```

Sau:

```
SUCCESS
```

---

## Update cart

```ts
.update(Cart)
.set({
 paymentStatus:PAID
})
```

Trước:

```
cart

UNPAID
```

Sau:

```
PAID
```

---

# 11. State machine

Bạn đang làm đúng kiểu:

```
PENDING
   |
   |
PROCESSING
   |
   |
 SUCCESS


hoặc


PROCESSING
   |
   |
 FAILED
```

Không cho:

```
SUCCESS -> PENDING
```

Vì code:

```ts
.andWhere(
'status IN (:...statuses)',
{
statuses:[
 PENDING,
 PROCESSING
]
}
)
```

Nghĩa là:

Chỉ update nếu đang ở trạng thái hợp lệ.

---

# 12. Kết thúc webhook

Redis:

```ts
await this.redis.set(
 idempotencyKey,
 'done'
)
```

Ví dụ:

```
webhook:VNPAY:VNP123 = done
```

Nếu VNPay gửi lại:

```
Webhook already processed
```

---

# Tổng flow thực tế

```
                CREATE PAYMENT

Frontend
   |
   |
   v
PaymentService
   |
   |
Create PaymentTransaction
(PENDING)
   |
   |
Call VNPay
   |
   |
Return paymentUrl
   |
   |
User pays
   |
   |
VNPay
   |
   |
Webhook
   |
   |
verify signature
   |
   |
Redis SETNX
   |
   |
Find transaction
   |
   |
Check amount
   |
   |
DB transaction
   |
   +----------------+
   |                |
Payment SUCCESS   Cart PAID
   |
   |
Redis DONE
   |
   |
200 OK
```

---

Một điểm mình đánh giá cao trong code của bạn: bạn đã có đủ 5 lớp bảo vệ thường thấy ở payment production:

1. ✅ Signature verification
    
2. ✅ Idempotency bằng Redis SETNX
    
3. ✅ Amount verification
    
4. ✅ Database transaction
    
5. ✅ State machine chống update sai trạng thái
    

Đây gần với cách các hệ thống thanh toán thật (Stripe, PayPal, VNPay, Momo) xử lý.