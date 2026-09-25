---
tags: [security, rate-limit]
---

# Rate Limiting

## 🏗️ Kiến trúc
- Global: `ThrottlerModule` với Redis storage (`@nest-lab/throttler-storage-redis`)
- Guard: `UserThrottlerGuard` (custom, `APP_GUARD`)
- Per-route: `@Throttle({ default: { ttl, limit } })`

## 🎯 Tracker
`UserThrottlerGuard.getTracker`:
1. Nếu có `Authorization: Bearer` → decode JWT (⚠️ **không verify**)
2. Nếu `role === 'admin'` → `admin:{sub}`
3. Nếu có `sub` → `user:{sub}`
4. Fallback → `ip:{req.ip}`

## 📊 Bảng limit hiện tại

| Route | TTL | Limit |
|---|---|---|
| `POST /auth/login` | 15m | 5 |
| `POST /auth/register/send-otp` | 1h | 3 |
| `POST /auth/register/verify-otp` | 15m | 10 |
| `POST /auth/otp/resend` | 1h | 5 |
| `POST /auth/refresh` | 1m | 20 |
| `POST /auth/logout` | 1m | 10 |
| `POST /auth/logout-all` | 1h | 5 |
| `GET /auth/profile` | 1m | 60 |
| `GET /auth/sessions` | 1m | 30 |
| `POST /users` (admin) | 1m | 20 |
| `GET /users` (admin) | 1m | 100 |
| `GET /users/me` | 1m | 60 |
| `GET /users/:id` (admin) | 1m | 100 |
| `PATCH /users/me` | 1m | 10 |
| `PATCH /users/:id` (admin) | 1m | 30 |
| `PATCH /users/:id/role` | 1m | 10 |
| `DELETE /users/:id` | 1m | 20 |
| Default | 1m | 100 |

## ⚠️ Vấn đề
- Decode JWT không verify → attacker fake `sub` để có bucket riêng
- Cần verify bằng `JwtService.verifyAsync` với secret

## ✅ TODO
- [ ] Verify JWT trước khi tin `sub`
- [ ] Thêm `@SkipThrottle()` cho `/health`
- [ ] Log chi tiết khi vượt limit (đã có)

## 🔗 Related
- [[Security]]
- [[Auth-Module]]