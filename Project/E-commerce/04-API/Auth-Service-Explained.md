---
tags: [api, auth, service, explained]
file: src/modules/auth/auth.service.ts
---

# Auth Service — Giải thích chi tiết

> **Mục đích**: Đây là **trái tim** của module Auth. Chứa toàn bộ business logic: register, login, refresh, logout, session management. Controller chỉ là HTTP layer, service mới là nơi "sự thật" xảy ra.

## 📦 Import & Dependencies

```ts
import {
  Injectable, UnauthorizedException, BadRequestException,
  ForbiddenException, Logger,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
import type { Response } from 'express';
import { OtpService } from './services/otp.service';
import { SessionService, SessionData } from './services/session.service';
import { OtpPurpose } from 'src/common/constants/redis-keys.constant';
import { UserService } from '../user/user.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';
import { User } from '../user/user.entity';
import { UserRole } from 'src/common/enums/user-role.enum';
```

### Giải thích

| Import                           | Vai trò                                                           |
| -------------------------------- | ----------------------------------------------------------------- |
| `Injectable`                     | Đánh dấu class là provider cho DI                                 |
| `UnauthorizedException`          | 401 — sai credentials / chưa verify                               |
| `BadRequestException`            | 400 — request không hợp lệ                                        |
| `ForbiddenException`             | 403 — token reuse detected                                        |
| `Logger`                         | Log nội bộ (không log ra response)                                |
| `JwtService`                     | Sign access token                                                 |
| `ConfigService`                  | Đọc env (JWT secret, expires, NODE_ENV)                           |
| `Response` (type-only)           | Type Express response — dùng để set cookie                        |
| `OtpService`                     | Verify OTP (register)                                             |
| `SessionService` + `SessionData` | Quản lý session-per-device                                        |
| `OtpPurpose`                     | Constant cho purpose (`register`, `forgot_password`, `login_2fa`) |
| `UserService`                    | CRUD user + validate password                                     |
| `RegisterDto`, `LoginDto`        | Input DTO                                                         |
| `User`                           | Entity                                                            |
| `UserRole`                       | Enum role — dùng khi tạo user mới                                 |

---

## 🧩 Interfaces Export

```ts
export interface TokenPayload {
  sub: string;
  email: string;
  role: UserRole;
}

export interface AuthResponse {
  user: Omit<User, 'password'>;
  accessToken: string;
}

export interface RefreshResponse {
  accessToken: string;
}
```

### Giải thích

| Interface | Mục đích |
|---|---|
| `TokenPayload` | Shape của JWT payload. `sub` = userId (theo chuẩn JWT RFC) |
| `AuthResponse` | Response cho login/verify — **KHÔNG** chứa password (dùng `Omit`) |
| `RefreshResponse` | Response cho refresh — chỉ có access token mới (refresh token nằm trong cookie) |

> 💡 `Omit<User, 'password'>` là **compile-time safety** — nếu ai cố `return { user }` nguyên gốc, TypeScript sẽ warning.

---

## 🏛️ Class Definition

```ts
@Injectable()
export class AuthService {
  private readonly logger = new Logger(AuthService.name);

  private readonly REFRESH_COOKIE_NAME = 'refreshToken';
  private readonly DEVICE_COOKIE_NAME = 'deviceId';
  private readonly REFRESH_COOKIE_PATH = '/api/auth';
  private readonly REFRESH_TTL_MS = 7 * 24 * 60 * 60 * 1000; // 7 ngày

  constructor(
    private readonly userService: UserService,
    private readonly jwtService: JwtService,
    private readonly configService: ConfigService,
    private readonly otpService: OtpService,
    private readonly sessionService: SessionService,
  ) { }
}
```

### Giải thích

| Field | Ý nghĩa |
|---|---|
| `logger` | NestJS Logger, prefix `[AuthService]` |
| `REFRESH_COOKIE_NAME` | Tên cookie chứa refresh token |
| `DEVICE_COOKIE_NAME` | Tên cookie chứa device ID |
| `REFRESH_COOKIE_PATH` | **Path giới hạn** — cookie chỉ gửi tới `/api/auth/*`, không leak sang route khác |
| `REFRESH_TTL_MS` | 7 ngày (milliseconds) — khớp với `SessionService.SESSION_TTL` |

### DI inject 5 dependencies

| Dependency | Dùng để |
|---|---|
| `UserService` | CRUD user + validate password |
| `JwtService` | Sign access token |
| `ConfigService` | Đọc secret + expiry |
| `OtpService` | Verify OTP khi register |
| `SessionService` | Tạo/rotate/revoke session |

---

## 📝 1. REGISTER WITH OTP

```ts
async registerWithOtp(registerDto: RegisterDto): Promise<{ message: string }> {
  const user = await this.userService.create({
    ...registerDto,
    role: UserRole.USER,
  });

  await this.otpService.sendOtp(user.email, OtpPurpose.REGISTER);

  return {
    message: 'Đăng ký thành công. Vui lòng kiểm tra email để lấy mã OTP.',
  };
}
```

### Giải thích từng dòng

| Dòng                              | Ý nghĩa                                                                                        |
| --------------------------------- | ---------------------------------------------------------------------------------------------- |
| `this.userService.create(...)`    | Tạo user trong DB, hash password bcrypt, check unique email/username                           |
| `role: UserRole.USER`             | **Force** role = `user` — chống mass assignment (client có thể gửi `role: 'admin'` trong body) |
| `sendOtp(user.email, 'register')` | Sinh OTP + lưu Redis + gửi mail                                                                |
| Return `{ message }`              | Không trả token — user phải verify OTP mới login được                                          |

### ⚠️ Vấn đề nghiêm trọng

**User được tạo NGAY** trước khi verify OTP. Hệ quả:

1. **User rác**: ai đó đăng ký với email không phải của mình → user vẫn tồn tại trong DB với `isVerified = false`
2. **Email squatting**: attacker đăng ký email `admin@company.com` trước → khi owner thật đăng ký → 409 Conflict
3. **DoS DB**: bot spam register → tạo hàng loạt user rác (rate limit 3/1h giảm thiểu nhưng không triệt để)

### ✅ Cách fix (xem [[Phase-1-Auth]])

```ts
async registerWithOtp(registerDto: RegisterDto): Promise<{ message: string }> {
  // 1. Check email/username chưa tồn tại trong DB thật
  await this.userService.assertEmailAvailable(registerDto.email);
  await this.userService.assertUsernameAvailable(registerDto.username);

  // 2. Lưu PENDING registration vào Redis (TTL 15m)
  await this.redis.setJson(
    `pending:register:${registerDto.email}`,
    registerDto,
    15 * 60,
  );

  // 3. Gửi OTP
  await this.otpService.sendOtp(registerDto.email, OtpPurpose.REGISTER);

  return { message: '...' };
}
```

---

## ✅ 2. VERIFY REGISTER OTP

```ts
async verifyRegisterOtp(
  email: string,
  otp: string,
  deviceId: string,
  userAgent: string,
  ip: string,
  res: Response,
): Promise<AuthResponse> {
  await this.otpService.verifyOtp(email, otp, OtpPurpose.REGISTER);

  const user = await this.userService.findByEmail(email);
  if (!user) throw new BadRequestException('User không tồn tại');

  await this.userService.markVerified(user.id);
  user.isVerified = true;

  return this.issueSessionAndRespond(user, deviceId, userAgent, ip, res);
}
```

### Giải thích từng dòng

| Dòng | Ý nghĩa |
|---|---|
| `verifyOtp(email, otp, 'register')` | Verify — throw 400 nếu sai/hết hạn/quá attempts |
| `findByEmail(email)` | Lấy user vừa tạo ở step 1 |
| `if (!user) throw BadRequest` | Defensive check — trường hợp Redis còn OTP nhưng user đã bị xóa |
| `markVerified(user.id)` | UPDATE `is_verified = true` trong DB |
| `user.isVerified = true` | **Đồng bộ in-memory** — vì `markVerified` không trả về user mới, ta phải tự set để response chứa giá trị đúng |
| `issueSessionAndRespond(...)` | Helper tạo session + sign JWT + set cookie + return |

### ⚠️ Vấn đề thứ tự

**Verify OTP TRƯỚC khi check user** — nếu user không tồn tại, OTP vẫn bị "đốt" (bị xóa khỏi Redis). Cân nhắc đảo thứ tự:

```ts
const user = await this.userService.findByEmail(email);
if (!user) throw new BadRequestException('User không tồn tại');
await this.otpService.verifyOtp(email, otp, OtpPurpose.REGISTER);
```

Hoặc tốt hơn: verify xong mới lookup. Trade-off giữa security (không leak user existence) và UX.

> 💡 **Nguyên tắc bảo mật**: Khi email enumeration là rủi ro → nên **verify trước, lookup sau** (như code hiện tại). Khi UX quan trọng hơn → đảo lại.

---

## 🔑 3. LOGIN

```ts
async login(
  loginDto: LoginDto,
  deviceId: string,
  userAgent: string,
  ip: string,
  res: Response,
): Promise<AuthResponse> {
  const user = await this.userService.findByEmailWithPassword(loginDto.email);

  if (!user || !user.password) {
    throw new UnauthorizedException('Invalid credentials');
  }

  const isPasswordValid = await this.userService.validatePassword(
    loginDto.password,
    user.password,
  );

  if (!isPasswordValid) {
    throw new UnauthorizedException('Invalid credentials');
  }

  if (!user.isVerified) {
    throw new UnauthorizedException(
      'Tài khoản chưa được xác thực. Vui lòng xác thực OTP.',
    );
  }

  return this.issueSessionAndRespond(user, deviceId, userAgent, ip, res);
}
```

### Giải thích từng dòng

| Dòng | Ý nghĩa |
|---|---|
| `findByEmailWithPassword` | Dùng QueryBuilder `addSelect('user.password')` vì password có `select: false` |
| `if (!user \|\| !user.password)` | **Cùng 1 message** `'Invalid credentials'` cho cả 2 case → **chống user enumeration** (không cho attacker biết email có tồn tại hay không) |
| `validatePassword` | bcrypt.compare |
| **Message giống nhau** | Lần nữa — không leak "email đúng nhưng password sai" |
| `if (!user.isVerified)` | Sau khi password đúng mới check → **thông báo rõ ràng** vì lúc này user đã chứng minh được identity |
| `issueSessionAndRespond` | Tạo session + JWT + cookie |

### 🎯 Điểm hay

**Thứ tự check thông minh**:
1. Check password **trước** → không leak email tồn tại
2. Check verified **sau** → thông báo cho user biết cần làm gì

Nếu đảo lại → attacker có thể dùng response để phân biệt `email tồn tại + chưa verify` vs `email không tồn tại`.

### ⚠️ Vấn đề

**Timing attack**: Nếu `!user` → return ngay (nhanh); nếu `user` tồn tại → chạy `bcrypt.compare` (chậm ~100ms). Attacker có thể đo timing để biết email tồn tại.

**Cách fix**:
```ts
const DUMMY_HASH = '$2b$10$abcdef...'; // hash giả
const hashToCheck = user?.password ?? DUMMY_HASH;
const isValid = await bcrypt.compare(loginDto.password, hashToCheck);
if (!user || !isValid) throw new UnauthorizedException('Invalid credentials');
```

---

## 🔄 4. REFRESH TOKENS

```ts
async refreshTokens(
  userId: string,
  deviceId: string,
  refreshTokenFromCookie: string,
  userAgent: string,
  ip: string,
  res: Response,
): Promise<RefreshResponse> {
  // 1. Verify với hash trong session
  const isValid = await this.sessionService.verifyRefreshToken(
    userId, deviceId, refreshTokenFromCookie,
  );

  if (!isValid) {
    // ⚠️ Token reuse hoặc session hết hạn → revoke hết
    this.logger.warn(
      `Possible token reuse: user=${userId} device=${deviceId} — revoking all sessions`,
    );
    await this.sessionService.revokeAllUserSessions(userId);
    this.clearAuthCookies(res);
    throw new ForbiddenException(
      'Invalid refresh token. All sessions revoked.',
    );
  }

  // 2. Rotate — cấp refresh token mới
  const { refreshToken: newRefreshToken } =
    await this.sessionService.rotateSession(userId, deviceId, userAgent, ip);

  // 3. Load user + sign access token mới
  const user = await this.userService.findOne(userId);
  const accessToken = await this.signAccessToken(user);

  // 4. Set cookie mới
  this.setAuthCookies(res, newRefreshToken, deviceId);

  return { accessToken };
}
```

### Giải thích từng bước

#### Bước 1: Verify hash

```ts
const isValid = await this.sessionService.verifyRefreshToken(
  userId, deviceId, refreshTokenFromCookie,
);
```

- Lấy `session:{userId}:{deviceId}` từ Redis
- So sánh `SHA256(refreshTokenFromCookie) === session.refreshTokenHash`
- Đây là **bước quan trọng nhất** để phát hiện token reuse

#### Bước 2: Xử lý khi verify fail

```ts
if (!isValid) {
  this.logger.warn(`Possible token reuse: ...`);
  await this.sessionService.revokeAllUserSessions(userId);
  this.clearAuthCookies(res);
  throw new ForbiddenException('Invalid refresh token. All sessions revoked.');
}
```

**Đây là security feature, không phải bug**:

- Khi verify fail → có 2 khả năng:
  1. Session hết hạn (bình thường)
  2. **Token reuse** — attacker đã lấy được token cũ, đang cố dùng lại
- Phản ứng: **revoke TOÀN BỘ session của user** + clear cookie
- Kết quả: cả attacker và user thật đều bị đăng xuất → user thật phải login lại → an toàn

> 🎓 **Tại sao revoke ALL mà không chỉ revoke device hiện tại?**  
> Vì nếu attacker có refresh token, có thể đã lấy được **tất cả** refresh token của user (ví dụ: malware đọc cookie jar). Revoke all là cách duy nhất để chắc chắn.

#### Bước 3: Rotate session

```ts
const { refreshToken: newRefreshToken } =
  await this.sessionService.rotateSession(userId, deviceId, userAgent, ip);
```

- Sinh `newRefreshToken` mới (48 bytes hex)
- Update `session.refreshTokenHash = SHA256(newToken)`
- Lưu mapping `refresh:{newToken} → {userId, deviceId}`
- **Token cũ**: mapping `refresh:{oldToken}` vẫn còn trong Redis (TTL chưa hết) nhưng hash trong session đã đổi → nếu ai dùng token cũ → `verifyRefreshToken` fail → revoke all

> ⚠️ **Điểm yếu**: `refresh:{oldToken}` không bị xóa ngay → attacker có thể gọi `/refresh` nhiều lần với token cũ → **mỗi lần đều trigger revoke all** → DoS user.  
> **Fix**: Xóa `refresh:{oldToken}` ngay khi rotate (xem [[Phase-1-Auth]]).

#### Bước 4: Sign access token mới

```ts
const user = await this.userService.findOne(userId);
const accessToken = await this.signAccessToken(user);
```

- Load user mới từ DB (đảm bảo role mới nhất)
- Sign JWT mới

#### Bước 5: Set cookie mới

```ts
this.setAuthCookies(res, newRefreshToken, deviceId);
```

- Set cookie `refreshToken` = token mới
- Set cookie `deviceId` (giữ nguyên)

---

## 🚪 5. LOGOUT

```ts
async logout(
  userId: string,
  deviceId: string,
  res: Response,
): Promise<{ message: string }> {
  await this.sessionService.revokeSession(userId, deviceId);
  this.clearAuthCookies(res);
  return { message: 'Logged out successfully' };
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| `revokeSession(userId, deviceId)` | Xóa session + device + family trong Redis |
| `clearAuthCookies(res)` | Xóa cookie `refreshToken` + `deviceId` ở client |
| Return message | Không trả token |

### ⚠️ Access token vẫn valid

Access token hiện tại (nếu còn hạn 15m) **vẫn dùng được** cho tới khi hết hạn. Đây là trade-off của stateless JWT. Nếu muốn revoke ngay → cần **blacklist** access token trong Redis (xem [[Security]]).

---

## 🚪🚪 6. LOGOUT ALL

```ts
async logoutAll(userId: string, res: Response): Promise<{ message: string }> {
  await this.sessionService.revokeAllUserSessions(userId);
  this.clearAuthCookies(res);
  return { message: 'Logged out from all devices' };
}
```

### Giải thích

- `revokeAllUserSessions` scan `session:{userId}:*` và `device:{userId}:*` → DEL hết
- Clear cookie của **device hiện tại** (các device khác phải tự clear khi hết hạn access token)

---

## 📱 7. LIST SESSIONS

```ts
async listSessions(
  userId: string,
): Promise<Array<Omit<SessionData, 'refreshTokenHash'>>> {
  const sessions = await this.sessionService.listSessions(userId);
  return sessions.map(({ refreshTokenHash, ...safe }) => safe);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| `listSessions(userId)` | Scan Redis, get tất cả session |
| `.map(({ refreshTokenHash, ...safe }) => safe)` | **Destructuring để loại bỏ** `refreshTokenHash` khỏi response |
| Return type `Omit<SessionData, 'refreshTokenHash'>` | Type-safe: TS đảm bảo không ai vô tình thêm field này lại |

### 🎯 Điểm hay

Đây là **defense in depth** ở 2 tầng:
1. **Runtime**: destructuring loại bỏ field
2. **Compile-time**: return type `Omit<...>` ngăn chặn

Kể cả nếu Redis bị leak → attacker không thấy hash của refresh token (chỉ thấy metadata).

---

## 👤 8. GET PROFILE

```ts
async getProfile(userId: string): Promise<Omit<User, 'password'>> {
  const user = await this.userService.findOne(userId);
  const { password, ...safe } = user;
  return safe;
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| `findOne(userId)` | Lấy user mới nhất từ DB |
| `const { password, ...safe } = user` | Loại password |
| Return type `Omit<User, 'password'>` | Compile-time guarantee |

> 💡 `findOne` không select password (vì `select: false`) → `password` sẽ là `undefined`. Nhưng vẫn destructure để defensive + type-safe.

---

## 🛠️ 9. ISSUE SESSION AND RESPOND (PRIVATE)

```ts
private async issueSessionAndRespond(
  user: User,
  deviceId: string,
  userAgent: string,
  ip: string,
  res: Response,
): Promise<AuthResponse> {
  const { refreshToken } = await this.sessionService.createSession(
    user.id, deviceId, userAgent, ip,
  );

  const accessToken = await this.signAccessToken(user);
  this.setAuthCookies(res, refreshToken, deviceId);

  const { password, ...userWithoutPassword } = user;
  return { user: userWithoutPassword, accessToken };
}
```

### Giải thích

Đây là **helper** dùng chung cho `login` và `verifyRegisterOtp`:

| Bước | Ý nghĩa |
|---|---|
| 1. `createSession` | Sinh refresh token + lưu Redis (4 keys) |
| 2. `signAccessToken` | Sign JWT (15m) |
| 3. `setAuthCookies` | Set 2 cookie httpOnly |
| 4. Destructure user | Loại password khỏi response |
| 5. Return `AuthResponse` | `{ user, accessToken }` |

### 🎯 Tại sao tách helper?

**DRY principle**: 2 flow (`login` + `verifyRegisterOtp`) có **cùng logic cấp session**. Nếu viết lặp → dễ bug khi sửa 1 chỗ quên chỗ kia.

---

## 🔐 10. SIGN ACCESS TOKEN (PRIVATE)

```ts
private async signAccessToken(user: User): Promise<string> {
  const payload: TokenPayload = {
    sub: user.id,
    email: user.email,
    role: user.role,
  };

  return this.jwtService.signAsync(payload, {
    secret: this.configService.get<string>('JWT_ACCESS_SECRET'),
    expiresIn: this.configService.get<string>(
      'JWT_ACCESS_EXPIRES_IN', '15m',
    ) as any,
  });
}
```

### Giải thích

| Field | Ý nghĩa |
|---|---|
| `sub: user.id` | Chuẩn JWT — subject = user ID |
| `email` | Tiện cho logging/debug, không dùng để auth |
| `role` | Dùng cho `RolesGuard` (không cần query DB mỗi request) |
| `signAsync` | Dùng async version (không block event loop) |
| `secret` | Override secret từ config (khác với default ở `JwtModule`) |
| `expiresIn` | `'15m'` |
| `as any` | ⚠️ Type workaround — `@types/ms` chưa cover hết |

### ⚠️ Vấn đề

**`role` trong JWT có thể stale**: nếu admin đổi role của user từ `user` → `admin`, access token cũ vẫn giữ role cũ cho tới khi hết hạn 15m. Đây là trade-off của stateless JWT.

**Fix**: Nếu cần realtime, dùng `JwtStrategy.validate` để re-fetch user + check role từ DB mỗi request (chậm hơn nhưng chính xác). Hiện code **không** làm vậy → chấp nhận 15m delay.

---

## 🍪 11. SET AUTH COOKIES (PRIVATE)

```ts
private setAuthCookies(
  res: Response,
  refreshToken: string,
  deviceId: string,
): void {
  const isProduction =
    this.configService.get<string>('NODE_ENV') === 'production';

  const baseOptions = {
    httpOnly: true,
    secure: isProduction,
    sameSite: 'strict' as const,
    maxAge: this.REFRESH_TTL_MS,
    path: this.REFRESH_COOKIE_PATH,
  };

  res.cookie(this.REFRESH_COOKIE_NAME, refreshToken, baseOptions);
  res.cookie(this.DEVICE_COOKIE_NAME, deviceId, {
    ...baseOptions,
    httpOnly: false,
  });
}
```

### Giải thích từng option

| Option | Value | Tại sao |
|---|---|---|
| `httpOnly: true` | refreshToken | JS không đọc được → chống XSS đánh cắp |
| `httpOnly: false` | deviceId | Client cần đọc để debug/hiển thị |
| `secure: isProduction` | true ở prod | Chỉ gửi qua HTTPS |
| `sameSite: 'strict'` | | Chống CSRF — cookie không gửi từ cross-site request |
| `maxAge: 7 days` | | Khớp TTL session |
| `path: '/api/auth'` | | Cookie chỉ gửi tới auth endpoints → giảm tấn công surface |

### ⚠️ `as const` cho `sameSite`

TypeScript cần literal type `'strict'` (không phải `string`). Nếu thiếu `as const` → type error.

### 🎯 Điểm hay

**Path giới hạn** là layer bảo vệ ít người biết:
- Cookie `refreshToken` **không gửi** tới `/api/users/*`, `/api/products/*`...
- Nếu có XSS ở trang sản phẩm → JS không đọc được refresh token (vì httpOnly) và cũng không tự gửi được tới auth endpoint (vì sameSite strict)

---

## 🧹 12. CLEAR AUTH COOKIES (PRIVATE)

```ts
private clearAuthCookies(res: Response): void {
  const opts = { path: this.REFRESH_COOKIE_PATH };
  res.clearCookie(this.REFRESH_COOKIE_NAME, opts);
  res.clearCookie(this.DEVICE_COOKIE_NAME, opts);
}
```

### Giải thích

| Dòng | Ý nghĩa |
|---|---|
| `clearCookie(name, opts)` | Set cookie với `Max-Age=0` + `Expires` quá khứ → browser xóa |
| **PHẢI có `path` khớp** | Nếu không truyền `path` → browser không xóa cookie ở path `/api/auth` (vì cookie được set với path đó) |

> ⚠️ **Bug tiềm ẩn phổ biến**: `clearCookie` không truyền `path` → cookie vẫn còn. Code này **đúng** vì đã truyền `path`.

---

## 🔍 13. RESOLVE REFRESH TOKEN

```ts
async resolveRefreshToken(
  refreshToken: string,
): Promise<{ userId: string; deviceId: string } | null> {
  return this.sessionService.resolveRefreshToken(refreshToken);
}
```

### Giải thích

**Facade method** — controller không gọi trực tiếp `SessionService` mà đi qua `AuthService`. Lợi ích:
- Controller chỉ phụ thuộc `AuthService` (1 dependency)
- Nếu sau này cần thêm logic (log, rate check) → sửa 1 chỗ

> 💡 **Pattern**: Đây gọi là **Law of Demeter** — không reach xuyên qua nhiều layer.

---

## 🎯 Tổng kết bảo mật của AuthService

| Feature | Implementation | File |
|---|---|---|
| Password hash | bcrypt 10 rounds | `UserService` |
| Access token | JWT HS256, 15m | `signAccessToken` |
| Refresh token | Opaque 48 bytes, SHA-256 hash | `SessionService` |
| Cookie httpOnly | refreshToken | `setAuthCookies` |
| Cookie path scope | `/api/auth` | `setAuthCookies` |
| SameSite strict | Cả 2 cookie | `setAuthCookies` |
| Token rotation | Mỗi refresh tạo token mới | `rotateSession` |
| Reuse detection | Verify fail → revoke all | `refreshTokens` |
| User enumeration prevention | Cùng message `'Invalid credentials'` | `login` |
| Mass assignment prevention | Force `role: USER` | `registerWithOtp` |
| Password leak prevention | `Omit<User, 'password'>` | `AuthResponse`, `getProfile`, `listSessions` |
| Refresh token hash leak prevention | Destructuring + `Omit` | `listSessions` |

---

## ⚠️ Vấn đề cần fix (ưu tiên)

| # | Vấn đề | Mức độ | Fix |
|---|---|---|---|
| 1 | Register tạo user trước verify → user rác, email squatting | 🔴 Cao | Pending registration trong Redis |
| 2 | `refresh:{oldToken}` không xóa khi rotate → DoS user | 🟠 Trung | Xóa ngay trong `rotateSession` |
| 3 | `signAccessToken` dùng `as any` | 🟡 Thấp | Định nghĩa type `JwtExpiresIn` |
| 4 | Role trong JWT có thể stale 15m | 🟠 Trung | Re-fetch trong `JwtStrategy.validate` nếu cần |
| 5 | Access token không bị revoke khi logout | 🟠 Trung | Blacklist trong Redis (optional) |
| 6 | Timing attack trong login | 🟡 Thấp | Dummy hash compare |
| 7 | Chưa có forgot-password flow | 🔴 Cao | Xem [[Phase-1-Auth]] |
| 8 | Chưa có login-2FA | 🟡 Thấp | Optional feature |

---

## 🔗 Related
- [[Auth-Module]]
- [[Auth-Controller-Explained]]
- [[Session Service|Auth-Module]]
- [[Otp Service|Auth-Module]]
- [[Security]]
- [[Phase-1-Auth]]