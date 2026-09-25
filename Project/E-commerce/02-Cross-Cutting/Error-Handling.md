---
tags: [error]
---

# Error Handling

## 🎯 Convention
- NestJS `HttpException` subclasses
- Custom: `TooManyRequestsException` (429)
- Message tiếng Việt cho user-facing (rate limit, OTP)

## 📋 Bảng lỗi

| Tình huống | Exception | Status |
|---|---|---|
| Sai credentials | `UnauthorizedException('Invalid credentials')` | 401 |
| Chưa verify | `UnauthorizedException('Tài khoản chưa được xác thực...')` | 401 |
| Thiếu device ID | `BadRequestException('Missing or invalid X-Device-Id header')` | 400 |
| OTP sai | `BadRequestException('Mã OTP không đúng. Còn N lần thử.')` | 400 |
| OTP hết hạn | `BadRequestException('Mã OTP đã hết hạn hoặc không tồn tại')` | 400 |
| Quá nhiều lần OTP | `BadRequestException('Bạn đã nhập sai quá nhiều lần...')` | 400 |
| Cooldown OTP | `TooManyRequestsException` | 429 |
| Token reuse | `ForbiddenException('Invalid refresh token. All sessions revoked.')` | 403 |
| Email trùng | `ConflictException('Email already exists')` | 409 |
| User không tồn tại | `NotFoundException` | 404 |
| Thiếu role | `ForbiddenException('Requires one of...')` | 403 |

## ⚠️ TODO
- [ ] Global exception filter để chuẩn hóa response
- [ ] Không leak stack trace ở prod
- [ ] i18n cho message

## 🔗 Related
- [[Auth-Module]]
- [[User-Module]]