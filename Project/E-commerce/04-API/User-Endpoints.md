---
tags: [api, user]
---

# User Endpoints

Base: `/api/users`
Tất cả đều yêu cầu `JwtAuthGuard` + `RolesGuard`

## POST `/` (admin)
- Body: `CreateUserDto`
- Rate: 20 / 1m
- Response 201: `User`

## GET `/` (admin)
- Rate: 100 / 1m
- Response: `User[]`

## GET `/me`
- Rate: 60 / 1m
- Response: `User` (không password)

## GET `/:id` (admin)
- Param: UUID
- Rate: 100 / 1m
- Response: `User`

## PATCH `/me`
- Body: `UpdateProfileDto` (`firstName`, `lastName`, `phone`, `avatar`)
- Rate: 10 / 1m
- Response: `User`

## PATCH `/:id` (admin)
- Body: `UpdateUserDto` (partial CreateUserDto)
- Rate: 30 / 1m
- Response: `User`

## PATCH `/:id/role` (admin)
- Body: `{ role: 'user' | 'admin' }`
- Rate: 10 / 1m
- Response: `User`

## DELETE `/:id` (admin)
- Rate: 20 / 1m
- Response 204

## 🚧 Chưa có
- `PATCH /me/password`
- `POST /me/avatar` (upload)
- Pagination cho `GET /`

## 🔗 Related
- [[User-Module]]
- [[Rate-Limiting]]