---
tags: [module, auth]
dependencies: [User-Module, Redis-Infra, Mail-Infra]
---

# Auth Module

## 📁 Files
- `auth.controller.ts` — HTTP endpoints [[Auth-Controller-Explained]]
- `auth.service.ts` — business logic [[Auth-Service-Explained]]
- `auth.module.ts` — wiring [[Auth-Module-Explained]]
- `services/otp.service.ts` — OTP generate/verify [[Otp-Service-Explained]]
- `services/session.service.ts` — session per device [[Session-Service-Explained]]
- `strategies/jwt.strategy.ts` — passport strategy [[Jwt-Strategy-Explained]]
- `guards/jwt-auth.guard.ts`
- `guards/roles.guard.ts`
- `decorators/current-user.decorator.ts`
- `decorators/roles.decorator.ts`
- `dto/login.dto.ts`, `register.dto.ts`, `verify-otp.dto.ts`, `resend-otp.dto.ts`

## 🔑 Endpoints
Xem [[Auth-Endpoints]]

## 🧠 Logic chính

### OtpService
- `sendOtp(email, purpose)`:
  - Check cooldown key `otp:cooldown:{purpose}:{email}` → 429 nếu còn
  - Generate OTP bằng `crypto.randomInt`
  - Lưu payload JSON vào `otp:{purpose}:{email}` với TTL
  - Reset attempts, set cooldown 60s
  - Gửi mail
- `verifyOtp(email, otp, purpose)`:
  - `incrWithTtl(attemptsKey)` — atomic
  - Nếu > MAX_ATTEMPTS (5) → xóa OTP, throw 400
  - So sánh OTP → sai thì báo số lần còn lại
  - Đúng → xóa OTP + attempts

### SessionService
- `createSession(userId, deviceId, ua, ip)` → sinh refreshToken random 48 bytes, hash SHA-256, lưu:
  - `session:{userId}:{deviceId}` → SessionData (TTL 7d)
  - `device:{userId}:{deviceId}` → "1"
  - `family:{familyId}` → {userId, deviceId}
  - `refresh:{token}` → {userId, deviceId}
- `rotateSession` → tạo refreshToken mới, update hash, lưu mapping mới
- `verifyRefreshToken` → so sánh hash
- `revokeSession` → xóa session + device + family
- `revokeAllUserSessions` → scan `session:{userId}:*` + `device:{userId}:*`

### AuthService
- `registerWithOtp` → create user + send OTP
- `verifyRegisterOtp` → verify OTP + markVerified + create session
- `login` → check password + isVerified → create session
- `refreshTokens` → verify hash → nếu fail → revoke ALL (token reuse) → rotate → sign access token mới
- `logout` / `logoutAll`
- `issueSessionAndRespond` → private helper
- `setAuthCookies` → httpOnly, secure ở prod, sameSite strict, path `/api/auth`

## ⚠️ Vấn đề
- Register tạo user trước verify → user rác
- `revokeSession` không xóa `refresh:{token}` mapping ngay
- `UserThrottlerGuard` decode JWT không verify
- Chưa có forgot password / login 2FA hoàn chỉnh

## ✅ TODO
- [ ] Refactor pending registration
- [ ] Forgot password flow
- [ ] Login 2FA
- [ ] Test token reuse
- [ ] Xóa `refresh:{token}` khi revoke

## 🔗 Related
- [[Security]]
- [[Rate-Limiting]]
- [[Redis-Keys]]
- [[Auth-Endpoints]]