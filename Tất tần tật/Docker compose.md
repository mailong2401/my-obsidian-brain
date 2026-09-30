Nếu học **Docker Compose theo kiểu Senior Backend**, đừng hiểu nó đơn giản là “file để chạy nhiều container”. Hãy hiểu nó là **cách mô tả và điều phối một application stack gồm nhiều service bằng cấu hình khai báo (declarative configuration)**.

## 1. Docker Compose thực chất là gì?

Ví dụ project e-commerce của bạn có:

```text
                    ┌──────────────┐
                    │    Nginx     │
                    │     :80      │
                    └──────┬───────┘
                           │
                ┌──────────┴──────────┐
                ↓                     ↓
        ┌──────────────┐      ┌──────────────┐
        │   Next.js    │      │    NestJS    │
        │    :3000     │      │    :8080     │
        └──────────────┘      └──────┬───────┘
                                    │
                       ┌────────────┴────────────┐
                       ↓                         ↓
                ┌──────────────┐         ┌──────────────┐
                │  PostgreSQL  │         │    Redis     │
                │     :5432    │         │     :6379    │
                └──────────────┘         └──────────────┘
```

Nếu không có Compose, bạn phải tự:

```bash
docker network create app-network

docker run ...
docker run ...
docker run ...
docker run ...
```

Rồi tự cấu hình:

- network
    
- environment
    
- volume
    
- port
    
- dependency
    
- restart policy
    
- healthcheck
    
- image/build
    
- container name...
    

Docker Compose cho phép bạn **khai báo toàn bộ kiến trúc đó trong một file**.

Ví dụ:

```yaml
services:
  backend:
    build: ./ecom-be
    ports:
      - "8080:8080"
    depends_on:
      - postgres
      - redis

  frontend:
    build: ./ecom-fe
    ports:
      - "3000:3000"

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: ecommerce

  redis:
    image: redis:7-alpine
```

Sau đó:

```bash
docker compose up -d
```

Compose sẽ dựng cả stack.

---

# 2. Cách Senior nhìn Docker Compose

Junior thường nghĩ:

> `docker-compose.yml` = file để chạy Docker.

Senior sẽ nghĩ:

> `compose.yaml` = **deployment topology của một môi trường application**.

Tức là nó mô tả:

```text
Application
│
├── frontend
│
├── backend
│
├── database
│
├── cache
│
├── message broker
│
└── reverse proxy
```

và quan hệ giữa chúng.

Ví dụ:

```yaml
services:
  backend:
    depends_on:
      - postgres
      - redis
      - rabbitmq
```

Có thể đọc thành:

```text
backend
   │
   ├── cần PostgreSQL
   ├── cần Redis
   └── cần RabbitMQ
```

Đây chính là tư duy **service orchestration**.

---

# 3. `services` là gì?

Đây là phần quan trọng nhất.

```yaml
services:
  backend:
  frontend:
  postgres:
  redis:
```

Mỗi `service` đại diện cho **một loại container**.

Ví dụ:

```yaml
services:

  backend:
    build: ./ecom-be

  frontend:
    build: ./ecom-fe

  postgres:
    image: postgres:16-alpine

  redis:
    image: redis:7-alpine
```

Compose có thể tạo:

```text
backend  → container
frontend → container
postgres → container
redis    → container
```

Điểm quan trọng:

> **Service không phải container.**

Service là **định nghĩa**.

Container là **instance được tạo ra từ định nghĩa đó**.

Ví dụ:

```yaml
services:
  backend:
    image: my-backend
```

Bạn có thể hiểu:

```text
service: backend
        ↓
container: project-backend-1
```

Nếu scale:

```bash
docker compose up --scale backend=3
```

thì:

```text
backend service
      │
      ├── backend-1
      ├── backend-2
      └── backend-3
```

Đây là một điểm rất quan trọng khi tư duy production.

---

# 4. `image` vs `build`

Đây là thứ bạn **phải phân biệt rất rõ**.

## `image`

```yaml
postgres:
  image: postgres:16-alpine
```

Nghĩa là:

> Lấy image đã tồn tại từ registry và tạo container.

Ví dụ:

```text
Docker Hub
    ↓
postgres:16-alpine
    ↓
container
```

---

## `build`

```yaml
backend:
  build:
    context: ./ecom-be
```

Nghĩa là:

> Lấy source code trong `./ecom-be`, build Docker image.

Ví dụ:

```text
ecom-be/
├── Dockerfile
├── package.json
└── src/
       ↓
docker build
       ↓
backend image
       ↓
backend container
```

Vì vậy:

```yaml
backend:
  build: ./ecom-be

postgres:
  image: postgres:16-alpine
```

có nghĩa:

```text
Backend
→ tự build image từ source

Postgres
→ dùng image có sẵn
```

---

# 5. `ports` cực kỳ quan trọng

Ví dụ:

```yaml
backend:
  ports:
    - "8080:8080"
```

Format:

```text
HOST_PORT:CONTAINER_PORT
```

Nên:

```text
8080:8080
│    │
│    └── port bên trong container
└─────── port trên máy host
```

Bạn truy cập:

```text
localhost:8080
```

Docker chuyển:

```text
Host :8080
     ↓
Container :8080
```

---

## Nhưng đây là chỗ rất nhiều người mới hiểu sai

Giả sử:

```yaml
backend:
  ports:
    - "8080:8080"

postgres:
  ports:
    - "5432:5432"
```

Backend **không nhất thiết phải dùng**:

```env
DB_HOST=localhost
```

Trong Docker Compose, backend nên dùng:

```env
DB_HOST=postgres
```

Tại sao?

Vì Compose tạo network cho các service.

```text
backend
   │
   │ DB_HOST=postgres
   ↓
postgres
```

`postgres` chính là hostname/service name.

---

# 6. Docker Compose Network

Đây là phần bạn nên hiểu theo kiểu backend senior.

Compose thường tạo network riêng:

```text
ecom_default
```

Các container cùng network có thể giao tiếp với nhau bằng service name.

Ví dụ:

```yaml
services:

  backend:
    ...

  postgres:
    ...
```

Backend:

```env
DB_HOST=postgres
DB_PORT=5432
```

Redis:

```env
REDIS_HOST=redis
REDIS_PORT=6379
```

RabbitMQ:

```env
RABBITMQ_HOST=rabbitmq
```

Không cần:

```env
DB_HOST=localhost
```

---

# 7. `localhost` trong Docker rất dễ gây nhầm

Đây là lỗi cực kỳ phổ biến.

Giả sử:

```text
Host
│
├── backend container
│
└── postgres container
```

Trong backend container:

```text
localhost
```

có nghĩa:

> **chính backend container**

Không phải máy host.

Không phải PostgreSQL container.

Do đó:

```env
DB_HOST=localhost
```

sẽ tìm PostgreSQL ở:

```text
backend container
└── localhost:5432
```

Nếu PostgreSQL nằm container khác:

```text
postgres container
```

thì phải:

```env
DB_HOST=postgres
```

---

# 8. `depends_on`

Ví dụ:

```yaml
backend:
  depends_on:
    - postgres
    - redis
```

Ý nghĩa cơ bản:

```text
postgres
redis
   ↓
backend
```

Compose sẽ khởi động dependency trước.

Nhưng Senior phải nhớ:

> `depends_on` **không đồng nghĩa với database đã READY để nhận connection**.

Ví dụ PostgreSQL container đã:

```text
RUNNING
```

nhưng PostgreSQL bên trong vẫn đang:

```text
initializing...
```

Backend lúc đó kết nối:

```text
backend
   ↓
postgres:5432
   ↓
connection refused
```

Do đó production-like Compose nên kết hợp:

```yaml
healthcheck:
```

Ví dụ:

```yaml
postgres:
  image: postgres:16-alpine

  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    timeout: 5s
    retries: 5
```

Sau đó dependency có thể dựa trên health status tùy phiên bản Compose/configuration.

Đây là tư duy:

```text
container started
        ≠
application ready
```

Rất quan trọng.

---

# 9. `environment`

Ví dụ:

```yaml
backend:
  environment:
    NODE_ENV: production
    DB_HOST: postgres
    DB_PORT: 5432
    DB_NAME: ecommerce
```

Đây là environment variables được inject vào container.

Trong NestJS:

```typescript
process.env.DB_HOST
```

sẽ nhận:

```text
postgres
```

---

Nhưng production không nên viết password trực tiếp:

```yaml
environment:
  DB_PASSWORD: mypassword123
```

Có thể dùng:

```yaml
environment:
  DB_PASSWORD: ${DB_PASSWORD}
```

và `.env`:

```env
DB_PASSWORD=super-secret
```

Đồng thời:

```gitignore
.env
```

Không commit secret vào Git.

---

# 10. `volumes` — cực kỳ quan trọng với Database

Container có tính chất:

```text
container
    ↓
có thể bị xóa
```

Nếu PostgreSQL lưu data trực tiếp trong filesystem container:

```text
postgres container
    ↓
database files
```

Xóa container:

```bash
docker rm postgres
```

thì dữ liệu có thể mất.

Vì vậy:

```yaml
postgres:
  image: postgres:16-alpine
  volumes:
    - postgres_data:/var/lib/postgresql/data
```

Docker tạo:

```text
postgres_data
      │
      ↓
/var/lib/postgresql/data
      │
      ↓
PostgreSQL container
```

Container chết:

```text
❌ postgres container
```

Volume vẫn:

```text
✅ postgres_data
```

Container mới mount volume:

```text
new postgres container
        ↓
postgres_data
        ↓
old database data
```

Đây là:

> **container ephemeral, data persistent.**

Một nguyên tắc Docker rất quan trọng.

---

# 11. Named Volume vs Bind Mount

### Named volume

```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data
```

Docker quản lý storage.

Thường phù hợp cho:

```text
PostgreSQL
Redis
production data
```

---

### Bind mount

```yaml
volumes:
  - ./src:/app/src
```

Mount thư mục máy host vào container.

Thường rất tiện cho development:

```text
Host
./src
 ↓
Container
/app/src
```

Sửa code trên host → container nhìn thấy thay đổi.

---

# 12. `restart`

Ví dụ:

```yaml
restart: unless-stopped
```

Nó nói với Docker:

> Nếu container crash thì cố gắng restart.

Các policy phổ biến:

```text
no
always
on-failure
unless-stopped
```

Ví dụ:

```yaml
backend:
  restart: unless-stopped
```

Nếu NestJS crash:

```text
NestJS crash
    ↓
container exit
    ↓
Docker restart
    ↓
NestJS chạy lại
```

Nhưng Senior phải nhớ:

> Restart policy **không thay thế application reliability**.

Nếu code crash liên tục:

```text
crash
 ↓
restart
 ↓
crash
 ↓
restart
 ↓
...
```

Bạn vẫn phải tìm root cause.

---

# 13. `healthcheck`

Một container có thể:

```text
container = running
application = broken
```

Ví dụ Nest:

```text
container running
NestJS process chết
```

Docker cần biết:

```text
service có thực sự healthy không?
```

Ví dụ:

```yaml
backend:
  healthcheck:
    test:
      [
        "CMD",
        "wget",
        "--spider",
        "http://localhost:8080/api/health"
      ]
    interval: 10s
    timeout: 5s
    retries: 5
```

Application có:

```text
GET /api/health
```

Trả:

```json
{
  "status": "ok"
}
```

Docker có thể kiểm tra:

```text
backend
   ↓
GET /api/health
   ↓
200
   ↓
healthy
```

---

# 14. Compose không phải Kubernetes

Đây là kiến thức Senior nên phân biệt.

Docker Compose:

```text
Docker Compose
     ↓
thường dùng cho
     ↓
local development
testing
small deployments
single-host applications
```

Kubernetes:

```text
Kubernetes
     ↓
container orchestration
     ↓
multi-node
scaling
service discovery
rolling deployment
self-healing
etc.
```

Compose không phải Kubernetes thu nhỏ.

Compose tập trung vào:

> **định nghĩa và chạy application stack một cách thuận tiện.**

---

# 15. Docker Compose lifecycle

Ví dụ bạn có:

```yaml
services:
  backend:
    build: ./ecom-be

  postgres:
    image: postgres:16-alpine

  redis:
    image: redis:7-alpine
```

Chạy:

```bash
docker compose up
```

Flow:

```text
compose.yaml
     ↓
parse configuration
     ↓
resolve services
     ↓
create network
     ↓
pull/build images
     ↓
create volumes
     ↓
create containers
     ↓
start containers
```

---

## `docker compose up -d`

```bash
docker compose up -d
```

`-d`:

```text
detached mode
```

Terminal không bị attach vào logs.

---

## Xem container

```bash
docker compose ps
```

---

## Xem logs

```bash
docker compose logs
```

Backend:

```bash
docker compose logs backend
```

Realtime:

```bash
docker compose logs -f backend
```

---

## Dừng

```bash
docker compose stop
```

Container vẫn tồn tại.

---

## Xóa container

```bash
docker compose down
```

Thông thường:

```text
container → remove
network    → remove
```

Volume named thường **không bị xóa** nếu bạn không yêu cầu xóa volumes.

---

## Xóa cả volume

Cẩn thận:

```bash
docker compose down -v
```

Có thể xóa:

```text
postgres_data
redis_data
```

=> Database data có thể mất.

---

# 16. `docker compose up` vs `docker compose start`

Khác nhau:

```bash
docker compose up
```

có thể:

```text
build/pull
create
start
```

Còn:

```bash
docker compose start
```

chỉ start container đã tồn tại.

Ví dụ:

```text
down
 ↓
container bị remove
```

thì:

```bash
docker compose start
```

không giúp bạn tạo lại container.

Phải:

```bash
docker compose up
```

---

# 17. Compose thực sự giải quyết vấn đề gì?

Giả sử e-commerce của bạn:

```text
Next.js
NestJS
PostgreSQL
Redis
RabbitMQ
Nginx
MailHog
```

Không có Compose:

```bash
docker network create ...
docker run postgres ...
docker run redis ...
docker run rabbitmq ...
docker run backend ...
docker run frontend ...
docker run nginx ...
```

Rất nhiều command.

Với Compose:

```bash
docker compose up -d
```

Một command dựng cả stack.

Đó chính là giá trị lớn nhất:

> **Infrastructure as configuration.**

Bạn có thể commit:

```text
compose.yaml
```

vào Git.

Developer khác clone:

```bash
git clone ...
cd project
docker compose up -d
```

và có gần như cùng một môi trường.

---

# 18. Compose của một project Backend production-like

Ví dụ kiến trúc phù hợp với project NestJS của bạn:

```yaml
services:

  backend:
    build:
      context: ./ecom-be
      dockerfile: Dockerfile

    environment:
      NODE_ENV: production
      DB_HOST: postgres
      DB_PORT: 5432
      REDIS_HOST: redis
      REDIS_PORT: 6379

    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy

    restart: unless-stopped

  postgres:
    image: postgres:16-alpine

    environment:
      POSTGRES_DB: ecommerce
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    volumes:
      - postgres_data:/var/lib/postgresql/data

    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine

    volumes:
      - redis_data:/data

    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

Đây đã thể hiện khá nhiều tư duy production:

```text
                Compose
                   │
       ┌───────────┼────────────┐
       ↓           ↓            ↓
    Backend     PostgreSQL     Redis
       │           │            │
       │           │            │
       └───────────┼────────────┘
                   │
                Network
                   │
          ┌────────┴────────┐
          ↓                 ↓
     postgres_data      redis_data
          │                 │
       persistent        persistent
```

---

# 19. Nhưng Senior phải hiểu một tầng sâu hơn

Docker Compose **không làm cho application trở thành production-ready**.

Ví dụ:

```yaml
restart: always
```

không có nghĩa:

> hệ thống đã có HA.

Có:

```yaml
replicas: 3
```

không tự động có nghĩa:

> hệ thống scale production tốt.

Có:

```yaml
depends_on:
```

không có nghĩa:

> distributed system đã reliable.

Compose chỉ là **một lớp infrastructure orchestration**.

Application vẫn phải xử lý:

```text
Database transaction
Connection pool
Retry
Timeout
Circuit breaker
Idempotency
Caching
Rate limiting
Authentication
Authorization
Logging
Monitoring
Health check
Graceful shutdown
Migration
Backup
Recovery
```

Đây mới là phần bạn đang hướng tới khi học Backend Senior.

---

# 20. Đặc biệt: Compose + NestJS của bạn

Với project hiện tại, bạn nên hình dung:

```text
                   Internet
                      │
                      ↓
                   Nginx
                    :80
                      │
             ┌────────┴────────┐
             ↓                 ↓
         Next.js             NestJS
          :3000               :8080
                                │
              ┌─────────────────┼────────────────┐
              ↓                 ↓                ↓
          PostgreSQL          Redis           RabbitMQ
            :5432             :6379             :5672
              │                 │                │
              ↓                 ↓                ↓
          persistent          cache          message queue
             data
```

Request:

```text
Browser
   ↓
Nginx
   ↓
Next.js
   ↓
/api
   ↓
NestJS
   ↓
PostgreSQL
```

Một request khác:

```text
NestJS
   ↓
Redis
   ↓
cache
```

Hoặc:

```text
NestJS
   ↓
RabbitMQ
   ↓
Consumer
   ↓
Email / Payment / Notification
```

**Đây mới là cách nên tư duy Docker Compose ở mức backend senior:** không học thuộc YAML trước, mà hiểu **service topology → network → persistence → startup → health → failure → configuration → lifecycle**.

Nếu bạn nắm được 8 thứ này:

1. **Service**
    
2. **Container**
    
3. **Image**
    
4. **Network**
    
5. **Volume**
    
6. **Environment**
    
7. **Healthcheck / dependency**
    
8. **Lifecycle**
    

thì bạn đã có nền tảng Compose rất chắc để chuyển sang **Docker production → CI/CD → AWS/VPS → Kubernetes**.