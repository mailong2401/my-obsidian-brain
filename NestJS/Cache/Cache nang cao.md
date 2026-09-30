# Tài liệu Obsidian: Redis Caching trong NestJS — Toàn tập

> **Phạm vi:** Cache strategies, vấn đề thường gặp, giải pháp nâng cao, Redis persistence, cluster
> **Liên quan:** [[Payment Module]] · [[Product Module]] · [[Cart Module]] · [[Webhook]]

---

## 📑 Mục lục

- [[#1. Tổng quan Caching Strategies]]
- [[#2. Cache Aside (Look-Aside)]]
- [[#3. Write Through]]
- [[#4. Write Behind (Write-Back)]]
- [[#5. Cache Invalidation]]
- [[#6. TTL (Time-To-Live)]]
- [[#7. Eviction Policies]]
- [[#8. Cache Stampede (Thundering Herd)]]
- [[#9. Cache Penetration]]
- [[#10. Cache Avalanche]]
- [[#11. Distributed Lock]]
- [[#12. Redis Persistence]]
- [[#13. Redis Cluster trong NestJS]]
- [[#14. Tổng kết & Checklist]]

---

## 1. Tổng quan Caching Strategies

### 1.1. So sánh 3 chiến lược chính

| Strategy | Write Path | Read Path | Consistency | Performance |
|---|---|---|---|---|
| **Cache Aside** | DB only | Cache → miss → DB → set cache | Eventual (có thể stale) | Read nhanh, write nhanh |
| **Write Through** | Cache + DB (sync) | Cache only | Strong (cache luôn fresh) | Write chậm (2 I/O) |
| **Write Behind** | Cache first → async DB | Cache only | Eventual (có thể mất data) | Write cực nhanh |

### 1.2. Khi nào dùng gì?

| Scenario | Strategy đề xuất |
|---|---|
| Read-heavy, ít write (product catalog) | **Cache Aside** |
| Write-heavy, cần consistency (user session) | **Write Through** |
| Write cực nhiều, chấp nhận eventual (analytics counters) | **Write Behind** |
| Cần survive race condition + retry | **Cache Aside + Outbox**  |

> ⚠️ **Bài học từ production:** "Write to DB, manually invalidate cache" là pattern **nguy hiểm nhất** — nếu crash giữa 2 bước → stale cache vĩnh viễn .

---

## 2. Cache Aside (Look-Aside)

### 2.1. Khái niệm

**Cache Aside** = Application code chịu trách nhiệm đọc/ghi cache thủ công.

```
Read:
  1. Check cache
  2. Hit  → return
  3. Miss → query DB → set cache → return

Write:
  1. Write DB
  2. Invalidate cache (delete, không update)
```

### 2.2. Implementation trong NestJS

```typescript
// src/infrastructure/redis/cache.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

@Injectable()
export class CacheService {
  private readonly logger = new Logger(CacheService.name);

  constructor(@InjectRedis() private readonly redis: Redis) {}

  /**
   * Cache Aside — get hoặc fallback
   */
  async getOrSet<T>(
    key: string,
    fallback: () => Promise<T>,
    ttlSeconds: number,
  ): Promise<T> {
    // 1. Try cache
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    // 2. Miss → query DB
    const fresh = await fallback();

    // 3. Set cache
    if (fresh !== null && fresh !== undefined) {
      await this.redis.setex(key, ttlSeconds, JSON.stringify(fresh));
    }

    return fresh;
  }

  /**
   * Invalidate — xóa cache sau khi write
   */
  async invalidate(...keys: string[]): Promise<void> {
    if (keys.length === 0) return;
    await this.redis.del(...keys);
    this.logger.log(`Cache invalidated: ${keys.join(', ')}`);
  }

  /**
   * Invalidate theo pattern (SCAN an toàn, không dùng KEYS)
   */
  async invalidatePattern(pattern: string): Promise<number> {
    const keys: string[] = [];
    let cursor = '0';
    do {
      const [nextCursor, batch] = await this.redis.scan(
        cursor, 'MATCH', pattern, 'COUNT', 100,
      );
      cursor = nextCursor;
      keys.push(...batch);
    } while (cursor !== '0');

    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
    return keys.length;
  }
}
```

### 2.3. Áp dụng vào ProductService

```typescript
@Injectable()
export class ProductService {
  constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
    private readonly cache: CacheService,
  ) {}

  async findOne(id: string): Promise<Product> {
    const cacheKey = `product:${id}`;

    return this.cache.getOrSet(
      cacheKey,
      async () => {
        const product = await this.productRepository.findOne({ where: { id } });
        if (!product) throw new NotFoundException(`Product ${id} not found`);
        return product;
      },
      3600, // 1 giờ
    );
  }

  async update(id: string, dto: UpdateProductDto): Promise<Product> {
    const product = await this.findOne(id);
    Object.assign(product, dto);
    const saved = await this.productRepository.save(product);

    // ⚠️ Invalidate cache SAU KHI write DB thành công
    await this.cache.invalidate(`product:${id}`, `product:slug:${saved.slug}`);

    return saved;
  }

  async remove(id: string): Promise<void> {
    const product = await this.findOne(id);
    await this.productRepository.remove(product);
    await this.cache.invalidate(`product:${id}`, `product:slug:${product.slug}`);
  }
}
```

### 2.4. ⚠️ Race Condition trong Cache Aside

```
Timeline:
T1: Request A reads DB (old value)
T2: Request B updates DB (new value)
T3: Request B invalidates cache
T4: Request A writes OLD value into cache  ← BUG!
```

**Giải pháp:**
- **Double-delete:** delete cache → write DB → delete cache (lần 2 sau delay)
- **Outbox pattern:** DB write + outbox event trong 1 transaction, consumer update cache 
- **Version/Timestamp:** cache value kèm version, chỉ update nếu version mới hơn

### 2.5. Ưu / nhược điểm

| ✅ Ưu điểm | ❌ Nhược điểm |
|---|---|
| Đơn giản, dễ implement | Có thể stale |
| Cache chỉ chứa data cần thiết | Race condition khi concurrent |
| Resilient — cache down vẫn hoạt động | Application code phức tạp |
| Phù hợp read-heavy | Read miss = 3 round trips |

---

## 3. Write Through

### 3.1. Khái niệm

**Write Through** = Ghi **đồng thời** vào cache và DB trong cùng 1 operation.

```
Write:
  1. Write cache
  2. Write DB (sync)
  3. Return success (khi cả 2 thành công)

Read:
  1. Check cache
  2. Hit → return
  3. Miss → query DB (cache miss hiếm vì luôn warm)
```

### 3.2. Implementation trong NestJS

```typescript
// src/infrastructure/redis/write-through.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';
import { DataSource } from 'typeorm';

@Injectable()
export class WriteThroughService {
  private readonly logger = new Logger(WriteThroughService.name);

  constructor(
    @InjectRedis() private readonly redis: Redis,
    private readonly dataSource: DataSource,
  ) {}

  /**
   * Write Through — cache + DB đồng thời
   * ⚠️ Nếu 1 trong 2 fail → phải rollback cái còn lại
   */
  async writeThrough<T>(
    key: string,
    entity: any,
    ttlSeconds: number,
    repository: any,
  ): Promise<T> {
    const queryRunner = this.dataSource.createQueryRunner();
    await queryRunner.connect();
    await queryRunner.startTransaction();

    try {
      // 1. Write DB trong transaction
      const saved = await queryRunner.manager.save(repository, entity);

      // 2. Write cache (chưa commit DB)
      await this.redis.setex(key, ttlSeconds, JSON.stringify(saved));

      // 3. Commit DB
      await queryRunner.commitTransaction();

      this.logger.log(`Write-through OK: ${key}`);
      return saved;
    } catch (error) {
      // Rollback DB
      await queryRunner.rollbackTransaction();

      // Rollback cache (best-effort — có thể đã set)
      await this.redis.del(key).catch(() => {});

      this.logger.error(`Write-through FAILED: ${key}`, error);
      throw error;
    } finally {
      await queryRunner.release();
    }
  }
}
```

### 3.3. ⚠️ Vấn đề với Write Through

**"Looks safest. Is the most dangerous at scale."** 

| Vấn đề | Hệ quả |
|---|---|
| 2 synchronous I/O mỗi write | Write API chậm |
| Redis slow → write API slow | Latency tăng |
| Redis down → write blocked | Redis thành **hard dependency** |
| Cache có thể chứa data không bao giờ đọc | Wasted memory |

### 3.4. Khi nào dùng?

- Session store (luôn cần fresh)
- Config data ít thay đổi nhưng đọc nhiều
- Khi consistency **quan trọng hơn** performance

---

## 4. Write Behind (Write-Back)

### 4.1. Khái niệm

**Write Behind** = Ghi vào cache trước, **async** flush xuống DB sau.

```
Write:
  1. Write cache (return ngay)
  2. Background worker flush DB sau (batch)

Read:
  1. Check cache
  2. Hit → return (luôn có vì vừa write)
  3. Miss → query DB
```

### 4.2. Implementation với BullMQ

```typescript
// src/infrastructure/queue/write-behind.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import { InjectQueue } from '@nestjs/bullmq';
import { Queue } from 'bullmq';
import Redis from 'ioredis';

@Injectable()
export class WriteBehindService {
  private readonly logger = new Logger(WriteBehindService.name);

  constructor(
    @InjectRedis() private readonly redis: Redis,
    @InjectQueue('db-flush') private readonly flushQueue: Queue,
  ) {}

  /**
   * Write Behind — cache first, async DB
   * ⚠️ CHỈ dùng cho data chấp nhận mất mát (analytics, view count)
   */
  async writeBehind<T>(
    key: string,
    data: T,
    ttlSeconds: number,
    dbOperation: { type: string; payload: any },
  ): Promise<void> {
    // 1. Write cache ngay
    await this.redis.setex(key, ttlSeconds, JSON.stringify(data));

    // 2. Enqueue DB flush
    await this.flushQueue.add(
      'flush',
      {
        key,
        operation: dbOperation,
        timestamp: Date.now(),
      },
      {
        attempts: 5,
        backoff: { type: 'exponential', delay: 1000 },
        removeOnComplete: 100,
        removeOnFail: 500,
      },
    );

    this.logger.log(`Write-behind queued: ${key}`);
  }
}

// src/infrastructure/queue/db-flush.processor.ts
import { Processor, WorkerHost } from '@nestjs/bullmq';
import { Job } from 'bullmq';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';

@Processor('db-flush')
export class DbFlushProcessor extends WorkerHost {
  private readonly logger = new Logger(DbFlushProcessor.name);

  constructor(
    @InjectRepository(ProductView)
    private readonly viewRepo: Repository<ProductView>,
    @InjectRedis() private readonly redis: Redis,
  ) {
    super();
  }

  async process(job: Job): Promise<any> {
    const { key, operation } = job.data;

    this.logger.log(`Flushing ${key} to DB (attempt ${job.attemptsMade + 1})`);

    switch (operation.type) {
      case 'increment_view':
        await this.viewRepo.increment(
          { productId: operation.payload.productId },
          'viewCount',
          operation.payload.amount,
        );
        break;
      case 'update_counter':
        // ...
        break;
    }

    // Xóa flag "dirty" trong Redis
    await this.redis.del(`dirty:${key}`);

    return { success: true };
  }
}
```

### 4.3. Aggregation — gộp nhiều writes

```typescript
// Thay vì flush từng cái, gộp lại theo batch
@Cron('*/30 * * * * *')  // Mỗi 30s
async flushAggregatedCounters(): Promise<void> {
  const dirtyKeys = await this.redis.scanKeys('dirty:counter:*');

  for (const key of dirtyKeys) {
    const value = await this.redis.get(key);
    if (!value) continue;

    // Parse + update DB
    const { entityId, counter, delta } = JSON.parse(value);
    await this.counterRepo.increment({ id: entityId }, counter, delta);

    // Clear
    await this.redis.del(key);
  }
}
```

### 4.4. ⚠️ Rủi ro Write Behind

| Rủi ro | Giải pháp |
|---|---|
| Worker crash trước khi flush | BullMQ retry + persistent queue |
| Redis down → mất data | Redis AOF persistence (xem §12) |
| DB down → queue backlog | Monitoring + alert |
| Data ordering | Single worker hoặc key-based partitioning |

> ⚠️ **KHÔNG BAO GIỜ dùng Write Behind cho source of truth** (money, order status). Chỉ dùng cho analytics, counters, view count .

### 4.5. Real-world: View counter 100M views

```typescript
// Thay vì UPDATE products SET views = views + 1 mỗi request
// → Dùng Redis INCR + flush định kỳ

async recordView(productId: string): Promise<void> {
  const key = `views:${productId}`;

  // Atomic increment trong Redis (O(1))
  const count = await this.redis.incr(key);

  // Set TTL nếu là lần đầu
  if (count === 1) {
    await this.redis.expire(key, 3600);
  }

  // Mark dirty để cronjob flush
  await this.redis.sadd('dirty:views', productId);
}

@Cron('*/60 * * * * *')  // Mỗi phút
async flushViews(): Promise<void> {
  const productIds = await this.redis.smembers('dirty:views');

  for (const productId of productIds) {
    const key = `views:${productId}`;
    const count = await this.redis.getdel(key);  // Atomic get + delete

    if (count) {
      await this.productRepo.increment({ id: productId }, 'viewCount', Number(count));
    }
  }

  await this.redis.del('dirty:views');
}
```

**Kết quả:** 1000 writes/giây → 1 DB write/phút .

---

## 5. Cache Invalidation

### 5.1. "There are only two hard things in Computer Science"

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

### 5.2. Các chiến lược invalidation

| Strategy | Cách làm | Khi nào dùng |
|---|---|---|
| **TTL-based** | Đợi hết hạn | Chấp nhận stale ngắn |
| **Explicit delete** | Xóa key khi write | Cần fresh ngay |
| **Pattern delete** | Xóa theo prefix | Invalidate nhóm |
| **Version-based** | Version trong key | Tránh race condition |
| **Tag-based** | Gán tag cho key | Invalidate theo entity |

### 5.3. Pattern-based invalidation (SCAN an toàn)

```typescript
/**
 * ❌ KHÔNG DÙNG KEYS trong production — block Redis
 */
async badInvalidate(pattern: string): Promise<void> {
  const keys = await this.redis.keys(pattern);  // ⚠️ O(N), block
  await this.redis.del(...keys);
}

/**
 * ✅ DÙNG SCAN — non-blocking, cursor-based
 */
async invalidatePattern(pattern: string): Promise<number> {
  let cursor = '0';
  let deleted = 0;

  do {
    const [nextCursor, keys] = await this.redis.scan(
      cursor,
      'MATCH', pattern,
      'COUNT', 100,
    );
    cursor = nextCursor;

    if (keys.length > 0) {
      await this.redis.del(...keys);
      deleted += keys.length;
    }
  } while (cursor !== '0');

  return deleted;
}

// Usage
await this.invalidatePattern('product:*');           // Tất cả products
await this.invalidatePattern('cart:summary:*');      // Tất cả cart summaries
```

### 5.4. Tag-based invalidation

```typescript
/**
 * Gán tag cho key → dễ invalidate theo nhóm
 * Ví dụ: product:123 có tags ["products", "category:electronics"]
 */
async setWithTags(
  key: string,
  value: any,
  ttl: number,
  tags: string[],
): Promise<void> {
  const pipeline = this.redis.pipeline();

  // Set value
  pipeline.setex(key, ttl, JSON.stringify(value));

  // Add key vào từng tag set
  for (const tag of tags) {
    pipeline.sadd(`tag:${tag}`, key);
    pipeline.expire(`tag:${tag}`, ttl);  // Tag set cũng có TTL
  }

  await pipeline.exec();
}

async invalidateByTag(tag: string): Promise<number> {
  const tagKey = `tag:${tag}`;
  const keys = await this.redis.smembers(tagKey);

  if (keys.length === 0) return 0;

  const pipeline = this.redis.pipeline();
  pipeline.del(...keys);
  pipeline.del(tagKey);
  await pipeline.exec();

  return keys.length;
}

// Usage
await this.setWithTags(
  'product:123',
  product,
  3600,
  ['products', 'category:electronics', 'brand:samsung'],
);

// Khi update category
await this.invalidateByTag('category:electronics');
// → Xóa tất cả products thuộc category này
```

### 5.5. Invalidation trong hệ thống E-Commerce

```typescript
// Khi update Product
async updateProduct(id: string, dto: UpdateProductDto): Promise<Product> {
  const product = await this.findOne(id);
  Object.assign(product, dto);
  const saved = await this.productRepository.save(product);

  // Invalidate TẤT CẢ cache liên quan
  await Promise.all([
    this.cache.invalidate(`product:${id}`),
    this.cache.invalidate(`product:slug:${saved.slug}`),
    this.cache.invalidatePattern(`product:list:*`),  // List bị ảnh hưởng
    this.cache.invalidateByTag(`category:${saved.category}`),
  ]);

  return saved;
}
```

---

## 6. TTL (Time-To-Live)

### 6.1. Khái niệm

**TTL** = Thời gian sống của cache entry. Hết hạn → Redis tự động xóa.

### 6.2. Set TTL trong NestJS

```typescript
// Với ioredis
await redis.setex('key', 3600, 'value');           // TTL = 3600s
await redis.set('key', 'value', 'EX', 3600);        // Tương đương
await redis.expire('key', 3600);                    // Set TTL cho key có sẵn

// Với @nestjs/cache-manager
await cacheManager.set('key', 'value', 1000);       // TTL = 1000ms
await cacheManager.set('key', 'value', { ttl: 3600 } as any);  // NestJS 11

// Disable TTL (không bao giờ hết hạn)
await cacheManager.set('key', 'value', 0);
```

### 6.3. Chọn TTL phù hợp

| Data type | TTL đề xuất | Lý do |
|---|---|---|
| Product detail | 1-24h | Ít thay đổi |
| Product list | 5-15 phút | Thay đổi khi có product mới |
| User profile | 15-60 phút | Thay đổi khi user update |
| Cart summary | 5 phút | Thay đổi thường xuyên |
| Session | 7 ngày | Theo refresh token |
| OTP | 5 phút | Security |
| Config | 1h - 24h | Ít thay đổi |

### 6.4. ⚠️ TTL Jitter — chống Cache Avalanche

```typescript
/**
 * ❌ SAI — tất cả key cùng hết hạn 1 lúc
 */
await redis.setex(`product:${id}`, 3600, data);  // Tất cả TTL = 3600

/**
 * ✅ ĐÚNG — thêm jitter (random offset)
 */
const baseTtl = 3600;
const jitter = Math.floor(Math.random() * 600);  // 0-10 phút
const ttl = baseTtl + jitter;  // 3600-4200

await redis.setex(`product:${id}`, ttl, data);
```

**Tại sao?** Nếu 10,000 keys được set cùng lúc với TTL = 3600 → tất cả hết hạn cùng lúc → DB bị flood .

### 6.5. TTL cho từng loại key

```typescript
export const CacheTTL = {
  PRODUCT_DETAIL: 3600,        // 1h
  PRODUCT_LIST: 300,           // 5 phút
  USER_PROFILE: 1800,          // 30 phút
  CART_SUMMARY: 300,           // 5 phút
  SESSION: 7 * 24 * 3600,      // 7 ngày
  OTP: 300,                    // 5 phút
  RATE_LIMIT: 60,              // 1 phút
  WEBHOOK_IDEMPOTENCY: 7 * 24 * 3600,  // 7 ngày
} as const;

// Usage với jitter
function withJitter(baseTtl: number, jitterPercent = 0.1): number {
  const jitter = Math.floor(baseTtl * jitterPercent * Math.random());
  return baseTtl + jitter;
}
```

---

## 7. Eviction Policies

### 7.1. Khi nào eviction xảy ra?

Khi Redis đạt `maxmemory` limit, nó phải **evict** (xóa) keys để có chỗ cho data mới .

### 7.2. Các policies

| Policy | Mô tả | Khi nào dùng |
|---|---|---|
| `noeviction` | Không xóa, báo lỗi khi đầy | Data không được mất |
| `allkeys-lru` | Xóa ít dùng gần đây nhất | **Cache thuần** (recommended) |
| `allkeys-lfu` | Xóa ít dùng nhất | Cache có hot/cold pattern |
| `allkeys-random` | Xóa random | Ít dùng |
| `volatile-lru` | LRU trên keys có TTL | Mix cache + persistent |
| `volatile-lfu` | LFU trên keys có TTL | Mix + frequency |
| `volatile-ttl` | Xóa key sắp hết hạn | TTL-based priority |

### 7.3. Cấu hình trong Docker

```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7-alpine
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
```

### 7.4. Cấu hình trong NestJS (ioredis)

```typescript
// redis.module.ts
import { Module, Global } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { RedisModule as IORedisModule } from '@nestjs-modules/ioredis';

@Global()
@Module({
  imports: [
    IORedisModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'single',
        options: {
          host: config.get('REDIS_HOST', 'localhost'),
          port: config.get('REDIS_PORT', 6379),
          password: config.get('REDIS_PASSWORD'),
          db: config.get('REDIS_DB', 0),
          // ⚠️ maxmemory-policy là server-side, không phải client-side
          // Phải config trong redis.conf hoặc qua command line
        },
      }),
    }),
  ],
})
export class RedisModule {}
```

### 7.5. LRU vs LFU — chọn cái nào?

| Tiêu chí | LRU | LFU |
|---|---|---|
| **Pattern** | Recency (mới dùng) | Frequency (dùng nhiều) |
| **Hot data** | Keys vừa access | Keys access thường xuyên |
| **Cold data bị xóa** | Keys lâu không dùng | Keys ít dùng |
| **Vấn đề** | Scan 1 lần → cache pollution | Frequency counter decay |
| **Dùng khi** | Web cache thông thường | Cache có hot subset rõ ràng |

### 7.6. Monitoring eviction

```bash
# Xem số keys bị evict
redis-cli INFO stats | grep evicted_keys

# Xem memory usage
redis-cli INFO memory | grep used_memory_human

# Xem policy hiện tại
redis-cli CONFIG GET maxmemory-policy
```

---

## 8. Cache Stampede (Thundering Herd)

### 8.1. Vấn đề

Khi một key hot hết hạn, **tất cả requests** cùng lúc miss cache → cùng query DB → DB quá tải .

```
TTL expires at T=0

T=0.001: Request 1 → cache miss → DB query
T=0.002: Request 2 → cache miss → DB query
T=0.003: Request 3 → cache miss → DB query
...
T=0.100: Request 1000 → cache miss → DB query

→ 1000 DB queries cho cùng 1 key!
```

### 8.2. Giải pháp 1: Distributed Lock (Single-flight)

```typescript
// src/infrastructure/redis/stampede.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';
import { randomUUID } from 'crypto';

@Injectable()
export class StampedeProtectedCache {
  private readonly logger = new Logger(StampedeProtectedCache.name);

  constructor(@InjectRedis() private readonly redis: Redis) {}

  /**
   * Single-flight cache
   * Chỉ 1 request query DB, còn lại đợi và đọc từ cache
   */
  async getOrSet<T>(
    key: string,
    fallback: () => Promise<T>,
    ttlSeconds: number,
    lockTtl = 10,  // Lock TTL ngắn hơn data TTL
  ): Promise<T> {
    // 1. Try cache
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached) as T;
    }

    // 2. Cache miss → acquire lock
    const lockKey = `lock:${key}`;
    const lockValue = randomUUID();

    const acquired = await this.redis.set(
      lockKey,
      lockValue,
      'EX', lockTtl,
      'NX',
    );

    if (acquired === 'OK') {
      // 3. Winner: query DB
      try {
        const fresh = await fallback();
        await this.redis.setex(key, ttlSeconds, JSON.stringify(fresh));
        return fresh;
      } finally {
        // Release lock an toàn (chỉ xóa nếu value khớp)
        const script = `
          if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
          else
            return 0
          end
        `;
        await this.redis.eval(script, 1, lockKey, lockValue);
      }
    }

    // 4. Loser: đợi và retry
    for (let i = 0; i < 10; i++) {
      await new Promise((r) => setTimeout(r, 50));
      const retry = await this.redis.get(key);
      if (retry) {
        return JSON.parse(retry) as T;
      }
    }

    // 5. Timeout → fallback trực tiếp (tránh deadlock)
    this.logger.warn(`Stampede lock timeout for ${key}, falling back`);
    return fallback();
  }
}
```

### 8.3. Giải pháp 2: Stale-While-Revalidate (SWR)

```typescript
/**
 * SWR — Trả stale data ngay, refresh background
 * Phù hợp data chấp nhận stale ngắn
 */
async getOrSetSWR<T>(
  key: string,
  fallback: () => Promise<T>,
  ttlSeconds: number,
  staleTtl: number,  // Thời gian cho phép serve stale
): Promise<T> {
  const cached = await this.redis.get(key);

  if (cached) {
    const { data, createdAt } = JSON.parse(cached);
    const age = (Date.now() - createdAt) / 1000;

    if (age < ttlSeconds) {
      // Fresh → return
      return data;
    }

    if (age < ttlSeconds + staleTtl) {
      // Stale nhưng còn trong grace period → return + refresh background
      this.refreshInBackground(key, fallback, ttlSeconds);
      return data;
    }
  }

  // Cache miss hoặc quá stale → fetch
  const fresh = await fallback();
  await this.setWithTimestamp(key, fresh, ttlSeconds + staleTtl);
  return fresh;
}

private async refreshInBackground<T>(
  key: string,
  fallback: () => Promise<T>,
  ttlSeconds: number,
): Promise<void> {
  // Fire-and-forget
  fallback()
    .then((fresh) => this.setWithTimestamp(key, fresh, ttlSeconds))
    .catch((err) => this.logger.error(`Background refresh failed for ${key}`, err));
}

private async setWithTimestamp<T>(
  key: string,
  data: T,
  ttlSeconds: number,
): Promise<void> {
  await this.redis.setex(
    key,
    ttlSeconds,
    JSON.stringify({ data, createdAt: Date.now() }),
  );
}
```

### 8.4. Giải pháp 3: Jitter TTL

```typescript
// Thêm random offset để tránh synchronized expiry
function withJitter(baseTtl: number, jitterPercent = 0.2): number {
  const jitter = Math.floor(baseTtl * jitterPercent * Math.random());
  return baseTtl + jitter;
}

await this.redis.setex(key, withJitter(3600), data);
// TTL: 3600 - 4320 seconds
```

### 8.5. So sánh các giải pháp

| Giải pháp | Pros | Cons |
|---|---|---|
| **Distributed Lock** | Chỉ 1 DB query | Phức tạp, có thể deadlock |
| **SWR** | Response nhanh, đơn giản | Serve stale data |
| **Jitter TTL** | Đơn giản, hiệu quả | Chỉ giảm, không loại bỏ |
| **Background refresh** | DB không bị block | Cần worker |

---

## 9. Cache Penetration

### 9.1. Vấn đề

Query **data không tồn tại** → cache miss → DB query → DB cũng không có → **không cache gì cả** → request sau lại miss → DB lại query .

```
Request: GET /products/999999  (không tồn tại)

1. Cache miss
2. DB query → không có
3. Không cache (vì data null)
4. Return 404

Request tiếp theo: 999999 → lặp lại bước 1-4
→ Attacker có thể spam 999999 → DB quá tải
```

### 9.2. Giải pháp 1: Cache null values

```typescript
async findOne(id: string): Promise<Product> {
  const cacheKey = `product:${id}`;

  // Try cache
  const cached = await this.redis.get(cacheKey);

  if (cached === '__NULL__') {
    // Đã biết là không tồn tại → throw luôn
    throw new NotFoundException(`Product ${id} not found`);
  }

  if (cached) {
    return JSON.parse(cached) as Product;
  }

  // Query DB
  const product = await this.productRepository.findOne({ where: { id } });

  if (!product) {
    // Cache null với TTL ngắn hơn (5 phút)
    await this.redis.setex(cacheKey, 300, '__NULL__');
    throw new NotFoundException(`Product ${id} not found`);
  }

  await this.redis.setex(cacheKey, 3600, JSON.stringify(product));
  return product;
}
```

### 9.3. Giải pháp 2: Bloom Filter

```typescript
import { BloomFilter } from 'bloom-filters';

@Injectable()
export class ProductService implements OnModuleInit {
  private bloomFilter: BloomFilter;

  async onModuleInit() {
    // Load tất cả product IDs vào bloom filter
    const products = await this.productRepository.find({ select: ['id'] });

    this.bloomFilter = BloomFilter.create(
      products.length * 2,  // Capacity
      0.01,                 // False positive rate 1%
    );

    for (const p of products) {
      this.bloomFilter.add(p.id);
    }

    this.logger.log(`Bloom filter loaded with ${products.length} products`);
  }

  async findOne(id: string): Promise<Product> {
    // 1. Check bloom filter
    if (!this.bloomFilter.has(id)) {
      // Chắc chắn không tồn tại → reject ngay, không query DB
      throw new NotFoundException(`Product ${id} not found`);
    }

    // 2. Có thể tồn tại (có false positive) → query bình thường
    const product = await this.productRepository.findOne({ where: { id } });
    if (!product) {
      throw new NotFoundException(`Product ${id} not found`);
    }
    return product;
  }

  // Rebuild khi có product mới
  async rebuildBloomFilter(): Promise<void> {
    await this.onModuleInit();
  }
}
```

### 9.4. Giải pháp 3: Rate limiting theo IP/User

```typescript
// Đã có trong hệ thống — UserThrottlerGuard
@Throttle({ default: { ttl: 60_000, limit: 100 } })
async findOne(@Param('id') id: string) {
  // ...
}
```

### 9.5. So sánh

| Giải pháp | Pros | Cons |
|---|---|---|
| **Cache null** | Đơn giản, hiệu quả | Tốn memory cho null |
| **Bloom Filter** | Không tốn memory cho null | False positive, cần rebuild |
| **Rate limit** | Chặn attacker | Không chặn được distributed attack |

---

## 10. Cache Avalanche

### 10.1. Vấn đề

**Nhiều keys hết hạn cùng lúc** (hoặc Redis down) → tất cả requests đổ về DB cùng lúc .

```
Scenario 1: Mass expiration
- Deploy lúc 10:00 → warm 10,000 keys với TTL = 3600
- 11:00 → 10,000 keys hết hạn cùng lúc → DB flood

Scenario 2: Redis down
- Redis crash → tất cả cache miss → DB flood
```

### 10.2. Giải pháp 1: TTL Jitter (đã nói ở §6.4)

```typescript
await this.redis.setex(key, withJitter(3600, 0.2), data);
```

### 10.3. Giải pháp 2: Multi-level Cache

```typescript
/**
 * L1 (in-memory) + L2 (Redis)
 * L1 hit → không cần Redis
 * L2 hit → copy lên L1
 */
@Injectable()
export class MultiLevelCache {
  private l1 = new Map<string, { data: any; expiresAt: number }>();

  constructor(@InjectRedis() private readonly redis: Redis) {}

  async get<T>(key: string): Promise<T | null> {
    // 1. L1 check
    const l1Entry = this.l1.get(key);
    if (l1Entry && l1Entry.expiresAt > Date.now()) {
      return l1Entry.data as T;
    }

    // 2. L2 check
    const l2Value = await this.redis.get(key);
    if (l2Value) {
      const parsed = JSON.parse(l2Value) as T;

      // Promote lên L1 với TTL ngắn hơn
      this.l1.set(key, {
        data: parsed,
        expiresAt: Date.now() + 60_000,  // 1 phút
      });

      return parsed;
    }

    return null;
  }

  async set<T>(key: string, value: T, ttlSeconds: number): Promise<void> {
    // Set L2
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value));

    // Set L1 với TTL ngắn hơn (tránh stale lâu)
    this.l1.set(key, {
      data: value,
      expiresAt: Date.now() + Math.min(ttlSeconds * 1000, 60_000),
    });
  }

  async invalidate(key: string): Promise<void> {
    this.l1.delete(key);
    await this.redis.del(key);
  }
}
```

### 10.4. Giải pháp 3: Circuit Breaker cho Redis

```typescript
import { CircuitBreaker } from 'opossum';

@Injectable()
export class ResilientCache {
  private breaker: CircuitBreaker;

  constructor(@InjectRedis() private readonly redis: Redis) {
    this.breaker = new CircuitBreaker(
      (fn: () => Promise<any>) => fn(),
      {
        timeout: 1000,          // 1s timeout
        errorThresholdPercentage: 50,
        resetTimeout: 30_000,   // 30s để thử lại
      },
    );

    // Fallback khi Redis down
    this.breaker.fallback(() => null);
  }

  async get<T>(key: string): Promise<T | null> {
    const result = await this.breaker.fire(() => this.redis.get(key));
    return result ? JSON.parse(result) : null;
  }
}
```

### 10.5. Giải pháp 4: Cache Warming

```typescript
@Injectable()
export class CacheWarmupService implements OnModuleInit {
  async onModuleInit() {
    // Warm cache khi app start
    await this.warmHotProducts();
  }

  private async warmHotProducts(): Promise<void> {
    // Load top 100 sản phẩm bán chạy
    const hotProducts = await this.productRepository.find({
      order: { viewCount: 'DESC' },
      take: 100,
    });

    for (const product of hotProducts) {
      await this.redis.setex(
        `product:${product.id}`,
        withJitter(3600),
        JSON.stringify(product),
      );
    }

    this.logger.log(`Warmed ${hotProducts.length} hot products`);
  }

  // Cronjob warm lại định kỳ
  @Cron('0 */30 * * * *')  // Mỗi 30 phút
  async periodicWarmup(): Promise<void> {
    await this.warmHotProducts();
  }
}
```

---

## 11. Distributed Lock

### 11.1. Tại sao cần?

Trong hệ thống **multi-instance** (nhiều pod/container), cần **mutual exclusion** để tránh race condition .

### 11.2. Redis SET NX EX — Atomic lock

```typescript
// src/infrastructure/redis/lock.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { InjectRedis } from '@nestjs-modules/ioredis';
import Redis from 'ioredis';
import { randomUUID } from 'crypto';

@Injectable()
export class DistributedLockService {
  private readonly logger = new Logger(DistributedLockService.name);

  constructor(@InjectRedis() private readonly redis: Redis) {}

  /**
   * Acquire lock — atomic SET NX EX
   * Trả về lock value (UUID) hoặc null
   */
  async acquire(
    key: string,
    ttlSeconds: number = 10,
  ): Promise<string | null> {
    const lockValue = randomUUID();

    const result = await this.redis.set(
      `lock:${key}`,
      lockValue,
      'EX', ttlSeconds,
      'NX',
    );

    if (result === 'OK') {
      this.logger.debug(`Lock acquired: ${key} (owner=${lockValue})`);
      return lockValue;
    }

    return null;
  }

  /**
   * Release lock — chỉ xóa nếu value khớp (Lua script atomic)
   */
  async release(key: string, lockValue: string): Promise<boolean> {
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
      else
        return 0
      end
    `;

    const result = await this.redis.eval(script, 1, `lock:${key}`, lockValue);

    if (result === 1) {
      this.logger.debug(`Lock released: ${key}`);
      return true;
    }

    return false;
  }

  /**
   * Extend lock TTL — heartbeat
   */
  async extend(key: string, lockValue: string, ttlSeconds: number): Promise<boolean> {
    const script = `
      if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("expire", KEYS[1], ARGV[2])
      else
        return 0
      end
    `;

    const result = await this.redis.eval(
      script, 1, `lock:${key}`, lockValue, ttlSeconds,
    );

    return result === 1;
  }

  /**
   * Execute với lock — auto release
   */
  async withLock<T>(
    key: string,
    ttlSeconds: number,
    fn: () => Promise<T>,
  ): Promise<T> {
    const lockValue = await this.acquire(key, ttlSeconds);

    if (!lockValue) {
      throw new Error(`Could not acquire lock: ${key}`);
    }

    try {
      return await fn();
    } finally {
      await this.release(key, lockValue);
    }
  }
}
```

### 11.3. Áp dụng: Ngăn duplicate payment

```typescript
// payment.service.ts
async createPayment(userId: string, dto: CreatePaymentDto, ip: string) {
  const lockKey = `payment:create:${dto.cartId}`;

  return this.lockService.withLock(lockKey, 30, async () => {
    // Re-check trong lock (double-check)
    const existing = await this.paymentRepo.findOne({
      where: { cartId: dto.cartId, status: PaymentStatus.PENDING },
    });

    if (existing?.paymentUrl) {
      return {
        paymentUrl: existing.paymentUrl,
        transactionId: existing.id,
      };
    }

    // Tạo mới...
    return this.doCreatePayment(userId, dto, ip);
  });
}
```

### 11.4. ⚠️ Pitfalls của Distributed Lock

| Pitfall | Giải pháp |
|---|---|
| Lock hết hạn trước khi xong | Heartbeat extend |
| Client crash → lock leak | TTL tự động giải phóng |
| Redis master fail → mất lock | Redlock algorithm (multi-master) |
| Clock skew | Dùng Redis time, không dùng local time |
| Lock không fair | FIFO queue với sorted set |

### 11.5. Redlock (multi-master) — khi nào cần?

```typescript
// Chỉ dùng khi Redis single-point-of-failure không chấp nhận được
// Overkill cho hầu hết use cases
const redlock = new Redlock(
  [redis1, redis2, redis3],
  {
    driftFactor: 0.01,
    retryCount: 10,
    retryDelay: 200,
    retryJitter: 200,
  },
);
```

> ⚠️ **Martin Kleppmann's critique:** Redlock không đảm bảo safety nếu có clock skew hoặc GC pause. Dùng **fencing token** (monotonic counter) thay vì chỉ lock .

---

## 12. Redis Persistence

### 12.1. Tại sao cần persistence?

Redis là **in-memory** — restart → mất hết data. Persistence đảm bảo data survive restart .

### 12.2. RDB (Redis Database) — Snapshot

```bash
# redis.conf
save 900 1      # Save nếu 1 key thay đổi trong 900s
save 300 10     # Save nếu 10 keys thay đổi trong 300s
save 60 10000   # Save nếu 10000 keys thay đổi trong 60s

dbfilename dump.rdb
dir /data
```

**Ưu:** File nhỏ, restore nhanh
**Nhược:** Có thể mất data giữa các snapshot

### 12.3. AOF (Append-Only File) — Log

```bash
# redis.conf
appendonly yes
appendfilename "appendonly.aof"

# fsync policy
appendfsync everysec   # (recommended) — mất tối đa 1s data
# appendfsync always   # Chậm nhất, an toàn nhất
# appendfsync no       # Nhanh nhất, không đảm bảo
```

**Ưu:** Durable hơn, có thể replay
**Nhược:** File lớn hơn, restore chậm hơn

### 12.4. Hybrid (RDB + AOF) — recommended

```bash
# redis.conf
appendonly yes
aof-use-rdb-preamble yes   # AOF file bắt đầu bằng RDB snapshot

save 900 1
save 300 10
save 60 10000
```

### 12.5. Docker Compose config

```yaml
services:
  redis:
    image: redis:7-alpine
    command: >
      redis-server
      --appendonly yes
      --appendfsync everysec
      --aof-use-rdb-preamble yes
      --save 900 1
      --save 300 10
      --save 60 10000
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s

volumes:
  redis_data:
```

### 12.6. Trade-offs

| Config | Durability | Performance | Disk |
|---|---|---|---|
| RDB only | Thấp (mất giữa snapshots) | Cao | Ít |
| AOF `everysec` | Trung bình (mất ≤1s) | Trung bình | Nhiều |
| AOF `always` | Cao | Thấp | Nhiều |
| Hybrid | Cao | Trung bình | Trung bình |

### 12.7. Khi nào KHÔNG cần persistence?

- Cache thuần túy (có thể rebuild từ DB)
- Dùng Redis chỉ cho rate limiting
- Session có thể mất (user login lại)

> 💡 **Nguyên tắc:** Nếu Redis data có thể rebuild từ DB → không cần persistence. Nếu là source of truth → cần AOF.

---

## 13. Redis Cluster trong NestJS

### 13.1. Khi nào cần Cluster?

| Scenario | Giải pháp |
|---|---|
| Single Redis đủ | **Standalone** |
| Cần HA, tự động failover | **Sentinel** |
| Data > RAM 1 máy | **Cluster** |
| Cần sharding | **Cluster** |

### 13.2. Cấu hình Sentinel (HA)

```typescript
// src/infrastructure/redis/redis-sentinel.module.ts
import { Module, Global } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { RedisModule as IORedisModule } from '@nestjs-modules/ioredis';

@Global()
@Module({
  imports: [
    IORedisModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'sentinel',
        options: {
          sentinels: [
            { host: 'sentinel-1', port: 26379 },
            { host: 'sentinel-2', port: 26379 },
            { host: 'sentinel-3', port: 26379 },
          ],
          name: 'mymaster',  // Master name
          password: config.get('REDIS_PASSWORD'),
          db: 0,
          sentinelPassword: config.get('REDIS_SENTINEL_PASSWORD'),
          enableTLSForSentinelMode: false,
        },
      }),
    }),
  ],
})
export class RedisSentinelModule {}
```

### 13.3. Cấu hình Cluster

```typescript
// src/infrastructure/redis/redis-cluster.module.ts
import { Module, Global } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { RedisModule as IORedisModule } from '@nestjs-modules/ioredis';

@Global()
@Module({
  imports: [
    IORedisModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'cluster',
        options: {
          nodes: [
            { host: 'redis-node-1', port: 6379 },
            { host: 'redis-node-2', port: 6379 },
            { host: 'redis-node-3', port: 6379 },
            { host: 'redis-node-4', port: 6379 },
            { host: 'redis-node-5', port: 6379 },
            { host: 'redis-node-6', port: 6379 },
          ],
          redisOptions: {
            password: config.get('REDIS_PASSWORD'),
            tls: config.get('REDIS_TLS') === 'true' ? {} : undefined,
          },
          clusterRetryStrategy: (times) => {
            return Math.min(times * 100, 3000);
          },
          // Cluster-specific
          enableOfflineQueue: true,
          enableReadyCheck: true,
          scaleReads: 'slave',  // Đọc từ slave (giảm tải master)
          maxRedirections: 16,   // MOVED/ASK redirects
        },
      }),
    }),
  ],
})
export class RedisClusterModule {}
```

### 13.4. Docker Compose cho Cluster

```yaml
# docker-compose.cluster.yml
services:
  redis-node-1:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --requirepass redis123
    ports: ["7001:6379"]
    volumes: ["redis-node-1-data:/data"]

  redis-node-2:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --requirepass redis123
    ports: ["7002:6379"]
    volumes: ["redis-node-2-data:/data"]

  redis-node-3:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf --cluster-node-timeout 5000 --appendonly yes --requirepass redis123
    ports: ["7003:6379"]
    volumes: ["redis-node-3-data:/data"]

  # Thêm 3 nodes nữa cho replica...

  redis-cluster-init:
    image: redis:7-alpine
    depends_on:
      - redis-node-1
      - redis-node-2
      - redis-node-3
    command: >
      sh -c "
      sleep 5 &&
      redis-cli -a redis123 --cluster create
      redis-node-1:6379 redis-node-2:6379 redis-node-3:6379
      --cluster-replicas 0 --cluster-yes
      "
```

### 13.5. ⚠️ Cluster limitations

| Vấn đề | Giải thích |
|---|---|
| **Multi-key operations** | Không hoạt động cross-slot (MGET, MSET, pipeline) |
| **Hash tags** | Dùng `{user:123}:cart` để cùng slot |
| **Transactions** | MULTI/EXEC chỉ trong cùng slot |
| **Lua scripts** | Keys phải cùng slot |
| **Pub/Sub** | Broadcast toàn cluster (không sharded) |

**Hash tag trick:**
```typescript
// ❌ Khác slot → lỗi CROSSSLOT
await redis.mget('user:123:name', 'user:123:email');

// ✅ Cùng slot (nhờ hash tag)
await redis.mget('{user:123}:name', '{user:123}:email');
// → Cả 2 hash về cùng slot vì hash tag = "user:123"
```

### 13.6. Client-side sharding (thay thế)

Nếu không cần full cluster, có thể dùng client-side sharding:

```typescript
import { ShardingManager } from 'ioredis';

const shards = [
  new Redis({ host: 'redis-1', port: 6379 }),
  new Redis({ host: 'redis-2', port: 6379 }),
  new Redis({ host: 'redis-3', port: 6379 }),
];

const shardManager = new ShardingManager(shards, {
  keyPrefix: 'app:',
});

// Tự động route key đến shard
await shardManager.set('user:123', data);
```

---

## 14. Tổng kết & Checklist

### 14.1. Decision Tree — Chọn Strategy

```
Cần cache?
├─ NO → Không làm gì
└─ YES
   ├─ Read-heavy, ít write?
   │  └─ Cache Aside + TTL jitter
   ├─ Write-heavy, cần consistency?
   │  └─ Write Through (chấp nhận chậm)
   ├─ Write cực nhiều, chấp nhận eventual?
   │  └─ Write Behind + BullMQ
   ├─ Cần survive race condition?
   │  └─ Cache Aside + Outbox pattern
   └─ Multi-instance?
      └─ Distributed Lock + Redlock (nếu critical)
```

### 14.2. Checklist Implementation

**Cache Aside:**
- [ ] `CacheService` với `getOrSet()`, `invalidate()`
- [ ] Invalidate sau khi write DB thành công
- [ ] Dùng SCAN thay vì KEYS cho pattern delete
- [ ] Handle race condition (double-delete hoặc outbox)

**TTL:**
- [ ] TTL phù hợp cho từng data type
- [ ] Thêm jitter (10-20%) để tránh synchronized expiry
- [ ] Không dùng TTL = 0 (vô hạn) trừ khi thật cần

**Stampede:**
- [ ] Distributed lock cho hot keys
- [ ] SWR nếu chấp nhận stale
- [ ] Jitter TTL
- [ ] Cache warming cho hot data

**Penetration:**
- [ ] Cache null values với TTL ngắn
- [ ] Bloom filter cho large dataset
- [ ] Rate limiting

**Avalanche:**
- [ ] TTL jitter
- [ ] Multi-level cache (L1 + L2)
- [ ] Circuit breaker cho Redis
- [ ] Cache warming sau deploy

**Persistence:**
- [ ] AOF `everysec` cho durability
- [ ] RDB cho backup
- [ ] Hybrid (RDB + AOF) cho production

**Cluster:**
- [ ] Sentinel cho HA
- [ ] Cluster cho scale > RAM
- [ ] Hash tags cho multi-key ops
- [ ] Monitor `evicted_keys`, `used_memory`

### 14.3. Monitoring Commands

```bash
# Cache hit rate
redis-cli INFO stats | grep keyspace_hits
redis-cli INFO stats | grep keyspace_misses
# hit_rate = hits / (hits + misses)

# Memory
redis-cli INFO memory | grep used_memory_human
redis-cli INFO memory | grep maxmemory_human

# Evictions
redis-cli INFO stats | grep evicted_keys

# Slow queries
redis-cli SLOWLOG GET 10

# Connected clients
redis-cli INFO clients | grep connected_clients

# Replication lag
redis-cli INFO replication | grep master_repl_offset

# Keyspace
redis-cli INFO keyspace
```

### 14.4. Alert thresholds

| Metric | Warning | Critical |
|---|---|---|
| Hit rate | < 80% | < 50% |
| Memory usage | > 80% | > 95% |
| Evicted keys/min | > 100 | > 1000 |
| Slow queries/min | > 10 | > 100 |
| Replication lag | > 1s | > 10s |

### 14.5. Tags

#redis #caching #nestjs #cache-aside #write-through #write-behind #ttl #eviction #stampede #penetration #avalanche #distributed-lock #persistence #cluster #sentinel #obsidian-doc

---

> 📌 **Ghi chú cuối:**
> - Tài liệu này kết hợp lý thuyết + code thực tế từ dự án
> - Khi thêm cache layer mới → cập nhật §14.2 Checklist
> - **Nguyên tắc vàng:** *"Cache is easy, invalidation is hard"*
> - **Defense in depth:** Luôn có nhiều lớp bảo vệ (Redis + SQL + Application logic)