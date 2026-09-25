---
tags: [infra, mail]
---

# Mail Infrastructure

## 📁 Files
- `mail.module.ts` — `@Global()`, `MailerModule.forRootAsync`
- `mail.service.ts` — `sendOtpEmail`
- `templates/otp-verification.hbs` — Handlebars template

## 🧠 Logic
- Dev + Mailhog (`localhost:1025`): không auth, `ignoreTLS`, `rejectUnauthorized: false`
- Prod: dùng `MAIL_USER` / `MAIL_PASSWORD`, `secure` theo env
- Template context: `otp`, `purpose`, `ttlMinutes`, `year`

## 📧 Subject map
| purpose | subject |
|---|---|
| register | Xác thực tài khoản của bạn |
| forgot_password | Đặt lại mật khẩu |
| login_2fa | Mã xác thực đăng nhập |

## ⚠️ Vấn đề
- Gửi mail đồng bộ trong request → chậm
- Chưa có retry
- Chưa có queue

## ✅ TODO
- [ ] BullMQ `MailProcessor`
- [ ] Retry 3 lần exponential backoff
- [ ] Template cho welcome, reset password success

## 🔗 Related
- [[Auth-Module]]
- [[Phase-3-Infra]]