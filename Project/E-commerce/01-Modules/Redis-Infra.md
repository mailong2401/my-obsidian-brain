---
tags: [infra, redis]
---

# Redis Infrastructure

## 📁 Files
- `redis.module.ts` — `@Global()` module, `forRootAsync`
- `redis.service.ts` — wrapper với helper methods

## 🧰 API của RedisService
| Method | Mô tả |
|---|---|
| `set(key, value, ttl?)` | SET hoặc SETEX |
| `get(key)` | GET |
| `del(...keys)` | DEL nhiều key |
| `exists(key)` | EXISTS → bool |
| `incr(key)` | INCR |
| `expire(key, ttl)` | EXPIRE |
| `ttl(key)` | TTL (giây) |
| `setJson<T>(key, value, ttl?)` | SET JSON.stringify |
| `getJson<T>(key)` | GET + JSON.parse |
| `incrWithTtl(key, ttl)` | INCR + EXPIRE nếu lần đầu (atomic-ish) |
| `scanKeys(pattern, count)` | SCAN không block Redis |

## 🔑 Key patterns
Xem [[Redis-Keys]]

## ⚙️ Cấu hình
- Host/Port/Password/DB từ env
- Retry strategy: `min(times * 50, 2000)` ms
- `maxRetriesPerRequest: 3`

## ⚠️ Vấn đề
- `scanKeys` dùng SCAN nhưng `del(...keys)` với mảng lớn có thể block → cân nhắc chunk
- Chưa có connection event logging (ready, error, reconnecting)

## ✅ TODO
- [ ] Log connection events
- [ ] Chunk `del` khi > 1000 keys
- [ ] Helper `revokeByPattern(pattern)` atomic hơn

## 🔗 Related
- [[Redis-Keys]]
- [[Session Service|Auth-Module]]
- [[Rate-Limiting]]