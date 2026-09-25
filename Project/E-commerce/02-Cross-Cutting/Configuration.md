---
tags: [config]
---

# Configuration

## 📁 Files
- `.env.example` — template
- `ConfigModule.forRoot({ isGlobal: true })` trong `AppModule`

## 🔑 Biến môi trường

### App
| Key | Default | Ghi chú |
|---|---|---|
| NODE_ENV | development | |
| BACKEND_PORT | 8080 | |
| PORT | 8080 | dùng trong `main.ts` |
| CORS_ORIGIN | http://localhost:3000 | comma-separated |

### Postgres
| Key | Default |
|---|---|
| DB_HOST | postgres |
| DB_PORT | 5432 |
| DB_USERNAME | postgres |
| DB_PASSWORD | admin |
| DB_DATABASE | ecommerce |

### Redis
| Key | Default |
|---|---|
| REDIS_HOST | redis |
| REDIS_PORT | 6379 |
| REDIS_PASSWORD | redis123 |
| REDIS_DB | 0 |

### JWT
| Key | Default |
|---|---|
| JWT_ACCESS_SECRET | (bắt buộc) |
| JWT_ACCESS_EXPIRES_IN | 15m |
| JWT_REFRESH_SECRET | (không dùng vì opaque) |
| JWT_REFRESH_EXPIRES_IN | 7d |

### Mail
| Key | Default |
|---|---|
| MAIL_HOST | mailhog |
| MAIL_PORT | 1025 |
| MAIL_SECURE | false |
| MAIL_USER | |
| MAIL_PASSWORD | |
| MAIL_FROM | noreply@ecommerce.local |
| MAIL_FROM_NAME | E-Commerce |

### OTP
| Key | Default |
|---|---|
| OTP_LENGTH | 6 |
| OTP_TTL_SECONDS | 300 |
| OTP_MAX_ATTEMPTS | 5 |
| OTP_RESEND_COOLDOWN | 60 |

## ⚠️ Lưu ý
- `BACKEND_PORT` trong `.env.example` nhưng code dùng `PORT`
- `JWT_REFRESH_SECRET` không còn dùng → có thể xóa

## 🔗 Related
- [[Environment]]
- [[Deployment]]