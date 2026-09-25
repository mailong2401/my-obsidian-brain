---
tags: [api, auth]
---

# Auth Endpoints

Base: `/api/auth`

## POST `/login`
- Body: `{ email, password }`
- Headers: `X-Device-Id` (bắt buộc)
- Rate: 5 / 15m
- Response 200: `{ user, accessToken }` + set cookies `refreshToken`, `deviceId`
- Errors: 401 (sai credentials / chưa verify), 400 (thiếu device)

## POST `/register/send-otp`
- Body: `RegisterDto`
- Rate: 3 / 1h
- Response 201: `{ message }`
- Side effect: tạo user + gửi OTP

## POST `/register/verify-otp`
- Body: `{ email, otp, purpose: 'register' }`
- Headers: `X-Device-Id`
- Rate: 10 / 15m
- Response 200: `{ user, accessToken }` + cookies

## POST `/otp/resend`
- Body: `{ email, purpose }`
- Rate: 5 / 1h
- Response 200: `{ message }`
- Errors: 429 nếu cooldown

## POST `/refresh`
- Cookie: `refreshToken`
- Rate: 20 / 1m
- Response 200: `{ accessToken }` + cookie mới
- Errors: 401 thiếu/sai token, 403 token reuse → revoke all

## POST `/logout`
- Auth: Bearer
- Cookie: `deviceId`
- Rate: 10 / 1m
- Response 200: `{ message }` + clear cookies

## POST `/logout-all`
- Auth: Bearer
- Rate: 5 / 1h
- Response 200: `{ message }`

## GET `/profile`
- Auth: Bearer
- Rate: 60 / 1m
- Response 200: user không password

## GET `/sessions`
- Auth: Bearer
- Rate: 30 / 1m
- Response 200: array session (không có `refreshTokenHash`)

## 🚧 Chưa có
- `POST /forgot-password/send-otp`
- `POST /forgot-password/verify-otp`
- `POST /forgot-password/reset`
- `POST /login/verify-2fa`

## 🔗 Related
- [[Auth-Module]]
- [[Rate-Limiting]]