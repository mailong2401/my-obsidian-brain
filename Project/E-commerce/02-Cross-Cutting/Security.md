---
tags: [security]
---

# Security

## 🔐 Auth strategy
- **Access token**: JWT HS256, 15 phút, chứa `sub`, `email`, `role`
- **Refresh token**: opaque random 48 bytes (hex), hash SHA-256 lưu Redis, TTL 7d
- **Cookie**: `refreshToken` (httpOnly), `deviceId` (readable), `sameSite: strict`, `secure` ở prod, path `/api/auth`
- **Device ID**: bắt buộc header `X-Device-Id` (min 8 ký tự) — xem `DeviceId` decorator

## 🔄 Token rotation & reuse detection
- Mỗi lần refresh → tạo refresh token mới, update hash trong session
- Nếu verify fail (token cũ bị dùng lại) → **revoke toàn bộ session của user** + clear cookie
- Log warning `Possible token reuse`

## 🛡️ OTP
- `crypto.randomInt` (không dùng `Math.random`)
- TTL 5 phút, max 5 attempts, cooldown 60s
- Scope theo `purpose` → OTP register không dùng được cho forgot_password

## 🚦 Rate limiting
Xem [[Rate-Limiting]]

## ✅ Validation
- Global `ValidationPipe` với `whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`
- DTO dùng `class-validator` + `@Transform` trim/lowercase

## ⚠️ Lỗ hổng cần fix
- [ ] `UserThrottlerGuard` decode JWT không verify → bypass per-user limit
- [ ] Register tạo user trước verify OTP
- [ ] Cần CSRF token nếu dùng cookie cho action state-changing (hiện dùng sameSite strict nên tạm ổn)
- [ ] Cần rotate JWT secret định kỳ
- [ ] Cần blacklist access token khi logout (hiện access token vẫn valid tới 15m)

## 🔗 Related
- [[Auth-Module]]
- [[Rate-Limiting]]
- [[Redis-Keys]]