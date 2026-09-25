---
tags: [api, auth, module, explained]
file: src/modules/auth/auth.module.ts
---

# Auth Module — Giải thích chi tiết

> **Mục đích**: Đây là file **wiring** (dây nối) của module Auth. Nó KHÔNG chứa logic — chỉ khai báo:
> - Module nào cần import
> - Provider nào cần khởi tạo (DI)
> - Controller nào cần đăng ký route
> - Provider nào export ra ngoài cho module khác dùng

## 📦 Import & Code đầy đủ

```ts
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import type { StringValue } from 'ms';
import { OtpService } from './services/otp.service';
import { AuthService } from './auth.service';
import { AuthController } from './auth.controller';
import { UserModule } from '../user/user.module';
import { JwtStrategy } from './strategies/jwt.strategy';
import { SessionService } from './services/session.service';

@Module({
  imports: [
    UserModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get<string>('JWT_ACCESS_SECRET'),
        signOptions: {
          expiresIn: configService.get<string>(
            'JWT_ACCESS_EXPIRES_IN',
            '15m',
          ) as StringValue,
        },
      }),
    }),
  ],
  providers: [AuthService, JwtStrategy, OtpService, SessionService],
  controllers: [AuthController],
  exports: [AuthService],
})
export class AuthModule { }
```

---

## 🧩 Giải thích từng import

| Import | Loại | Vai trò |
|---|---|---|
| `Module` | Decorator | Đánh dấu class là NestJS module |
| `JwtModule` | Module | Cung cấp `JwtService` để sign/verify JWT |
| `PassportModule` | Module | Cung cấp infrastructure cho Passport strategies |
| `ConfigModule` | Module | Cho phép inject `ConfigService` vào factory |
| `ConfigService` | Provider | Đọc env |
| `StringValue` (type-only) | Type | Type của thư viện `ms` — cho phép `expiresIn: '15m'` type-safe |
| `OtpService` | Provider | Logic OTP |
| `AuthService` | Provider | Business logic chính |
| `AuthController` | Controller | HTTP routes |
| `UserModule` | Module | Cần vì `AuthService` dùng `UserService` |
| `JwtStrategy` | Provider | Passport strategy validate access token |
| `SessionService` | Provider | Logic session-per-device |

> 💡 **`import type { StringValue } from 'ms'`**: dùng `import type` → TS sẽ **xóa hoàn toàn** dòng này khi compile (không sinh `require('ms')`), tránh bundle thừa.

---

## 🏛️ Cấu trúc `@Module` Decorator

```ts
@Module({
  imports: [...],      // Module khác cần dùng
  providers: [...],    // Service/Strategy của module này
  controllers: [...],  // Route handlers
  exports: [...],      // Provider cho module khác dùng
})
export class AuthModule { }
```

### So sánh 4 metadata

| Metadata | Ý nghĩa | Ở đây |
|---|---|---|
| `imports` | Module phụ thuộc — để DI resolve dependency | `UserModule`, `PassportModule`, `JwtModule` |
| `providers` | Class được NestJS quản lý + inject | `AuthService`, `JwtStrategy`, `OtpService`, `SessionService` |
| `controllers` | Class định nghĩa route | `AuthController` |
| `exports` | Provider cho module khác dùng | `AuthService` |

> ⚠️ **Quan trọng**: `exports` chỉ export **provider**, KHÔNG export `imports`. Nếu muốn module khác dùng `JwtService`, phải re-export `JwtModule`:
> ```ts
> exports: [AuthService, JwtModule],
> ```

---

## 📥 Phần 1: `imports`

### 1.1. `UserModule`

```ts
UserModule,
```

**Tại sao cần?**

`AuthService` inject `UserService`:

```ts
constructor(
  private readonly userService: UserService,  // ← cần UserModule export UserService
  ...
) { }
```

`UserModule` phải **export** `UserService`:

```ts
// user.module.ts
@Module({
  ...
  exports: [UserService],  // ← điều kiện bắt buộc
})
export class UserModule { }
```

**Nếu thiếu `UserModule` trong imports**:

```
Nest can't resolve dependencies of the AuthService (?).
Please make sure that the argument UserService at index [0] is available in the AuthModule context.
```

### 1.2. `PassportModule`

```ts
PassportModule,
```

**Tại sao cần?**

- `JwtStrategy extends PassportStrategy(Strategy, 'jwt')` — cần Passport infrastructure để đăng ký strategy
- `JwtAuthGuard extends AuthGuard('jwt')` — cần Passport để lookup strategy tên `'jwt'`
- `PassportModule` cung cấp `PassportStrategy` base class + register mechanism

**Tại sao KHÔNG có config?**

```ts
PassportModule,  // ← dùng default
```

Không cần `register()` vì:
- Không set `defaultStrategy` (vì có thể có nhiều strategy: `jwt`, `local`, `google`...)
- Strategy được đăng ký tự động qua `JwtStrategy` provider

> 💡 Nếu sau này thêm `LocalStrategy` cho login → chỉ cần thêm provider, không cần đổi `PassportModule`.

### 1.3. `JwtModule.registerAsync`

```ts
JwtModule.registerAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (configService: ConfigService) => ({
    secret: configService.get<string>('JWT_ACCESS_SECRET'),
    signOptions: {
      expiresIn: configService.get<string>(
        'JWT_ACCESS_EXPIRES_IN',
        '15m',
      ) as StringValue,
    },
  }),
}),
```

**Tại sao dùng `registerAsync`?**

`ConfigService` chỉ có sau khi `ConfigModule` được load. Nếu dùng `register()` (sync) → không inject được `ConfigService`.

**Pattern async config trong NestJS**:

```
registerAsync({
  imports: [ConfigModule],       // 1. Import module cần
  inject: [ConfigService],       // 2. Khai báo dependency cần inject
  useFactory: (config) => ({...}) // 3. Factory nhận dependency → return config
})
```

**So sánh 3 cách config**:

| Cách | Khi nào dùng |
|---|---|
| `JwtModule.register({ secret: 'hard-code' })` | ❌ Không bao giờ (leak secret) |
| `JwtModule.register({ secret: process.env.X })` | ⚠️ Chạy được nhưng không testable |
| `JwtModule.registerAsync({...})` | ✅ Chuẩn — DI, testable, đọc từ ConfigService |

**Tại sao `expiresIn` có `as StringValue`?**

```ts
expiresIn: configService.get<string>('JWT_ACCESS_EXPIRES_IN', '15m') as StringValue,
```

- `configService.get<string>()` → return type `string`
- `JwtModuleOptions.signOptions.expiresIn` → type `string | number` nhưng thực tế là template literal type từ `ms`:
  ```ts
  type StringValue = `${number}${'ms'|'s'|'m'|'h'|'d'|'w'|'y'}`
  ```
- Vì `string` **không** assignable cho `StringValue` (union hẹp hơn) → cần `as StringValue` để force

> ⚠️ **Đây là type workaround**, không phải best practice. Cách sạch hơn:
> ```ts
> import type { StringValue } from 'ms';
> 
> expiresIn: (configService.get<string>('JWT_ACCESS_EXPIRES_IN') ?? '15m') as StringValue,
> ```
> Hoặc define config schema với validation (ví dụ Joi) → `configService.get<StringValue>('JWT_ACCESS_EXPIRES_IN')`.

**Tại sao `secret` KHÔNG có `as`?**

```ts
secret: configService.get<string>('JWT_ACCESS_SECRET'),
```

Vì `secret` type là `string | Buffer` → `string` assignable trực tiếp. ✅

**Tại sao 2 chỗ đọc config (ở đây và ở `AuthService.signAccessToken`)?**

Trong `AuthService`:

```ts
return this.jwtService.signAsync(payload, {
  secret: this.configService.get<string>('JWT_ACCESS_SECRET'),
  expiresIn: ...,
});
```

→ **Override lại config** dù `JwtModule` đã set default. Đây là **redundant** — có thể bỏ ở 1 trong 2 chỗ:

**Option A** (recommend): Giữ ở module, bỏ ở service
```ts
// auth.service.ts
return this.jwtService.signAsync(payload);
```

**Option B**: Giữ ở service, module chỉ register base
```ts
// auth.module.ts
JwtModule.register({})  // empty — service tự set
```

Hiện tại code đang làm **cả 2** → nếu 2 chỗ khác nhau (ví dụ service set secret khác) → service thắng. Không bug nhưng **dễ nhầm lẫn**.

---

## 🧬 Phần 2: `providers`

```ts
providers: [AuthService, JwtStrategy, OtpService, SessionService],
```

### Vai trò từng provider

| Provider | Vai trò | Inject gì? |
|---|---|---|
| `AuthService` | Business logic chính | `UserService`, `JwtService`, `ConfigService`, `OtpService`, `SessionService` |
| `JwtStrategy` | Passport strategy verify access token | `ConfigService`, `UserService` |
| `OtpService` | Sinh/verify OTP | `RedisService`, `MailService`, `ConfigService` |
| `SessionService` | Quản lý session trong Redis | `RedisService` |

### ⚠️ `JwtStrategy` — tại sao nằm trong `providers`?

`JwtStrategy` **không phải service** theo nghĩa thông thường — nó là **Passport strategy**. Nhưng NestJS yêu cầu:

- Đăng ký là provider để DI resolve dependency (`ConfigService`, `UserService`)
- Khi provider được instantiate, constructor `super({...})` gọi `passport.use()` → đăng ký strategy
- `JwtAuthGuard` lookup strategy theo tên `'jwt'` (set ở `PassportStrategy(Strategy, 'jwt')`)

### 🔁 Chuỗi DI khi app khởi động

```
1. NestJS scan AuthModule
2. Thấy providers → tạo instance theo thứ tự dependency
3. AuthService cần UserService → lookup từ UserModule (đã import)
4. AuthService cần JwtService → lookup từ JwtModule
5. AuthService cần ConfigService → lookup từ ConfigModule (global)
6. AuthService cần OtpService, SessionService → cùng module
7. JwtStrategy cần ConfigService + UserService → OK
8. OtpService cần RedisService + MailService → lookup từ RedisModule + MailModule (đều @Global)
9. Khởi tạo xong → controller sẵn sàng nhận request
```

> 💡 **Tại sao `RedisService` và `MailService` không cần import?**  
> Vì `RedisModule` và `MailModule` đều có `@Global()` decorator → provider của chúng available ở **mọi module** mà không cần import.

### 🎯 Tại sao KHÔNG có `JwtAuthGuard`, `RolesGuard` trong providers?

```ts
// Không có trong providers
JwtAuthGuard
RolesGuard
```

Vì 2 guard này **stateless** — không giữ state giữa các request. NestJS hỗ trợ **instantiate tự động** khi dùng `@UseGuards(JwtAuthGuard)`:

- Nếu guard không cần inject gì phức tạp → NestJS tự tạo
- Nếu cần inject → thêm vào providers

> ⚠️ `RolesGuard` có inject `Reflector`:
> ```ts
> constructor(private reflector: Reflector) { }
> ```
> `Reflector` là provider có sẵn toàn cục (từ `@nestjs/core`) → **không cần khai báo trong providers** vì NestJS tự resolve.
> 
> **Nhưng** nếu muốn dùng `RolesGuard` như **global guard** (qua `APP_GUARD`) → phải đăng ký ở module root (xem `AppModule`).

---

## 🎮 Phần 3: `controllers`

```ts
controllers: [AuthController],
```

**Vai trò**: đăng ký `AuthController` → NestJS scan các `@Post`, `@Get`... để build route table.

**Route prefix**: `@Controller('auth')` + global prefix `api` (set trong `main.ts`) → tất cả route có base `/api/auth`.

**Bảng route sau khi build**:

| Method | Path | Handler |
|---|---|---|
| POST | `/api/auth/login` | `login()` |
| POST | `/api/auth/register/send-otp` | `registerSendOtp()` |
| POST | `/api/auth/register/verify-otp` | `registerVerifyOtp()` |
| POST | `/api/auth/otp/resend` | `resendOtp()` |
| POST | `/api/auth/refresh` | `refresh()` |
| POST | `/api/auth/logout` | `logout()` |
| POST | `/api/auth/logout-all` | `logoutAll()` |
| GET | `/api/auth/profile` | `getProfile()` |
| GET | `/api/auth/sessions` | `listSessions()` |

---

## 📤 Phần 4: `exports`

```ts
exports: [AuthService],
```

### Tại sao export `AuthService`?

Để module khác có thể inject. **Hiện tại** chưa có module nào dùng, nhưng có thể trong tương lai:

```ts
// Ví dụ tương lai: module Order cần verify user
@Module({
  imports: [AuthModule],  // ← để dùng AuthService
})
export class OrderModule {
  constructor(private authService: AuthService) { }
}
```

### Tại sao KHÔNG export các provider khác?

| Provider | Export? | Lý do |
|---|---|---|
| `AuthService` | ✅ | API công khai của module — facade |
| `OtpService` | ❌ | Internal — module khác không nên gọi trực tiếp |
| `SessionService` | ❌ | Internal — chỉ `AuthService` dùng |
| `JwtStrategy` | ❌ | Passport tự quản lý, không ai gọi trực tiếp |

> 🎓 **Nguyên tắc**: Chỉ export cái gì thực sự cần. Export nhiều → module coupling cao → khó refactor.

### ⚠️ `JwtModule` không export

Nếu module khác cần `JwtService` (ví dụ: module `Notification` cần sign token cho websocket) → phải import `JwtModule` ở module đó, **không** lấy qua `AuthModule`.

**Lý do**: `exports: [AuthService]` chỉ export `AuthService`, không transitive export `JwtModule`.

Nếu muốn transitive:
```ts
exports: [AuthService, JwtModule],  // ← re-export
```

---

## 🔗 Dependency Graph

```
                    ┌─────────────────┐
                    │  ConfigModule   │ (global)
                    │  @Global()      │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐   ┌──────────────┐   ┌──────────────────┐
│  RedisModule  │   │  MailModule  │   │   UserModule     │
│  @Global()    │   │  @Global()   │   │  exports: [User] │
└───────┬───────┘   └──────┬───────┘   └────────┬─────────┘
        │                  │                    │
        └──────────────────┼────────────────────┘
                           │
                           ▼
              ┌───────────────────────────┐
              │       AuthModule          │
              │  imports: [UserModule,    │
              │    PassportModule,        │
              │    JwtModule]             │
              │  providers: [AuthService, │
              │    JwtStrategy,           │
              │    OtpService,            │
              │    SessionService]        │
              │  exports: [AuthService]   │
              └───────────┬───────────────┘
                          │
                          ▼
              ┌───────────────────────────┐
              │      AuthController       │
              │   (routes /api/auth/*)    │
              └───────────────────────────┘
```

---

## 🎯 Pattern & Best Practices áp dụng

| Pattern | Áp dụng ở đâu |
|---|---|
| **Module encapsulation** | Chỉ export `AuthService`, giấu internal services |
| **Async config** | `JwtModule.registerAsync` để inject `ConfigService` |
| **Facade** | Controller chỉ gọi `AuthService`, không gọi `OtpService`/`SessionService` trực tiếp |
| **Global module** | `RedisModule`, `MailModule` dùng `@Global()` để tránh import lặp |
| **Type-only import** | `import type { StringValue } from 'ms'` |

---

## ⚠️ Vấn đề cần lưu ý

| # | Vấn đề | Mức độ | Gợi ý |
|---|---|---|---|
| 1 | `JwtModule` set secret/expires, `AuthService` cũng set lại → redundant | 🟡 Thấp | Chọn 1 chỗ, xóa chỗ kia |
| 2 | `as StringValue` là type workaround | 🟡 Thấp | Dùng Joi validation cho env |
| 3 | Chưa export `JwtModule` — nếu module khác cần `JwtService` phải tự import | 🟡 Thấp | Cân nhắc re-export nếu cần |
| 4 | `AuthService` được export nhưng chưa ai dùng | 🟢 Info | OK — chuẩn bị cho tương lai |
| 5 | Không có `JwtModule.registerAsync` validation secret | 🟠 Trung | Throw nếu `JWT_ACCESS_SECRET` undefined |

### 💡 Fix cho vấn đề #5

```ts
useFactory: (configService: ConfigService) => {
  const secret = configService.get<string>('JWT_ACCESS_SECRET');
  if (!secret) {
    throw new Error('JWT_ACCESS_SECRET is not set');
  }
  return {
    secret,
    signOptions: {
      expiresIn: configService.get<string>('JWT_ACCESS_EXPIRES_IN', '15m') as StringValue,
    },
  };
},
```

**Tại sao quan trọng?** Nếu thiếu env → JWT sign với secret `undefined` → mọi token đều invalid hoặc (tệ hơn) accept token của attacker.

---

## 📌 Tổng kết

`auth.module.ts` là file **wiring** — đọc để hiểu:

1. **Phụ thuộc**: `AuthModule` cần `UserModule` (cho `UserService`), `PassportModule` (cho strategy), `JwtModule` (cho `JwtService`).
2. **Provider**: 4 provider — 1 service chính (`AuthService`), 2 internal (`OtpService`, `SessionService`), 1 strategy (`JwtStrategy`).
3. **API**: chỉ export `AuthService` cho module khác dùng.
4. **Global**: `RedisModule` + `MailModule` không cần import nhờ `@Global()`.

Nếu muốn hiểu sâu hơn về NestJS module system → xem note [[NestJS-Module-System]] (cần tạo).

---

## 🔗 Related
- [[Auth-Module]]
- [[Auth-Controller-Explained]]
- [[Auth-Service-Explained]]
- [[User-Module]]
- [[Configuration]]