---
tags: [module, user]
---

# User Module

## 📁 Files
- `user.entity.ts` — TypeORM entity
- `user.service.ts`
- `user.controller.ts`
- `user.module.ts`
- `dto/create-user.dto.ts`, `update-profile.dto.ts`, `update-user.dto.ts`

## 🗄️ Entity User
| Field | Type | Note |
|---|---|---|
| id | uuid | PK |
| firstName | varchar(100) | |
| lastName | varchar(100) | |
| email | varchar(255) | unique |
| username | varchar(50) | unique |
| password | varchar(255) | `select: false`, nullable |
| isVerified | boolean | default false |
| phone | varchar(20) | nullable |
| role | enum | `user` / `admin` |
| avatar | varchar(500) | nullable |
| createdAt | timestamp | |
| updatedAt | timestamp | |

## 🔑 Endpoints
Xem [[User-Endpoints]]

## 🧠 Logic chính
- `create` → check email/username unique → hash bcrypt (10 rounds)
- `findByEmailWithPassword` → query builder `addSelect('user.password')`
- `update` → check unique nếu đổi email/username → hash password nếu có
- `updateProfile` → chỉ cho phép firstName, lastName, phone, avatar
- `updateRole` → admin only
- `markVerified` → set `isVerified = true`

## ⚠️ Vấn đề
- Chưa có đổi mật khẩu
- Chưa có soft delete
- `findAll` chưa pagination
- Chưa upload avatar

## ✅ TODO
- [ ] `PATCH /users/me/password`
- [ ] Pagination + filter cho `GET /users`
- [ ] Soft delete (`@DeleteDateColumn`)
- [ ] Upload avatar (S3/MinIO)

## 🔗 Related
- [[Database-Schema]]
- [[User-Endpoints]]