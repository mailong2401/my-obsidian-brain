---
tags: [api, auth, strategy, jwt, passport, explained]
file: src/modules/auth/strategies/jwt.strategy.ts
---

# JWT Strategy — Giải thích chi tiết

> **Mục đích**: Đây là **Passport strategy** chịu trách nhiệm verify access token (JWT) và "hydrate" `req.user` cho mọi request có `Authorization: Bearer <token>`. Không có strategy này → `JwtAuthGuard` không hoạt động.

## 📦 Import & Code đầy đủ

```ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';
import { UserService } from '../../user/user.service';
import { TokenPayload } from '../auth.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    private readonly configService: ConfigService,
    private readonly userService: UserService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_ACCESS_SECRET'),
    });
  }

  async validate(payload: TokenPayload) {
    const user = await this.userService.findOne(payload.sub);
    if (!user) {
      throw new UnauthorizedException('User not found');
    }
    return user;
  }
}
```

---

## 🧩 Giải thích từng import

| Import | Loại | Vai trò |
|---|---|---|
| `Injectable` | Decorator | Đánh dấu provider cho DI |
| `UnauthorizedException` | Exception | 401 — user không tồn tại |
| `PassportStrategy` | Base class | Abstract class của NestJS để wrap Passport strategy |
| `ExtractJwt` | Helper | Utility lấy token từ request (header, cookie, query...) |
| `Strategy` | Class | Passport JWT strategy gốc (từ `passport-jwt`) |
| `ConfigService` | Provider | Đọc `JWT_ACCESS_SECRET` |
| `UserService` | Provider | Load user từ DB |
| `TokenPayload` | Type | Shape của JWT payload (khớp `AuthService.TokenPayload`) |

---

## 🏛️ Class Definition

```ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor(
    private readonly configService: ConfigService,
    private readonly userService: UserService,
  ) {
    super({ ... });
  }
}
```

### Giải thích

#### `@Injectable()`

- Đánh dấu `JwtStrategy` là **provider** → NestJS DI có thể inject `ConfigService` + `UserService`.
- Được đăng ký trong `AuthModule.providers` → NestJS tự instantiate khi app khởi động.

#### `extends PassportStrategy(Strategy, 'jwt')`

**Đây là dòng quan trọng nhất.** Phân tích:

```ts
PassportStrategy(Strategy, 'jwt')
```

| Tham số | Ý nghĩa |
|---|---|
| `Strategy` | Class gốc từ `passport-jwt` — implement logic verify JWT |
| `'jwt'` | **Tên định danh** — dùng bởi `AuthGuard('jwt')` để lookup strategy |

**Cơ chế bên trong**:

1. `PassportStrategy(Strategy, 'jwt')` return 1 **mixin class** — class mới kế thừa `Strategy` + wrap để tích hợp NestJS.
2. Khi NestJS instantiate `JwtStrategy` → constructor gọi `super(options)` → `passport.use('jwt', new Strategy(options, verifyCallback))`.
3. Passport đăng ký strategy tên `'jwt'` vào global registry.
4. Khi có request với `JwtAuthGuard` → `AuthGuard('jwt')` → lookup strategy `'jwt'` → chạy `authenticate()`.

**Tại sao cần đặt tên `'jwt'`?**

- Nếu chỉ `PassportStrategy(Strategy)` → tên mặc định là `'jwt'` (vì `Strategy` từ `passport-jwt`) → vẫn hoạt động.
- Đặt tên rõ ràng `'jwt'` giúp **explicit** + tránh nhầm nếu sau này có nhiều strategy (ví dụ: `'jwt-refresh'`, `'jwt-2fa'`).

```ts
// Ví dụ tương lai
@Injectable()
export class JwtRefreshStrategy extends PassportStrategy(Strategy, 'jwt-refresh') { ... }

@Injectable()
export class JwtTwoFaStrategy extends PassportStrategy(Strategy, 'jwt-2fa') { ... }
```

→ 3 strategy độc lập, phân biệt qua tên.

#### DI inject 2 dependencies

| Dependency | Dùng để |
|---|---|
| `ConfigService` | Đọc `JWT_ACCESS_SECRET` cho `super()` |
| `UserService` | Load user từ DB trong `validate()` |

> ⚠️ **Lưu ý**: `ConfigService` phải inject **trước** `super()` gọi. TypeScript/JS cho phép dùng biến từ constructor params trong `super()` vì chúng đã được binding trước khi thân constructor chạy.

---

## ⚙️ Passport Options

```ts
super({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  ignoreExpiration: false,
  secretOrKey: configService.get<string>('JWT_ACCESS_SECRET'),
});
```

### 1. `jwtFromRequest`

```ts
jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
```

**Chức năng**: Định nghĩa **cách lấy token** từ request.

**`fromAuthHeaderAsBearerToken()`**:
- Đọc header `Authorization`
- Parse format `Bearer <token>`
- Return `token` hoặc `null` nếu không có/sai format

**Ví dụ request hợp lệ**:
```http
GET /api/auth/profile HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

**Các extractor khác có sẵn**:

| Extractor | Lấy token từ |
|---|---|
| `fromAuthHeaderAsBearerToken()` | Header `Authorization: Bearer X` |
| `fromHeader('x-access-token')` | Custom header |
| `fromBody('token')` | Body field |
| `fromQuery('token')` | Query param |
| `fromCookie('accessToken')` | Cookie |
| `fromExtractors([...])` | Chain nhiều extractor |

**Ví dụ chain (fallback)**:
```ts
jwtFromRequest: ExtractJwt.fromExtractors([
  ExtractJwt.fromAuthHeaderAsBearerToken(),
  ExtractJwt.fromCookie('accessToken'), // fallback
]),
```

→ Code hiện tại **chỉ** dùng Bearer header → OK cho SPA/mobile.

### 2. `ignoreExpiration`

```ts
ignoreExpiration: false,
```

**Chức năng**: Có bỏ qua expiry của JWT không.

- `false` (hiện tại) → **verify expiry** — token hết hạn (15m) → throw `TokenExpiredError`
- `true` → **bỏ qua expiry** — cho phép dùng token hết hạn

**Tại sao `false`?** Vì access token **nên** hết hạn sau 15m → buộc client refresh → rotation xảy ra.

**Khi nào dùng `true`?** Ví dụ: internal service-to-service token với expiry rất dài → không cần check.

> ⚠️ **Đừng bao giờ** set `true` cho user-facing auth.

### 3. `secretOrKey`

```ts
secretOrKey: configService.get<string>('JWT_ACCESS_SECRET'),
```

**Chức năng**: Secret để verify signature.

**Cơ chế**:
- Passport dùng secret này để recompute HMAC SHA-256 của `header.payload`
- So sánh với signature trong token
- Nếu khớp → token authentic
- Nếu không khớp → `JsonWebTokenError: invalid signature`

**⚠️ Vấn đề**: `configService.get<string>('JWT_ACCESS_SECRET')` có thể trả về `undefined` nếu env chưa set.

**Hệ quả**:
- `secretOrKey: undefined` → Passport sẽ throw khi verify (hoặc tệ hơn — nếu pass `undefined` làm secret, token có thể bị forge)
- Behavior tùy version `passport-jwt`

**Fix**:
```ts
const secret = configService.get<string>('JWT_ACCESS_SECRET');
if (!secret) {
  throw new Error('JWT_ACCESS_SECRET is not configured');
}
super({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  ignoreExpiration: false,
  secretOrKey: secret,
});
```

→ **Fail fast** khi app khởi động — tốt hơn là fail lúc runtime.

### 4. Các option KHÁC không dùng nhưng nên biết

| Option | Default | Ghi chú |
|---|---|---|
| `algorithms` | `['HS256', 'HS384', 'HS512']` | ⚠️ **Nên set explicitly** `['HS256']` để chống algorithm confusion |
| `issuer` | — | Verify `iss` claim |
| `audience` | — | Verify `aud` claim |
| `passReqToCallback` | `false` | Nếu `true` → `validate(payload, req)` — có thể dùng để lấy thêm info từ request |

**⚠️ Security tip**: Nên set `algorithms: ['HS256']`:
```ts
super({
  jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
  ignoreExpiration: false,
  secretOrKey: secret,
  algorithms: ['HS256'], // ← explicit
});
```

**Tại sao?** Chống **algorithm confusion attack**:
- Attacker đổi header thành `{"alg": "none"}` → nếu strategy accept → bypass
- Attacker đổi thành `{"alg": "RS256"}` → nếu secret là public key → có thể forge

Set `algorithms: ['HS256']` → strategy chỉ accept HS256.

---

## 🧠 Phương thức `validate`

```ts
async validate(payload: TokenPayload) {
  const user = await this.userService.findOne(payload.sub);
  if (!user) {
    throw new UnauthorizedException('User not found');
  }
  return user;
}
```

### Cơ chế Passport

**Passport flow khi có request**:

```
1. JwtAuthGuard.canActivate()
2. → passport.authenticate('jwt') 
3. → Strategy authenticate():
   3.1. Extract token từ header (jwtFromRequest)
   3.2. Verify signature + expiry
   3.3. Nếu fail → return 401
   3.4. Nếu OK → gọi verify callback
4. → NestJS PassportStrategy wrapper gọi: this.validate(payload)
5. → Return value của validate được gán vào req.user
6. → Guard return true → request tiếp tục tới controller
```

### Giải thích từng dòng

#### Bước 1: Nhận payload

```ts
async validate(payload: TokenPayload) {
```

**`payload`** là decoded JWT payload:
```json
{
  "sub": "user-uuid-123",
  "email": "a@b.com",
  "role": "user",
  "iat": 1704067200,
  "exp": 1704068100
}
```

**Type `TokenPayload`** (từ `AuthService`):
```ts
export interface TokenPayload {
  sub: string;
  email: string;
  role: UserRole;
}
```

→ Type đảm bảo các field có sẵn. `iat`/`exp` không cần vì Passport đã verify.

#### Bước 2: Load user từ DB

```ts
const user = await this.userService.findOne(payload.sub);
```

**Tại sao load lại từ DB thay vì dùng payload?**

| Approach | Pros | Cons |
|---|---|---|
| **Dùng payload** (không query) | Nhanh, không DB hit | Role có thể stale, user có thể bị xóa |
| **Load từ DB** (code hiện tại) | Data fresh, user bị xóa → 401 ngay | +1 DB query/request |

**Code chọn load từ DB** → **fresh data**, an toàn hơn:
- Nếu admin xóa user → access token cũ vẫn còn 15m nhưng `findOne` throw 404 → 401
- Nếu role đổi → `req.user.role` là role mới (dù payload cũ)
- Nếu user bị ban/disable → có thể check thêm flag

**Nhưng**: Tốn 1 DB query cho mỗi request có auth.

**Trade-off**: Với app vừa (< 1000 RPS) → OK. Với scale lớn → nên cache user vào Redis (TTL 30s) hoặc chỉ dùng payload.

#### Bước 3: Check user tồn tại

```ts
if (!user) {
  throw new UnauthorizedException('User not found');
}
```

**⚠️ Nhưng `findOne` đã throw `NotFoundException` nếu không tìm thấy!**

Xem `UserService.findOne`:
```ts
async findOne(id: string): Promise<User> {
  const user = await this.userRepository.findOne({ where: { id } });
  if (!user) {
    throw new NotFoundException(`User with ID "${id}" not found`);
  }
  return user;
}
```

→ **Dead code**: `if (!user)` **không bao giờ** chạy vì `findOne` đã throw trước.

**Hệ quả**:
- Khi user bị xóa → `findOne` throw `NotFoundException` (404)
- Request trả 404 **thay vì** 401
- Attacker có thể phân biệt: "user tồn tại nhưng token sai" (401) vs "token đúng nhưng user bị xóa" (404)

**Semantic HTTP**:
- 401: token không hợp lệ / không authenticate
- 404: resource không tồn tại

→ Về mặt auth, nên là **401** (vì access token là vấn đề, không phải URL).

**Fix**:
```ts
async validate(payload: TokenPayload) {
  const user = await this.userService.findById(payload.sub); // không throw
  if (!user) {
    throw new UnauthorizedException('User not found');
  }
  return user;
}
```

Cần thêm method `findById` (không throw) vào `UserService`:
```ts
async findById(id: string): Promise<User | null> {
  return await this.userRepository.findOne({ where: { id } });
}
```

#### Bước 4: Return user → gán `req.user`

```ts
return user;
```

**Passport wrapper tự động**:
```ts
req.user = user; // ← NestJS PassportStrategy làm việc này
```

Sau đó:
- `@CurrentUser()` decorator đọc `req.user` → trả về user
- `RolesGuard` đọc `req.user.role` → check permission

---

## 🔄 Luồng đầy đủ (request có auth)

```
┌─────────────────────────────────────────────────────────────┐
│ Client: GET /api/auth/profile                                │
│         Authorization: Bearer eyJhbGc...                     │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. NestJS route handler identified: AuthController.getProfile│
│ 2. Guards execute: JwtAuthGuard                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. JwtAuthGuard → passport.authenticate('jwt')               │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. JwtStrategy (Passport):                                   │
│    4.1. ExtractJwt.fromAuthHeaderAsBearerToken()             │
│         → "eyJhbGc..."                                       │
│    4.2. Verify signature với JWT_ACCESS_SECRET               │
│    4.3. Verify exp (15m)                                     │
│    4.4. Nếu OK → decoded payload                             │
│         { sub: "u1", email: "a@b.com", role: "user" }        │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. JwtStrategy.validate(payload):                            │
│    5.1. userService.findOne("u1") → query DB                 │
│    5.2. Nếu không tìm thấy → NotFoundException (404)         │
│    5.3. Return user object                                   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. NestJS PassportStrategy wrapper:                          │
│    req.user = user                                           │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. JwtAuthGuard.canActivate() → return true                  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│ 8. Controller handler:                                       │
│    getProfile(@CurrentUser() user: User) { ... }             │
│    → user = req.user                                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 🎯 Best Practices áp dụng

| Pattern | Áp dụng |
|---|---|
| Strategy pattern | Passport strategy tách biệt khỏi auth logic |
| DI | Inject `ConfigService`, `UserService` |
| Fail-safe | Throw 401 nếu user không tồn tại |
| Fresh data | Load user từ DB thay vì tin payload |
| Named strategy | `'jwt'` tường minh |

---

## ⚠️ Vấn đề cần fix

| # | Vấn đề | Mức độ | Fix |
|---|---|---|---|
| 1 | `secretOrKey` có thể undefined | 🔴 Cao | Throw nếu env missing |
| 2 | Không set `algorithms: ['HS256']` → algorithm confusion | 🟠 Trung | Thêm vào options |
| 3 | `if (!user)` là dead code (findOne đã throw) | 🟠 Trung | Dùng `findById` không throw |
| 4 | Trả 404 thay vì 401 khi user bị xóa | 🟠 Trung | Fix cùng #3 |
| 5 | Load user từ DB mỗi request → tốn query | 🟡 Thấp | Cache Redis (TTL 30s) hoặc tin payload |
| 6 | Không log khi token invalid | 🟡 Thấp | Thêm logger (optional) |
| 7 | Không check `isVerified` | 🟡 Thấp | Nếu muốn chặn user chưa verify |

### 💡 Fix gợi ý (composite)

```ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  private readonly logger = new Logger(JwtStrategy.name);

  constructor(
    private readonly configService: ConfigService,
    private readonly userService: UserService,
  ) {
    const secret = configService.get<string>('JWT_ACCESS_SECRET');
    if (!secret) {
      throw new Error('JWT_ACCESS_SECRET is not configured');
    }

    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: secret,
      algorithms: ['HS256'],
    });
  }

  async validate(payload: TokenPayload) {
    const user = await this.userService.findById(payload.sub);

    if (!user) {
      this.logger.warn(`JWT valid but user not found: ${payload.sub}`);
      throw new UnauthorizedException('User not found');
    }

    if (!user.isVerified) {
      throw new UnauthorizedException('Account not verified');
    }

    return user;
  }
}
```

---

## 🧪 Test scenarios

```ts
describe('JwtStrategy', () => {
  it('should extract token from Authorization header', ...);
  it('should reject request without token', ...); // 401
  it('should reject malformed Authorization header', ...);
  it('should reject expired token', ...); // 401
  it('should reject token with invalid signature', ...); // 401
  it('should reject token with wrong algorithm (alg=none)', ...);
  it('should reject token for deleted user', ...); // 401 (sau fix)
  it('should return user object on valid token', ...);
  it('should set req.user with user entity', ...);
  it('should throw error if JWT_ACCESS_SECRET missing', ...); // app startup
});
```

---

## 🔗 Related
- [[Auth-Module]]
- [[Auth-Service-Explained]]
- [[Session-Service-Explained]]
- [[Security]]
- [[Phase-1-Auth]]