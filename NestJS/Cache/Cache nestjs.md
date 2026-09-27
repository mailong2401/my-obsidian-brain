# 1. Cache là gì?

**Caching** là kỹ thuật lưu tạm dữ liệu ở nơi có tốc độ truy cập nhanh hơn để giảm việc truy cập nguồn dữ liệu gốc.

Ví dụ bình thường:

```
Client
  |
  v
NestJS API
  |
  v
PostgreSQL
  |
  v
Trả dữ liệu
```

Mỗi lần gọi API:

```
GET /products/123

NestJS
   |
   |
   v
SELECT * FROM products WHERE id=123
```

Nếu 10.000 người cùng xem sản phẩm:

```
10.000 query PostgreSQL
```

Database bị quá tải.

---

Có cache:

```
Client
 |
 v
NestJS
 |
 v
Redis Cache
 |
 +---- Có dữ liệu
 |
 return luôn
```

Database chỉ bị gọi khi cache không có.

---

# 2. Cache hoạt động như thế nào?

Có một khái niệm quan trọng:

## Cache Hit

Cache có dữ liệu:

```
Request
 |
 v
Redis

product:123

{
 id:123,
 name:"iPhone"
}

return
```

Không cần DB.

---

## Cache Miss

Cache không có:

```
Request
 |
 v
Redis

Không có

 |
 v

Database

SELECT product
 |
 v

Lưu Redis

 |
 v

Return
```

Flow:

```
          Request
             |
             v
        Check Cache
             |
     +-------+-------+
     |               |
   HIT             MISS
     |               |
     |               v
     |             DB
     |               |
     |               v
     |          Save Cache
     |
     v
  Response
```

---

# 3. Các loại Cache phổ biến

## 1. Local Memory Cache

Lưu trong RAM của Node.js.

Ví dụ:

```ts
const cache = new Map();

cache.set(
 "product:1",
 {
  name:"Laptop"
 }
)
```

Ưu:

- cực nhanh
    
- đơn giản
    

Nhược:

- restart server mất dữ liệu
    
- nhiều server không đồng bộ
    

Ví dụ:

```
             Load Balancer

        /                 \

   Server A              Server B

Memory cache          Memory cache

product:1             không có
```

Không phù hợp production lớn.

---

# 2. Redis Cache (phổ biến nhất)

Kiến trúc:

```
NestJS
 |
 |
Redis
 |
 |
PostgreSQL
```

Redis chạy ngoài:

```
Docker

redis:7-alpine
```

Redis lưu:

```
key:value


product:123

{
 id:123,
 name:"Laptop"
}
```

---

# 4. NestJS Cache Module

NestJS có sẵn:

```
@nestjs/cache-manager
```

Cài:

```bash
npm install @nestjs/cache-manager cache-manager
```

---

Import:

```ts
@Module({
 imports:[
  CacheModule.register({
    ttl:60000,
    max:100
  })
 ]
})
export class AppModule{}
```

Ý nghĩa:

```ts
ttl:60000
```

Cache sống 60 giây.

---

# 5. Inject Cache trong Service

Ví dụ ProductService:

```ts
@Injectable()
export class ProductService {


constructor(
 @Inject(CACHE_MANAGER)
 private cacheManager: Cache
){}


}
```

---

Lấy cache:

```ts
const product =
await this.cacheManager.get(
 `product:${id}`
);
```

---

Lưu cache:

```ts
await this.cacheManager.set(
 `product:${id}`,
 product,
 60000
);
```

---

Xóa cache:

```ts
await this.cacheManager.del(
 `product:${id}`
);
```

---

# 6. Ví dụ CRUD có Cache

## GET product

Không cache:

```ts
async findOne(id:string){

return this.repo.findOne({
where:{id}
})

}
```

Mỗi lần:

```
API
 |
DB
```

---

Có cache:

```ts
async findOne(id:string){

const key=`product:${id}`;


const cached =
await this.cache.get(key);


if(cached){
 return cached;
}


const product =
await this.repo.findOne({
where:{id}
});


await this.cache.set(
 key,
 product,
 60000
);


return product;

}
```

Flow:

Lần đầu:

```
GET /products/1

Redis
 |
miss

Postgres
 |
save Redis
 |
response
```

Lần sau:

```
GET /products/1

Redis
 |
hit

response
```

---

# 7. Cache Invalidation (phần khó nhất)

Cache khó không phải lưu.

Khó là:

> Khi dữ liệu thay đổi thì xóa cache.

Ví dụ:

Redis:

```
product:1

{
name:"iPhone 15"
}
```

User update:

```
PATCH /products/1

name="iPhone 16"
```

Database:

```
iPhone 16
```

Nhưng Redis:

```
iPhone 15
```

=> dữ liệu sai.

---

Giải pháp:

## Delete cache khi update

```ts
async update(id,dto){


const product =
await this.repo.save(dto);


await this.cache.del(
 `product:${id}`
);


return product;

}
```

Lần sau GET:

```
Redis miss

load DB

save cache
```

---

# 8. Cache Aside Pattern

Đây là pattern phổ biến nhất.

```
Application
     |
     |
 Check Cache
     |
 +---+---+
 |       |
Hit     Miss
 |       |
Return  Query DB
          |
          |
       Update Cache
```

NestJS thường dùng pattern này.

---

# 9. Redis trong NestJS Production

Thường dùng:

```
NestJS
 |
 |
Redis
 |
 |
PostgreSQL
```

Redis dùng cho:

## 1. Cache

Ví dụ:

```
product:list:page:1
```

---

## 2. Session

Ví dụ:

```
session:user123
```

---

## 3. Refresh token

Như bạn đang làm:

```
refresh_token:userId:deviceId
```

---

## 4. Rate limit

Ví dụ:

```
login_attempt:user123

5
```

---

## 5. OTP

Ví dụ:

```
otp:register:email@gmail.com

123456

TTL 5 phút
```

---

# 10. Cache Key Design

Rất quan trọng.

Không nên:

```
user
```

Nên:

```
user:123
```

Ví dụ:

## Product

```
product:{id}
```

## Product list

```
products:category:{id}:page:{page}
```

## User profile

```
user:{userId}:profile
```

## Permission

```
user:{id}:permissions
```

---

# 11. TTL (Time To Live)

TTL là thời gian cache tồn tại.

Ví dụ:

```ts
ttl=300
```

nghĩa:

```
5 phút
```

Sau 5 phút:

```
Redis tự xoá
```

---

Chọn TTL:

|Data|TTL|
|---|---|
|Product detail|5-60 phút|
|Category|1 giờ|
|User profile|5 phút|
|Cart|không nên cache lâu|
|OTP|5 phút|
|Config|vài giờ|

---

# 12. Redis Cache vs Database

||Redis|Postgres|
|---|---|---|
|RAM|Yes|No|
|Nhanh|⭐⭐⭐⭐⭐|⭐⭐|
|Lưu lâu|No|Yes|
|Query phức tạp|No|Yes|
|Transaction|No|Yes|

Redis không thay database.

---

# 13. Cache Controller bằng Decorator

NestJS có:

```ts
@UseInterceptors(CacheInterceptor)
@Controller('products')
export class ProductController{}
```

Ví dụ:

```ts
@Get()
@CacheTTL(300)
findAll(){

return this.service.findAll();

}
```

Nest tự cache response.

Nhưng production thường ít dùng vì:

- khó invalidation
    
- khó kiểm soát key
    

Service-level cache phổ biến hơn.

---

# 14. Cache với Redis Cluster

Khi lớn:

```
        Load Balancer

             |
   +---------+---------+

 Redis 1 Redis 2 Redis 3


             |

        NestJS servers
```

Redis chia dữ liệu.

---

# 15. Cache Stampede (rất quan trọng)

Vấn đề:

Cache hết hạn:

```
product:1 expired
```

10.000 user cùng request:

```
10.000 request
       |
       v
10.000 query DB
```

DB chết.

---

Giải pháp:

## Lock

Ví dụ Redis:

```
lock:product:1
```

Request đầu:

```
SETNX lock
```

được load DB.

Request khác:

```
wait
```

---

# 16. Cache Warmup

Preload cache trước:

Ví dụ:

Server start:

```
Load top products

↓

Redis
```

User vào:

```
hit cache
```

---

# 17. Khi nào không nên cache?

Không cache:

## Payment

```
payment status
```

Vì cần realtime.

## Inventory

Ví dụ:

```
stock = 1
```

Cache sai:

```
2 người mua
```

---

## Cart

Cart thường thay đổi liên tục.

---

# 18. Trong dự án e-commerce NestJS của bạn nên cache gì?

Với stack của bạn:

```
Next.js
 |
NestJS
 |
Postgres
 |
Redis
```

Nên:

## Product detail

```
product:{id}
TTL 30 phút
```

---

## Category

```
category:list
TTL 1h
```

---

## Search suggestion

```
search:iphone
TTL 5 phút
```

---

## User permission

```
user:{id}:roles
TTL 10 phút
```

---

## Không cache:

```
Cart
Payment
Order checkout
Inventory realtime
```

---

# 19. Level Senior Backend cần hiểu Cache

Không chỉ biết:

```ts
cache.set()
cache.get()
```

Mà phải hiểu:

```
Cache Aside
Write Through
Write Behind
Cache Invalidation
TTL
Eviction
Cache Stampede
Cache Penetration
Cache Avalanche
Distributed Lock
Redis Persistence
Cluster
```

Đây là những phần thường xuất hiện khi thiết kế hệ thống lớn.