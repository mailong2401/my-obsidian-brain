---
tags: [index, overview]
created: 2026-01-01
status: active
---

# E-Commerce Backend — Overview

## 🎯 Mục tiêu dự án
Xây dựng backend E-Commerce với NestJS, tập trung vào **bảo mật auth** (session-per-device, refresh token rotation, OTP, rate limiting) và khả năng mở rộng.

## 🧱 Tech Stack
- **Framework**: NestJS 11 + TypeScript 5.7
- **Database**: PostgreSQL 16 (TypeORM)
- **Cache**: Redis 7 (ioredis)
- **Auth**: JWT access (15m) + Opaque refresh (7d) qua httpOnly cookie
- **Mail**: Nodemailer + Handlebars + Mailhog (dev)
- **Rate limit**: @nestjs/throttler + Redis storage
- **Docs**: Swagger tại `/docs`

## 🗂️ Cấu trúc source
Xem [[Architecture#Cấu trúc thư mục]]

## 🧩 Modules
- [[Auth-Module]]
- [[User-Module]]
- [[Redis-Infra]]
- [[Mail-Infra]]

## 🔐 Cross-cutting
- [[Security]]
- [[Rate-Limiting]]
- [[Error-Handling]]
- [[Configuration]]

## 📊 Data
- [[Database-Schema]]
- [[Redis-Keys]]

## 🚀 Roadmap
- [[Task-List]]
- [[Phase-0-Foundation]]
- [[Phase-1-Auth]]

## ⚠️ Vấn đề đang mở
- [ ] `typeorm` version trong package.json sai (^1.1.1)
- [ ] `UserThrottlerGuard` decode JWT không verify
- [ ] Register tạo user trước khi verify OTP
- [ ] Chưa có migration
- [ ] Refresh token revoke chưa tức thời

## 📌 Quick links
- [[Daily-Log]]
- [[Deployment]]