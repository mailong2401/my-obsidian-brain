---
tags: [api, auth, controller, explained]
file: src/modules/auth/auth.controller.ts
---

# Auth Controller — Giải thích chi tiết

> **Mục đích**: Đây là "cửa ngõ" HTTP của toàn bộ module Auth. Nó **không chứa business logic** — chỉ nhận request, gọi service, trả response.

## 📦 Import & Dependencies

```ts
import {
  Controller, Post, Body, Get, UseGuards,
  HttpCode, HttpStatus, Req, Res, Headers, Ip,
  UnauthorizedException,
} from '@nestjs/common';
import { Throttle } from '@nestjs/throttler';
import type { Request, Response } from 'express';

import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { JwtAuthGuard } from './guards/jwt-auth.guard';
import { CurrentUser } from './decorators/current-user.decorator';
import { User } from '../user/user.entity';
import { OtpService } from './services/otp.service';
import { VerifyOtpDto } from './dto/verify-otp.dto';
import { ResendOtpDto } from './dto/resend-otp.dto';
import { DeviceId } from 'src/common/decorators/device-id.decorator';
```

### Giải thích từng import

| Import | Vai trò |
|---|---|
| `Controller`, `Post`, `Get` | Decorator định nghĩa route |
| `Body` | Lấy body từ request |
| `UseGuards` | Gắn guard (JWT, Roles...) |
| `HttpCode`, `HttpStatus` | Override HTTP status trả về |
| `Req`, `Res` | Truy cập `Request`/`Response` của Express |
| `Headers` | Lấy 1 header cụ thể (ở đây là `user-agent`) |
| `Ip` | Lấy IP client (đã tính cả `X-Forwarded-For` nếu trust proxy) |
| `UnauthorizedException` | Throw 401 khi thiếu/sai refresh token |
| `Throttle` | Override rate limit cho từng route |
| `Request`, `Response` (type-only) | Type của Express, dùng `import type` để không bundle runtime |
| `AuthService` | Business logic chính |
| `OtpService` | Logic OTP (dùng cho route resend) |
| `RegisterDto`, `LoginDto`, `VerifyOtpDto`, `ResendOtpDto` | DTO validate input |
| `JwtAuthGuard` | Guard yêu cầu Bearer token hợp lệ |
| `CurrentUser` | Custom decorator lấy `req.user` |
| `User` | Entity (chỉ dùng làm type) |
| `DeviceId` | Custom decorator đọc header `X-Device-Id` |

---

## 🏛️ Class Definition

```ts
@Controller('auth')
export class AuthController {
  constructor(
    private readonly authService: AuthService,
    private readonly otpService: OtpService,
  ) { }
}
```

### Giải thích

- **`@Controller('auth')`**: Tiền tố route = `/auth`. Kết hợp với global prefix `api` (set trong `main.ts`) → tất cả endpoint có base `/api/auth`.
- **Constructor injection**: NestJS DI inject `AuthService` và `OtpService` (cả 2 đều là provider trong `AuthModule`).
- **`private readonly`**: Đảm bảo không mutate reference, chỉ đọc.

> ⚠️ **Lưu ý**: Controller này **không** inject `SessionService` vì mọi thao tác session đều đi qua `AuthService` (đúng nguyên tắc: controller → 1 service facade).

---

## 🔐 1. LOGIN

```ts
@Post('login')
@Throttle({ default: { ttl: 900_000, limit: 5 } })
@HttpCode(HttpStatus.OK)
async login(
  @Body() loginDto: LoginDto,
  @DeviceId() deviceId: string,
  @Headers('user-agent') userAgent: string,
  @Ip() ip: string,
  @Res({ passthrough: true }) res: Response,
) {
  return this.authService.login(loginDto, deviceId, userAgent, ip, res);
}
```

### Giải thích từng dòng

| Dòng | Ý nghĩa |
|---|---|
| `@Post('login')` | Route `POST /api/auth/login` |
| `@Throttle({ default: { ttl: 900_000, limit: 5 } })` | **5 lần / 15 phút** — chống brute force password. `ttl` tính bằng **milliseconds** |
| `@HttpCode(HttpStatus.OK)` | Mặc định `POST` là 201, nhưng login là hành động đọc → trả 200 |
| `@Body() loginDto` | Parse + validate body theo `LoginDto` (email + password) |
| `@DeviceId() deviceId` | Custom decorator: đọc `X-Device-Id`, throw 400 nếu thiếu hoặc < 8 ký tự |
| `@Headers('user-agent')` | Lấy UA để lưu vào session (audit/UI) |
| `@Ip() ip` | Lấy IP client (NestJS tự xử lý `trust proxy`) |
| `@Res({ passthrough: true })` | Inject response **nhưng vẫn để NestJS tự gửi response** — cần thiết để set cookie |
| `return this.authService.login(...)` | Delegate toàn bộ logic cho service |

### ⚠️ Tại sao `passthrough: true`?

Nếu **không có** `passthrough`, NestJS sẽ coi bạn tự chịu trách nhiệm gửi response → bạn phải gọi `res.json()` thủ công.  
Với `passthrough: true`, bạn **vẫn có thể set cookie / header** qua `res`, nhưng NestJS vẫn serialize `return value` thành response body.

### 🧠 Luồng xử lý

```
Request → Throttle check (5/15m)
       → ValidationPipe (LoginDto)
       → DeviceId decorator (check X-Device-Id)
       → AuthService.login()
           ├─ findByEmailWithPassword
           ├─ validatePassword (bcrypt)
           ├─ check isVerified
           ├─ SessionService.createSession → Redis
           ├─ signAccessToken → JWT
           └─ setAuthCookies (refreshToken + deviceId)
       → Response 200 { user, accessToken } + Set-Cookie
```

---

## 📝 2. REGISTER — STEP 1: SEND OTP

```ts
@Post('register/send-otp')
@Throttle({ default: { ttl: 3_600_000, limit: 3 } })
@HttpCode(HttpStatus.CREATED)
async registerSendOtp(@Body() dto: RegisterDto) {
  return this.authService.registerWithOtp(dto);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `POST /api/auth/register/send-otp` |
| `@Throttle({ ttl: 3_600_000, limit: 3 })` | **3 lần / 1 giờ** — chống spam email. TTL 1h = 3_600_000 ms |
| `@HttpCode(HttpStatus.CREATED)` | Trả 201 vì có side effect (tạo resource: OTP + có thể là user pending) |
| `@Body() dto: RegisterDto` | Validate toàn bộ field đăng ký |
| Return `{ message }` | Không trả token — user phải verify OTP trước |

### ⚠️ Vấn đề thiết kế hiện tại

`AuthService.registerWithOtp` **tạo User ngay** rồi mới gửi OTP → user rác nếu không verify.  
→ Cần refactor: lưu **pending registration** vào Redis, chỉ tạo User ở step 2 (xem [[Phase-1-Auth]]).

### 🧠 Luồng

```
Request → Throttle (3/1h)
       → Validate RegisterDto
       → AuthService.registerWithOtp
           ├─ UserService.create (check email/username unique + hash password)
           └─ OtpService.sendOtp(email, 'register')
               ├─ Check cooldown (60s)
               ├─ crypto.randomInt → OTP
               ├─ Redis SETEX otp:register:{email}
               ├─ Redis SETEX otp:cooldown:register:{email}
               └─ MailService.sendOtpEmail
       → Response 201 { message }
```

---

## ✅ 3. REGISTER — STEP 2: VERIFY OTP

```ts
@Post('register/verify-otp')
@Throttle({ default: { ttl: 900_000, limit: 10 } })
@HttpCode(HttpStatus.OK)
async registerVerifyOtp(
  @Body() dto: VerifyOtpDto,
  @DeviceId() deviceId: string,
  @Headers('user-agent') userAgent: string,
  @Ip() ip: string,
  @Res({ passthrough: true }) res: Response,
) {
  return this.authService.verifyRegisterOtp(
    dto.email, dto.otp, deviceId, userAgent, ip, res,
  );
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `POST /api/auth/register/verify-otp` |
| `@Throttle({ ttl: 900_000, limit: 10 })` | **10 lần / 15 phút** — chống brute force OTP. **Ngoài ra** `OtpService` còn có attempts counter 5 lần/OTP → 2 tầng bảo vệ |
| `@HttpCode(HttpStatus.OK)` | 200 vì đây là hành động verify (không tạo resource mới ở HTTP level) |
| `@DeviceId()` | Bắt buộc — vì sau khi verify thành công sẽ tạo session theo device |
| `@Res({ passthrough: true })` | Cần để set cookie refreshToken + deviceId |

### 🧠 Luồng

```
Request → Throttle (10/15m)
       → Validate VerifyOtpDto (email, otp 4-8 digits, purpose='register')
       → DeviceId check
       → AuthService.verifyRegisterOtp
           ├─ OtpService.verifyOtp(email, otp, 'register')
           │   ├─ incrWithTtl attempts (max 5)
           │   ├─ getJson otp:register:{email}
           │   ├─ Compare OTP
           │   └─ DEL otp + attempts nếu OK
           ├─ UserService.findByEmail → markVerified
           ├─ SessionService.createSession → Redis
           ├─ signAccessToken
           └─ setAuthCookies
       → Response 200 { user, accessToken } + Set-Cookie
```

---

## 🔁 4. OTP RESEND

```ts
@Post('otp/resend')
@Throttle({ default: { ttl: 3_600_000, limit: 5 } })
@HttpCode(HttpStatus.OK)
async resendOtp(@Body() dto: ResendOtpDto) {
  await this.otpService.sendOtp(dto.email, dto.purpose);
  return { message: 'Đã gửi lại mã OTP' };
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `POST /api/auth/otp/resend` |
| `@Throttle({ ttl: 3_600_000, limit: 5 })` | **5 lần / 1 giờ** — chống spam resend |
| `ResendOtpDto` | `{ email, purpose }` với `purpose ∈ ['register', 'forgot_password', 'login_2fa']` |
| **Gọi trực tiếp `OtpService`** | Đây là **ngoại lệ** — route này không cần business logic phức tạp, chỉ resend OTP nên controller gọi thẳng service |
| Cooldown 60s | Vẫn được enforce bên trong `OtpService.sendOtp` → nếu user spam → 429 |

### ⚠️ Lưu ý

Route này dùng chung cho **mọi purpose** → 1 route, 3 use case. Đây là trade-off giữa DRY và clarity.  
Nếu sau này logic resend khác nhau theo purpose → nên tách route.

---

## 🔄 5. REFRESH TOKEN

```ts
@Post('refresh')
@Throttle({ default: { ttl: 60_000, limit: 20 } })
@HttpCode(HttpStatus.OK)
async refresh(
  @Req() req: Request,
  @Res({ passthrough: true }) res: Response,
  @Headers('user-agent') userAgent: string,
  @Ip() ip: string,
) {
  const refreshToken = req.cookies?.['refreshToken'];
  if (!refreshToken) {
    throw new UnauthorizedException('Missing refresh token');
  }

  const resolved = await this.authService.resolveRefreshToken(refreshToken);
  if (!resolved) {
    throw new UnauthorizedException('Invalid refresh token');
  }

  return this.authService.refreshTokens(
    resolved.userId, resolved.deviceId,
    refreshToken, userAgent, ip, res,
  );
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `POST /api/auth/refresh` |
| `@Throttle({ ttl: 60_000, limit: 20 })` | **20 lần / 1 phút** — vì client tự động refresh, cần limit cao hơn |
| **KHÔNG** có `@UseGuards(JwtAuthGuard)` | Access token hết hạn rồi mới refresh → không thể require JWT |
| `@Req() req` | Cần để đọc `req.cookies` (đã parse bởi `cookie-parser`) |
| `req.cookies?.['refreshToken']` | Optional chaining vì `req.cookies` có thể undefined nếu middleware chưa chạy |
| `resolveRefreshToken` | Tra cứu `refresh:{token}` trong Redis → `{ userId, deviceId }` |
| **2 bước resolve + verify** | Bước 1: resolve (lookup). Bước 2 (trong service): verify hash. Đây là **defense in depth** — kể cả attacker đoán được token format, vẫn phải khớp hash trong session |

### 🧠 Luồng

```
Request → Throttle (20/1m)
       → Đọc cookie refreshToken
       → AuthService.resolveRefreshToken (Redis lookup)
       → AuthService.refreshTokens
           ├─ SessionService.verifyRefreshToken
           │   ├─ getSession (Redis)
           │   └─ SHA256(plain) === session.refreshTokenHash ?
           ├─ NẾU FAIL → revokeAllUserSessions + clearCookies + 403
           ├─ NẾU OK → SessionService.rotateSession
           │   ├─ Tạo refreshToken mới
           │   ├─ Update session với hash mới
           │   └─ Lưu refresh:{newToken} mapping
           ├─ Load user + signAccessToken
           └─ setAuthCookies (refreshToken mới + deviceId)
       → Response 200 { accessToken } + Set-Cookie mới
```

### ⚠️ Tại sao route này **không** có `@DeviceId()` decorator?

Vì `deviceId` được **lấy từ cookie** (không phải header). Lý do:
- Refresh thường được client tự động gọi (axios interceptor) → dễ quên set header
- Cookie `deviceId` đã được set từ login → tự động gửi kèm

⚠️ **Điểm yếu**: Cookie `deviceId` có `httpOnly: false` (để client đọc được) → có thể bị JS đọc nếu XSS. Nhưng vì refresh token vẫn httpOnly → attacker không thể làm gì nếu chỉ có deviceId.

---

## 🚪 6. LOGOUT (1 DEVICE)

```ts
@Post('logout')
@Throttle({ default: { ttl: 60_000, limit: 10 } })
@UseGuards(JwtAuthGuard)
@HttpCode(HttpStatus.OK)
async logout(
  @CurrentUser() user: User,
  @Req() req: Request,
  @Res({ passthrough: true }) res: Response,
) {
  const deviceId = req.cookies?.['deviceId'];
  if (!deviceId) throw new UnauthorizedException('Missing device id');
  return this.authService.logout(user.id, deviceId, res);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `POST /api/auth/logout` |
| `@UseGuards(JwtAuthGuard)` | Yêu cầu access token hợp lệ → phải login trước khi logout |
| `@CurrentUser() user: User` | Custom decorator lấy `req.user` (do `JwtStrategy.validate` set) |
| `req.cookies?.['deviceId']` | Lấy deviceId từ cookie — biết cần revoke session nào |
| `if (!deviceId) throw 401` | Nếu không có cookie → không biết revoke cái gì |
| `authService.logout(user.id, deviceId, res)` | Revoke session + clear cookies |

### ⚠️ Tại sao vẫn cần `JwtAuthGuard` nếu đã có cookie?

Vì:
1. **Xác thực user thực sự** — cookie `deviceId` có thể bị forge
2. **`user.id`** cần cho `revokeSession(userId, deviceId)` — cookie không chứa userId
3. **Defense in depth**: kể cả cookie bị leak, attacker vẫn cần access token

### ⚠️ Path cookie có khớp không?

Cookie set với `path: '/api/auth'`, route logout là `/api/auth/logout` → **khớp prefix** → cookie vẫn được gửi. ✅

### 🧠 Luồng

```
Request → JwtAuthGuard (verify Bearer)
       → Throttle (10/1m)
       → CurrentUser (lấy user)
       → Đọc cookie deviceId
       → AuthService.logout
           ├─ SessionService.revokeSession
           │   ├─ getSession để lấy familyId
           │   ├─ DEL family:{familyId}
           │   └─ DEL session:{userId}:{deviceId} + device:{userId}:{deviceId}
           └─ clearAuthCookies
       → Response 200 { message }
```

---

## 🚪🚪 7. LOGOUT ALL DEVICES

```ts
@Post('logout-all')
@Throttle({ default: { ttl: 3_600_000, limit: 5 } })
@UseGuards(JwtAuthGuard)
@HttpCode(HttpStatus.OK)
async logoutAll(
  @CurrentUser() user: User,
  @Res({ passthrough: true }) res: Response,
) {
  return this.authService.logoutAll(user.id, res);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `POST /api/auth/logout-all` |
| `@Throttle({ ttl: 3_600_000, limit: 5 })` | **5 lần / 1 giờ** — thao tác nguy hiểm (đá hết mọi device), không nên gọi nhiều |
| `@UseGuards(JwtAuthGuard)` | Yêu cầu login |
| Không cần `deviceId` | Vì revoke **tất cả** session của user |

### 🧠 Luồng

```
Request → JwtAuthGuard
       → Throttle (5/1h)
       → AuthService.logoutAll(user.id, res)
           ├─ SessionService.revokeAllUserSessions
           │   ├─ scanKeys session:{userId}:*
           │   ├─ DEL tất cả
           │   ├─ scanKeys device:{userId}:*
           │   └─ DEL tất cả
           └─ clearAuthCookies
       → Response 200
```

### ⚠️ Lưu ý

`scanKeys` dùng SCAN (không block Redis). Nhưng `DEL(...keys)` với mảng lớn có thể block → cần chunk nếu user có > 1000 session (hiếm).

---

## 👤 8. GET PROFILE

```ts
@Get('profile')
@Throttle({ default: { ttl: 60_000, limit: 60 } })
@UseGuards(JwtAuthGuard)
async getProfile(@CurrentUser() user: User) {
  return this.authService.getProfile(user.id);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `GET /api/auth/profile` |
| `@Throttle({ ttl: 60_000, limit: 60 })` | **60 lần / 1 phút** — route đọc, ít rủi ro |
| `@UseGuards(JwtAuthGuard)` | Yêu cầu login |
| Không set `@HttpCode` | Mặc định `GET` là 200 ✅ |
| `authService.getProfile(user.id)` | Re-fetch từ DB (fresh data, không tin token payload) |

### ⚠️ Tại sao không trả `user` từ `@CurrentUser` luôn?

Vì `@CurrentUser` trả về user đã được `JwtStrategy.validate` fetch — nhưng để **nhất quán** với data mới nhất (role có thể vừa đổi), controller gọi lại service.

> 💡 **Tối ưu**: Nếu muốn nhanh, có thể return luôn `user` từ `@CurrentUser` (đã fetch trong strategy). Hiện tại code gọi lại → thêm 1 DB query. Không sai, chỉ là trade-off freshness vs performance.

---

## 📱 9. LIST SESSIONS

```ts
@Get('sessions')
@Throttle({ default: { ttl: 60_000, limit: 30 } })
@UseGuards(JwtAuthGuard)
async listSessions(@CurrentUser() user: User) {
  return this.authService.listSessions(user.id);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| Route | `GET /api/auth/sessions` |
| `@Throttle({ ttl: 60_000, limit: 30 })` | **30 lần / 1 phút** |
| `authService.listSessions(user.id)` | Scan Redis `session:{userId}:*` → trả list (đã filter bỏ `refreshTokenHash`) |

### 🧠 Response shape

```json
[
  {
    "deviceId": "abc-123",
    "familyId": "xyz",
    "userAgent": "Mozilla/5.0...",
    "ip": "1.2.3.4",
    "createdAt": 1704067200000,
    "lastUsedAt": 1704153600000
  }
]
```

⚠️ **`refreshTokenHash` bị loại bỏ** trước khi trả — xem `AuthService.listSessions` dùng destructuring `{ refreshTokenHash, ...safe }`.

---

## 🎯 Tổng kết bảo mật

| Layer | Cơ chế | Route nào |
|---|---|---|
| Rate limit global | `UserThrottlerGuard` (Redis) | Tất cả |
| Rate limit per-route | `@Throttle` | Tất cả |
| Device binding | `@DeviceId` (header) hoặc cookie | Login, Register, Verify |
| JWT auth | `JwtAuthGuard` | Logout, Profile, Sessions |
| Cookie httpOnly | `setAuthCookies` | Login, Verify, Refresh |
| SameSite strict | Cookie option | Tất cả cookie |
| OTP attempts | `OtpService.verifyOtp` | Verify |
| OTP cooldown | `OtpService.sendOtp` | Send, Resend |
| Token reuse detect | `refreshTokens` | Refresh |
| Password hash | bcrypt 10 rounds | `UserService` |

## ⚠️ Điểm yếu cần fix (tóm tắt)

1. **Register tạo user trước verify** → user rác.
2. **`@Throttle` hard-code** trong controller → khó maintain, nên đưa vào constant.
3. **Route `otp/resend`** gọi thẳng `OtpService` (không qua `AuthService`) → phá vỡ facade pattern.
4. **`refresh` route** không có `@DeviceId` → phụ thuộc cookie `deviceId` (httpOnly false → XSS đọc được, tuy không nguy hiểm).
5. **`logout` đọc `deviceId` từ cookie** nhưng `deviceId` cũng có thể lấy từ session (an toàn hơn nếu cookie bị clear).
6. **Chưa có** `forgot-password`, `login-2fa` endpoint → cần bổ sung.

## 🔗 Related
- [[Auth-Module]]
- [[Auth-Endpoints]]
- [[Rate-Limiting]]
- [[Security]]