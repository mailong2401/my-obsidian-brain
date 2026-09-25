---
tags: [devops, docker]
---

# Docker

## docker-compose.yml

### Services
| Service | Image | Port | Note |
|---|---|---|---|
| postgres | postgres:16-alpine | 5432 | healthcheck pg_isready |
| redis | redis:7-alpine | 6379 | requirepass, appendonly |
| mailhog | mailhog/mailhog | 1025 (SMTP), 8025 (UI) | |

### Volumes
- `postgres_data`
- `redis_data`

## Dockerfile (multi-stage)
1. **builder**: node:22-alpine, corepack enable, `pnpm install --frozen-lockfile`, `pnpm build`
2. **runner**: node:22-alpine, `pnpm install --prod`, copy `dist`

## Commands
```bash
docker compose up -d
docker compose logs -f
docker compose down
docker compose down -v  # xóa volume