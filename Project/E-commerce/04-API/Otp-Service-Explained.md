---
tags: [api, auth, otp, service, explained]
file: src/modules/auth/services/otp.service.ts
---

# OTP Service — Giải thích chi tiết

> **Mục đích**: Quản lý vòng đời OTP (One-Time Password) — sinh, gửi, verify, chống spam, chống brute force. Đây là **security-critical service** — bug ở đây có thể dẫn tới bypass auth.

## 📦 Import & Code đầy đủ

```ts
import { Injectable, BadRequestException, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { randomInt, randomBytes } from 'crypto';
import { RedisService } from 'src/infrastructure/redis/redis.service';
import { MailService } from 'src/infrastructure/mail/mail.service';
import { TooManyRequestsException } from 'src/common/exceptions/too-many-requests.exception';
import {
  RedisKeys,
  OtpPurposeType,
} from 'src/common/constants/redis-keys.constant';
import { OtpPayload } from 'src/common/interfaces/otp-payload.interface';
```

### Giải thích import

| Import | Vai trò |
|---|---|
| `Injectable` | Đánh dấu provider cho DI |
| `BadRequestException` | 400 — OTP sai/hết hạn/quá attempts |
| `Logger` | Log nội bộ |
| `ConfigService` | Đọc env: `OTP_LENGTH`, `OTP_TTL_SECONDS`, `OTP_MAX_ATTEMPTS`, `OTP_RESEND_COOLDOWN` |
| `randomInt` | ⭐ **Crypto-secure** random — KHÔNG dùng `Math.random()` |
| `randomBytes` | ⚠️ **Không dùng trong file này** — có thể là import thừa (leftover) |
| `RedisService` | Wrapper Redis (get/set/setJson/incr...) |
| `MailService` | Gửi email OTP |
| `TooManyRequestsException` | 429 — cooldown |
| `RedisKeys` | Helper build key pattern |
| `OtpPurposeType` | Union type: `'register' \| 'forgot_password' \| 'login_2fa'` |
| `OtpPayload` | Shape của payload OTP lưu Redis |

> ⚠️ **Phát hiện**: `randomBytes` được import nhưng **không dùng** trong file này → dead import. Nên xóa.

---

## 🏛️ Class Definition

```ts
@Injectable()
export class OtpService {
  private readonly logger = new Logger(OtpService.name);

  private readonly OTP_LENGTH: number;
  private readonly OTP_TTL: number;
  private readonly MAX_ATTEMPTS: number;
  private readonly RESEND_COOLDOWN: number;

  constructor(
    private readonly redis: RedisService,
    private readonly mailService: MailService,
    private readonly config: ConfigService,
  ) {
    this.OTP_LENGTH = this.config.get<number>('OTP_LENGTH', 6);
    this.OTP_TTL = this.config.get<number>('OTP_TTL_SECONDS', 300);
    this.MAX_ATTEMPTS = this.config.get<number>('OTP_MAX_ATTEMPTS', 5);
    this.RESEND_COOLDOWN = this.config.get<number>('OTP_RESEND_COOLDOWN', 60);
  }
}
```

### Giải thích

| Field | Default | Ý nghĩa |
|---|---|---|
| `OTP_LENGTH` | 6 | Độ dài OTP (số chữ số) |
| `OTP_TTL` | 300 (5 phút) | Thời gian sống của OTP (giây) |
| `MAX_ATTEMPTS` | 5 | Số lần nhập sai tối đa trước khi OTP bị hủy |
| `RESEND_COOLDOWN` | 60 | Thời gian chờ giữa 2 lần gửi OTP (giây) |

### 🎯 Tại sao đọc config trong constructor?

**3 lợi ích**:
1. **Fail fast**: Nếu env sai → crash ngay khi khởi động, không phải đợi request đầu tiên
2. **Cache**: Mỗi request không cần gọi `configService.get()` lại → nhanh hơn
3. **Type safety**: Field có type `number` rõ ràng

### ⚠️ Vấn đề tiềm ẩn

**`ConfigService.get<number>()` không tự convert!**

Nếu env là `OTP_LENGTH=6` (string), `configService.get<number>('OTP_LENGTH', 6)` sẽ trả về **string `'6'`** (không phải `number 6`) vì:
- `ConfigService` đọc từ `process.env` → tất cả đều là string
- Generic `<number>` chỉ là **type assertion**, không phải runtime cast

**Hệ quả**:
```ts
this.OTP_LENGTH = '6';           // string
Math.pow(10, this.OTP_LENGTH - 1); // '6' - 1 = 5 (JS auto-convert) → OK
// Nhưng nếu dùng .toFixed() hoặc so sánh strict → bug
```

**Fix**:
```ts
this.OTP_LENGTH = Number(this.config.get('OTP_LENGTH', 6));
// hoặc dùng Joi validation:
// OTP_LENGTH: Joi.number().integer().min(4).max(8).default(6)
```

> 🎓 **Best practice**: Luôn validate env schema ở startup (Joi / Zod) → vừa convert type, vừa check range.

---

## 🎲 1. GENERATE OTP (PRIVATE)

```ts
private generateOtp(): string {
  const min = Math.pow(10, this.OTP_LENGTH - 1);
  const max = Math.pow(10, this.OTP_LENGTH) - 1;
  return randomInt(min, max + 1).toString();
}
```

### Giải thích từng dòng

| Dòng | Ý nghĩa | Ví dụ với `OTP_LENGTH = 6` |
|---|---|---|
| `min = 10^(L-1)` | Số nhỏ nhất có L chữ số | `10^5 = 100000` |
| `max = 10^L - 1` | Số lớn nhất có L chữ số | `10^6 - 1 = 999999` |
| `randomInt(min, max + 1)` | Random integer trong `[min, max+1)` | `[100000, 1000000)` → `100000..999999` |
| `.toString()` | Convert sang string | `"123456"` |

### 🎯 Tại sao KHÔNG dùng `Math.random()`?

**`Math.random()` KHÔNG crypto-secure**:

| Tiêu chí | `Math.random()` | `crypto.randomInt()` |
|---|---|---|
| Nguồn entropy | PRNG (V8 xorshift128+) | CSPRNG (OS entropy) |
| Predictable? | ✅ Có thể đoán nếu biết seed | ❌ Không thể đoán |
| Dùng cho | UI animation, game | **Security token, OTP, password** |

**Tấn công thực tế**: Nếu dùng `Math.random()`:
1. Attacker quan sát một vài OTP của user khác → reverse-engineer seed
2. Predict OTP tiếp theo của victim → bypass auth

→ **Bắt buộc** dùng `crypto.randomInt` cho OTP.

### ⚠️ Tại sao KHÔNG có leading zero?

Code hiện tại sinh OTP từ `100000` đến `999999` → **luôn 6 chữ số**, không bao giờ bắt đầu bằng `0`.

**Ưu điểm**: Đơn giản, không cần pad.

**Nhược điểm**: Không gian OTP = `10^6 - 10^5 = 900,000` thay vì `1,000,000` → giảm ~10% entropy (không đáng kể với max 5 attempts).

### ⚠️ Vấn đề `randomInt` range

```ts
randomInt(min, max + 1)
```

Node.js `crypto.randomInt(min, max)` — **max exclusive**. Vậy `randomInt(100000, 1000000)` sinh ra `100000..999999`. Đúng ✅.

**Nhưng**: `crypto.randomInt` có giới hạn `max - min <= 2^48 - 1`. Với OTP length bình thường (≤ 8) → OK.

---

## 📤 2. SEND OTP

```ts
async sendOtp(email: string, purpose: OtpPurposeType): Promise<void> {
  const cooldownKey = RedisKeys.OTP_COOLDOWN(email, purpose);
  const isCooldown = await this.redis.exists(cooldownKey);
  if (isCooldown) {
    const ttl = await this.redis.ttl(cooldownKey);
    throw new TooManyRequestsException(
      `Vui lòng đợi ${ttl} giây trước khi yêu cầu mã mới`,
    );
  }

  const otp = this.generateOtp();
  const payload: OtpPayload = {
    email,
    otp,
    purpose,
    createdAt: Date.now(),
  };

  await this.redis.setJson(
    RedisKeys.OTP(email, purpose),
    payload,
    this.OTP_TTL,
  );

  await this.redis.del(RedisKeys.OTP_ATTEMPTS(email, purpose));
  await this.redis.set(cooldownKey, '1', this.RESEND_COOLDOWN);

  await this.mailService.sendOtpEmail(
    email,
    otp,
    purpose,
    Math.floor(this.OTP_TTL / 60),
  );
}
```

### Giải thích từng bước

#### Bước 1: Check cooldown

```ts
const cooldownKey = RedisKeys.OTP_COOLDOWN(email, purpose);
const isCooldown = await this.redis.exists(cooldownKey);
if (isCooldown) {
  const ttl = await this.redis.ttl(cooldownKey);
  throw new TooManyRequestsException(
    `Vui lòng đợi ${ttl} giây trước khi yêu cầu mã mới`,
  );
}
```

- `RedisKeys.OTP_COOLDOWN(email, purpose)` → key `otp:cooldown:{purpose}:{email}`
- `exists` → check Redis có key không (trả `0` hoặc `1`, wrapper convert thành `bool`)
- Nếu còn → lấy TTL còn lại → throw 429 với message rõ ràng

**Tại sao 2 lệnh Redis (`exists` + `ttl`)?**

Có thể dùng `ttl` trực tiếp: nếu `ttl > 0` → đang cooldown; nếu `ttl === -2` → key không tồn tại.

**Tối ưu**:
```ts
const ttl = await this.redis.ttl(cooldownKey);
if (ttl > 0) {
  throw new TooManyRequestsException(`Vui lòng đợi ${ttl} giây...`);
}
```

→ **Giảm 1 round-trip Redis** (giảm latency ~0.5-1ms).

#### Bước 2: Sinh OTP + tạo payload

```ts
const otp = this.generateOtp();
const payload: OtpPayload = {
  email,
  otp,
  purpose,
  createdAt: Date.now(),
};
```

`OtpPayload` (từ `common/interfaces/otp-payload.interface.ts`):

```ts
export interface OtpPayload {
  email: string;
  otp: string;
  purpose: OtpPurposeType;
  createdAt: number;
}
```

**Tại sao cần `email` + `purpose` trong payload khi đã có trong key?**

- **Defense in depth**: verify lại khi `getJson` → nếu key bị forge, payload vẫn có thể check chéo
- **Debug**: khi đọc Redis thủ công → biết payload thuộc về ai, purpose gì
- **Future-proof**: nếu sau này đổi key pattern → payload vẫn đủ info

**`createdAt` dùng để làm gì?**

Hiện tại **chưa dùng** — có thể dùng cho:
- Audit: OTP tạo lúc nào
- Future: sliding TTL / tính tuổi OTP chính xác hơn
- Debug: so sánh timestamp

#### Bước 3: Lưu OTP vào Redis

```ts
await this.redis.setJson(
  RedisKeys.OTP(email, purpose),
  payload,
  this.OTP_TTL,
);
```

- Key: `otp:{purpose}:{email}` (ví dụ `otp:register:a@b.com`)
- Value: JSON string của `OtpPayload`
- TTL: 300 giây

**⚠️ Tại sao lưu cả `email` + `purpose` trong payload?**

Vì Redis key có thể bị **race condition** khi 2 request đồng thời. Dù key đã phân biệt `email` + `purpose`, việc có payload đầy đủ giúp verify chéo.

#### Bước 4: Reset attempts counter

```ts
await this.redis.del(RedisKeys.OTP_ATTEMPTS(email, purpose));
```

**Tại sao phải xóa attempts?**

- User vừa yêu cầu OTP mới → **reset lại số lần thử**
- Nếu không xóa: user nhập sai 4 lần → request OTP mới → vẫn còn 4 attempts cũ → chỉ được thử 1 lần nữa
- Xóa → user có lại 5 lần thử

#### Bước 5: Set cooldown

```ts
await this.redis.set(cooldownKey, '1', this.RESEND_COOLDOWN);
```

- Key: `otp:cooldown:{purpose}:{email}`
- Value: `'1'` (placeholder)
- TTL: 60 giây

**Tại sao set cooldown SAU KHI set OTP?**

Để tránh race: nếu set cooldown trước rồi OTP lỗi → user bị lock 60s mà không có OTP.

**Nhưng**: Nếu `setJson` fail (Redis down) → `set cooldown` cũng fail → user không bị lock. ✅

#### Bước 6: Gửi email

```ts
await this.mailService.sendOtpEmail(
  email,
  otp,
  purpose,
  Math.floor(this.OTP_TTL / 60),
);
```

- `Math.floor(this.OTP_TTL / 60)` = số phút (300s → 5 phút)
- Truyền vào template để hiển thị "Mã có hiệu lực trong X phút"

### ⚠️ Vấn đề nghiêm trọng

#### Vấn đề 1: Gửi mail **đồng bộ** (await)

```ts
await this.mailService.sendOtpEmail(...);
```

**Hệ quả**:
- Request HTTP phải chờ SMTP server response (~200ms - 2s tùy network)
- Nếu SMTP chậm → user chờ lâu
- Nếu SMTP fail → throw error → user thấy 500 dù OTP đã lưu Redis thành công

**Hệ quả tệ hơn**: Nếu mail fail nhưng OTP đã lưu → user không nhận được OTP nhưng không thể request lại vì **cooldown đã set**.

**Fix**:
```ts
// Đẩy vào queue, không await
this.mailQueue.add('send-otp', { email, otp, purpose }).catch(err => {
  this.logger.error('Failed to enqueue mail', err);
});
// Return ngay
```

Hoặc tối thiểu:
```ts
try {
  await this.mailService.sendOtpEmail(...);
} catch (err) {
  this.logger.error(`Mail fail for ${email}`, err);
  // Không throw — để cooldown + OTP vẫn valid
  // User có thể dùng resend (sau 60s) nếu không nhận được
}
```

#### Vấn đề 2: Race condition giữa cooldown check và set

```
Request A: exists(cooldown) → false
Request B: exists(cooldown) → false
Request A: set OTP + set cooldown
Request B: set OTP (ghi đè) + set cooldown (reset)
```

→ Cả 2 request đều "thành công" → **2 email được gửi** trong cùng khoảng thời gian.

**Fix**: Dùng `SET NX EX` (set if not exists):

```ts
const acquired = await this.redis.set(
  cooldownKey, '1', this.RESEND_COOLDOWN, 'NX',
);
if (!acquired) {
  const ttl = await this.redis.ttl(cooldownKey);
  throw new TooManyRequestsException(...);
}
```

→ **Atomic** — chỉ 1 request thắng.

**Nhưng cần thêm method vào `RedisService`**:
```ts
async setNx(key: string, value: string, ttl: number): Promise<boolean> {
  const result = await this.redis.set(key, value, 'EX', ttl, 'NX');
  return result === 'OK';
}
```

---

## ✅ 3. VERIFY OTP

```ts
async verifyOtp(
  email: string,
  otp: string,
  purpose: OtpPurposeType,
): Promise<boolean> {
  const otpKey = RedisKeys.OTP(email, purpose);
  const attemptsKey = RedisKeys.OTP_ATTEMPTS(email, purpose);

  const attempts = await this.redis.incrWithTtl(attemptsKey, this.OTP_TTL);
  if (attempts > this.MAX_ATTEMPTS) {
    await this.redis.del(otpKey);
    throw new BadRequestException(
      'Bạn đã nhập sai quá nhiều lần. Vui lòng yêu cầu mã mới.',
    );
  }

  const payload = await this.redis.getJson<OtpPayload>(otpKey);
  if (!payload) {
    throw new BadRequestException('Mã OTP đã hết hạn hoặc không tồn tại');
  }

  if (payload.otp !== otp) {
    const remaining = this.MAX_ATTEMPTS - attempts;
    throw new BadRequestException(
      `Mã OTP không đúng. Còn ${remaining} lần thử.`,
    );
  }

  await this.redis.del(otpKey, attemptsKey);
  return true;
}
```

### Giải thích từng bước

#### Bước 1: Increment attempts (atomic)

```ts
const attempts = await this.redis.incrWithTtl(attemptsKey, this.OTP_TTL);
```

`incrWithTtl` (từ `RedisService`):
```ts
async incrWithTtl(key: string, ttlSeconds: number): Promise<number> {
  const value = await this.redis.incr(key);
  if (value === 1) {
    await this.redis.expire(key, ttlSeconds);
  }
  return value;
}
```

- Lần đầu gọi → `attempts = 1`, set TTL = OTP_TTL
- Lần sau → tăng dần, **không reset TTL**

**Tại sao cần atomic?** Nếu không dùng `INCR`:
```ts
const current = await redis.get(attemptsKey); // 0
await redis.set(attemptsKey, current + 1);    // 1
```
→ 2 request đồng thời cùng đọc `0` → cùng ghi `1` → **mất 1 lần đếm**.

`INCR` atomic ở Redis level → an toàn.

**Tại sao set TTL chỉ khi `value === 1`?**

Nếu set TTL mỗi lần:
- User nhập sai lần 1 → TTL 300s
- Chờ 200s → nhập sai lần 2 → TTL reset về 300s
→ Attacker có thể **giữ attempts counter sống mãi** bằng cách nhập sai đều đặn.

Set TTL chỉ lần đầu → counter hết hạn cùng OTP → **đúng logic**.

#### Bước 2: Check vượt max attempts

```ts
if (attempts > this.MAX_ATTEMPTS) {
  await this.redis.del(otpKey);
  throw new BadRequestException(
    'Bạn đã nhập sai quá nhiều lần. Vui lòng yêu cầu mã mới.',
  );
}
```

- Nếu `attempts > 5` → xóa OTP
- **Tại sao xóa OTP?** Buộc user phải request OTP mới → reset counter

**⚠️ Vấn đề**: Attacker có thể **DoS user** bằng cách:
1. Biết email user
2. Gọi verify OTP 5 lần với OTP sai
3. OTP hợp lệ bị xóa → user thật không thể verify

**Hệ quả**: User phải đợi cooldown 60s để request OTP mới → annoyance.

**Mitigation**: Rate limit ở controller (10/15m cho verify) + cooldown 60s cho send → giới hạn tấn công nhưng không triệt để.

**Fix đề xuất**: Cân nhắc **không xóa OTP** khi vượt max, chỉ **lock verify** trong X phút:
```ts
if (attempts > this.MAX_ATTEMPTS) {
  await this.redis.set(`otp:lock:${email}:${purpose}`, '1', 300);
  throw new BadRequestException('Quá nhiều lần thử. Thử lại sau 5 phút.');
}
```

#### Bước 3: Lấy payload

```ts
const payload = await this.redis.getJson<OtpPayload>(otpKey);
if (!payload) {
  throw new BadRequestException('Mã OTP đã hết hạn hoặc không tồn tại');
}
```

- `getJson` tự parse, trả `null` nếu key không tồn tại hoặc JSON hỏng
- Nếu null → OTP hết hạn (TTL 5 phút) hoặc bị xóa

**⚠️ Lưu ý**: `attempts` đã tăng **trước khi** check payload → nếu OTP hết hạn, user vẫn bị tính 1 attempt.

**Hệ quả**: User gọi verify sau khi OTP hết hạn 5 lần → vẫn bị "quá nhiều lần" dù chưa từng nhập sai.

**Fix**: Chỉ tăng attempts **sau khi** confirm payload tồn tại:
```ts
const payload = await this.redis.getJson<OtpPayload>(otpKey);
if (!payload) throw new BadRequestException('Mã OTP đã hết hạn...');

const attempts = await this.redis.incrWithTtl(attemptsKey, this.OTP_TTL);
if (attempts > this.MAX_ATTEMPTS) { ... }

if (payload.otp !== otp) { ... }
```

> 💡 **Trade-off**: Đảo thứ tự → giảm DoS nhưng tăng 1 round-trip Redis (get + incr thay vì incr + get). Với Redis local < 1ms → OK.

#### Bước 4: So sánh OTP

```ts
if (payload.otp !== otp) {
  const remaining = this.MAX_ATTEMPTS - attempts;
  throw new BadRequestException(
    `Mã OTP không đúng. Còn ${remaining} lần thử.`,
  );
}
```

**⚠️ Timing attack?** So sánh `!==` trên string:
- JS engine so sánh từng ký tự → có thể leak timing nếu độ dài khác
- Nhưng: OTP chỉ 6 chữ số, không phải secret dài → timing attack không thực tế

**Nếu paranoid**: Dùng `crypto.timingSafeEqual`:
```ts
const a = Buffer.from(payload.otp);
const b = Buffer.from(otp);
if (a.length !== b.length || !crypto.timingSafeEqual(a, b)) { ... }
```

**`remaining` calculation**:
- `attempts = 1` (lần đầu) → `remaining = 4` (còn 4 lần)
- `attempts = 5` → `remaining = 0` (đã dùng hết)
- `attempts > 5` → đã throw ở bước 2

**UX**: Thông báo "Còn 4 lần thử" → user biết mức độ nghiêm trọng.

#### Bước 5: Cleanup khi thành công

```ts
await this.redis.del(otpKey, attemptsKey);
return true;
```

- Xóa cả OTP và attempts → user không thể dùng lại OTP
- **Idempotency**: Nếu user gọi verify lần 2 → OTP không tồn tại → throw 400 → đúng behavior

---

## 🔍 Phân tích bảo mật

### 🛡️ Layers bảo vệ

| Layer | Cơ chế | Chống |
|---|---|---|
| 1 | `crypto.randomInt` | Predictable OTP |
| 2 | TTL 5 phút | OTP sống quá lâu |
| 3 | Cooldown 60s | Spam send |
| 4 | Max attempts 5 | Brute force |
| 5 | Attempts reset khi resend | Lock user vĩnh viễn |
| 6 | Xóa OTP sau verify | Replay attack |
| 7 | Scope theo `purpose` | Cross-purpose attack |
| 8 | Controller rate limit | Layer ngoài cùng |

### ⚠️ Lỗ hổng đã phân tích

| # | Vấn đề | Mức độ | Fix |
|---|---|---|---|
| 1 | `randomBytes` import thừa | 🟢 Cosmetic | Xóa import |
| 2 | `ConfigService.get<number>` không convert | 🟠 Trung | `Number()` hoặc Joi |
| 3 | Mail gửi sync, fail → user không nhận OTP | 🔴 Cao | Queue / try-catch |
| 4 | Race condition cooldown | 🟠 Trung | `SET NX EX` |
| 5 | DoS user bằng cách spam verify sai | 🟠 Trung | Lock thay vì xóa OTP |
| 6 | Tăng attempts trước khi check payload → OTP hết hạn vẫn tính | 🟡 Thấp | Đảo thứ tự |
| 7 | Verify OTP sau khi OTP hết hạn vẫn tăng attempts | 🟡 Thấp | Cùng #6 |
| 8 | `exists` + `ttl` = 2 round-trip | 🟡 Thấp | Chỉ dùng `ttl` |

---

## 🧪 Test scenarios nên có

```ts
describe('OtpService', () => {
  it('should generate OTP with correct length', ...);
  it('should generate different OTPs on multiple calls', ...);
  it('should throw 429 if cooldown active', ...);
  it('should set OTP + cooldown in Redis', ...);
  it('should reset attempts on resend', ...);
  it('should verify correct OTP and cleanup Redis', ...);
  it('should throw 400 on wrong OTP with remaining count', ...);
  it('should throw 400 and delete OTP after max attempts', ...);
  it('should throw 400 on expired OTP', ...);
  it('should not allow verify after successful verify', ...);
  it('should not allow cross-purpose verify', ...); // register OTP không dùng được cho forgot_password
  it('should handle concurrent send (race condition)', ...); // ⚠️ cần fix trước
});
```

---

## 🔗 Related
- [[Auth-Module]]
- [[Auth-Service-Explained]]
- [[Redis-Keys]]
- [[Mail-Infra]]
- [[Security]]
- [[Phase-1-Auth]]