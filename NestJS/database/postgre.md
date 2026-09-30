# Tài liệu Obsidian: PostgreSQL trong NestJS — Toàn tập

> **Phạm vi:** ACID, MVCC, Isolation Levels, Index Types, Partitioning, Replication, Connection Pooling, Performance Tuning
> **Liên quan:** [[Product Module]] · [[Cart Module]] · [[Payment Module]] · [[Redis Caching]]

---

## 📑 Mục lục

- [[#1. Tổng quan PostgreSQL]]
- [[#2. ACID & MVCC]]
- [[#3. Isolation Levels]]
- [[#4. Index Types & Chiến lược]]
- [[#5. Partitioning]]
- [[#6. Replication]]
- [[#7. Connection Pooling]]
- [[#8. Performance Tuning]]
- [[#9. TypeORM trong NestJS]]
- [[#10. Monitoring & Maintenance]]
- [[#11. Checklist Production]]

---

## 1. Tổng quan PostgreSQL

### 1.1. Tại sao chọn PostgreSQL?

PostgreSQL là **SGBD (Hệ quản trị CSDL) tuân thủ chuẩn SQL nhất** so với các đối thủ . Các đặc điểm chính:

| Đặc điểm | Mô tả |
|---|---|
| **ACID compliant** | Đảm bảo tính toàn vẹn giao dịch |
| **MVCC** | Multi-Version Concurrency Control — đọc không block ghi |
| **Extensible** | 1000+ extensions (PostGIS, pg_trgm, pgvector...) |
| **Advanced types** | JSONB, Array, Range, UUID, INET, TSVECTOR |
| **Partitioning** | Declarative partitioning (Range, List, Hash) |
| **Replication** | Streaming + Logical replication |

### 1.2. PostgreSQL vs NoSQL

| Tiêu chí | PostgreSQL | MongoDB/Cassandra |
|---|---|---|
| **Consistency** | ACID (strong) | BASE (eventual) |
| **Schema** | Strict (relational) | Flexible (schema-less) |
| **Transactions** | Full ACID | Limited |
| **Use case** | E-commerce, fintech, ERP | Logging, analytics, IoT |

> 💡 **Nguyên tắc:** Hầu hết ứng dụng cần **ACID guarantees** hơn là flexibility của NoSQL .

---

## 2. ACID & MVCC

### 2.1. ACID là gì?

ACID là viết tắt của 4 tính chất mà mọi transaction phải tuân thủ :

| Tính chất | Mô tả | Ví dụ |
|---|---|---|
| **Atomicity** | Transaction là "tất cả hoặc không có gì" | Chuyển tiền: trừ tài khoản A VÀ cộng tài khoản B, nếu 1 bước fail → rollback hết |
| **Consistency** | Transaction đưa DB từ trạng thái hợp lệ sang trạng thái hợp lệ khác | Vi phạm FK constraint → reject transaction |
| **Isolation** | Các transaction đồng thời không ảnh hưởng lẫn nhau | Transaction A không thấy data chưa commit của B |
| **Durability** | Transaction đã commit sẽ được lưu vĩnh viễn | Sau COMMIT thành công, data survive crash |

### 2.2. MVCC — Multi-Version Concurrency Control

PostgreSQL sử dụng MVCC để đạt isolation hiệu quả :

```
Mỗi row trong PostgreSQL có:
  - xmin: transaction ID tạo row
  - xmax: transaction ID xóa/update row

Khi đọc, PostgreSQL kiểm tra:
  - Row visible nếu xmin committed VÀ (xmax chưa commit HOẶC xmax là transaction của mình)
```

**Ưu điểm MVCC:**
- Đọc không block ghi, ghi không block đọc
- Mỗi transaction có snapshot riêng
- COMMIT/ROLLBACK nhanh

**Nhược điểm MVCC:**
- **VACUUM** cần thiết để cleanup dead tuples
- **Bloat** — bảng phình to nếu không vacuum
- Không có covering indexes (PostgreSQL 14 vẫn hạn chế) 

### 2.3. Cấu trúc Row Version

```
┌─────────────────────────────────────────────────┐
│  Row Version Layout                              │
├─────────────────────────────────────────────────┤
│  xmin (creating txid)  │  xmax (deleting txid)  │
│  cmin / cmax           │  ctid (physical addr)  │
│  ... data columns ...                           │
└─────────────────────────────────────────────────┘

Visibility Rules:
- Row visible nếu:
  ✓ xmin < snapshot.xmax (đã commit trước snapshot)
  ✓ xmin committed
  ✓ (xmax == 0 OR xmax >= snapshot.xmax OR xmax aborted)
```

---

## 3. Isolation Levels

### 3.1. So sánh Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|---|---|---|---|---|
| **Read Uncommitted** | ✅ Possible | ✅ Possible | ✅ Possible | ✅ Possible |
| **Read Committed** | ❌ Not possible | ✅ Possible | ✅ Possible | ✅ Possible |
| **Repeatable Read** | ❌ Not possible | ❌ Not possible | ❌ Not possible | ✅ Possible |
| **Serializable** | ❌ Not possible | ❌ Not possible | ❌ Not possible | ❌ Not possible |

> ⚠️ **PostgreSQL không hỗ trợ Read Uncommitted** — tự động升级 lên Read Committed .

### 3.2. Read Committed (Default)

```sql
-- Mỗi statement có snapshot MỚI
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- Thấy data đã commit trước statement này
-- Transaction khác UPDATE + COMMIT
SELECT balance FROM accounts WHERE id = 1;  -- Thấy data MỚI (do snapshot mới)
COMMIT;
```

**Đặc điểm:**
- Snapshot được refresh cho **mỗi statement** 
- UPDATE/DELETE chỉ tìm rows committed trước statement start
- Nếu row đã bị update bởi transaction khác → **wait** cho đến khi transaction đó commit/rollback 

**Khi nào dùng:** Hầu hết ứng dụng (default).

### 3.3. Repeatable Read

```sql
-- Snapshot được fix tại statement đầu tiên
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE id = 1;  -- Snapshot tại đây
-- Transaction khác UPDATE + COMMIT
SELECT balance FROM accounts WHERE id = 1;  -- Vẫn thấy data CŨ
COMMIT;
```

**Đặc điểm:**
- Snapshot fix tại **statement đầu tiên** của transaction 
- Không thấy changes từ transactions committed sau khi bắt đầu
- Có thể gặp **serialization failure** → phải retry 

**Khi nào dùng:** Reports, analytics cần consistent view.

### 3.4. Serializable

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... queries ...
COMMIT;
```

**Đặc điểm:**
- Đảm bảo kết quả như thể transactions chạy tuần tự
- Có thể abort transaction với `serialization_failure` → retry
- Overhead cao nhất

**Khi nào dùng:** Critical financial operations.

### 3.5. Áp dụng trong NestJS/TypeORM

```typescript
// Per-query
await this.dataSource.transaction('REPEATABLE READ', async (manager) => {
  const account = await manager.findOne(Account, { where: { id } });
  // ... logic ...
});

// Per-query runner
const queryRunner = this.dataSource.createQueryRunner();
await queryRunner.connect();
await queryRunner.startTransaction('SERIALIZABLE');
try {
  // ... operations ...
  await queryRunner.commitTransaction();
} catch (err) {
  await queryRunner.rollbackTransaction();
  throw err;
} finally {
  await queryRunner.release();
}
```

---

## 4. Index Types & Chiến lược

### 4.1. Các loại Index

| Index Type | Use Case | Ví dụ |
|---|---|---|
| **B-tree** | Default, equality + range, ORDER BY | `WHERE price > 100`, `ORDER BY created_at` |
| **Hash** | Equality only (ít dùng) | `WHERE id = 123` |
| **GIN** | Full-text, array, JSONB, trigram | `WHERE tags @> ARRAY['a']`, `WHERE name LIKE '%xyz%'` |
| **GiST** | Geometric, range types, full-text | `WHERE location <-> point` |
| **SP-GiST** | Partitioned search trees | Phone routing, IP routing |
| **BRIN** | Large tables, time-series, correlated data | `WHERE created_at > '2024-01-01'` |
| **BLOOM** | Multi-column arbitrary combinations | `WHERE col1 = x AND col2 = y AND col3 = z` |

### 4.2. B-tree Index — Deep Dive

**Multicolumn index rules** :

```sql
-- Index on (a, b, c)
CREATE INDEX idx_test ON table (a, b, c);

-- ✅ Sử dụng được: WHERE a = 5
-- ✅ Sử dụng được: WHERE a = 5 AND b >= 42
-- ✅ Sử dụng được: WHERE a = 5 AND b = 42 AND c < 77
-- ⚠️ Ít hiệu quả: WHERE b = 42 (phải scan toàn bộ index)
-- ❌ Không dùng: WHERE c = 77
```

> **Nguyên tắc:** Leftmost prefix — cột đầu tiên quan trọng nhất.

**ORDER BY optimization** :

```sql
-- B-tree có thể satisfy ORDER BY mà không cần sort
CREATE INDEX idx_orders_created ON orders (created_at DESC NULLS LAST);

-- Query này dùng index, không cần sort step
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
```

### 4.3. Partial Index

Chỉ index một subset của rows — giảm size, tăng performance :

```sql
-- Chỉ index unbilled orders (chiếm ít nhưng access nhiều)
CREATE INDEX orders_unbilled_idx ON orders (order_nr) 
WHERE billed IS NOT TRUE;

-- Query sử dụng được
SELECT * FROM orders WHERE billed IS NOT TRUE AND order_nr < 10000;
```

**Use case:**
- Exclude "uninteresting" values (soft-deleted rows, archived data)
- Partial unique index — enforce uniqueness trên subset 

### 4.4. GIN Index — JSONB & Full-text

```sql
-- JSONB containment
CREATE INDEX idx_products_attributes ON products USING GIN (attributes);
SELECT * FROM products WHERE attributes @> '{"color": "red"}';

-- Full-text search với pg_trgm
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN (name gin_trgm_ops);
SELECT * FROM products WHERE name LIKE '%iphone%';
```

### 4.5. BRIN Index — Time-series

```sql
-- BRIN cho bảng lớn, data correlated với physical order
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at)
WITH (pages_per_range = 128);

-- Hiệu quả cho: WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
-- BRIN lưu min/max values per block range → loại bỏ blocks không cần scan 
```

### 4.6. Index trong TypeORM

```typescript
@Entity('products')
@Index(['status', 'createdAt'])           // Composite
@Index(['category'])                       // Single
@Index('idx_products_name_trgm', { synchronize: false })  // Custom
export class Product {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @Index({ unique: true })
  @Column()
  slug!: string;

  @Index('idx_price_brin', { synchronize: false })
  @Column({ type: 'decimal' })
  price!: number;
}
```

**Tạo index thủ công qua migration:**

```typescript
export class AddProductIndexes1700000000000 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    // GIN index cho full-text
    await queryRunner.query(`
      CREATE INDEX CONCURRENTLY idx_products_name_trgm 
      ON products USING GIN (name gin_trgm_ops)
    `);

    // Partial index
    await queryRunner.query(`
      CREATE INDEX CONCURRENTLY idx_orders_unbilled 
      ON orders (order_nr) WHERE billed IS NOT TRUE
    `);

    // BRIN
    await queryRunner.query(`
      CREATE INDEX CONCURRENTLY idx_logs_created_brin 
      ON logs USING BRIN (created_at)
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query('DROP INDEX CONCURRENTLY IF EXISTS idx_products_name_trgm');
    // ...
  }
}
```

> ⚠️ **`CREATE INDEX CONCURRENTLY`** — không lock table, nhưng chậm hơn và không chạy trong transaction .

---

## 5. Partitioning

### 5.1. Tại sao Partitioning?

Khi bảng lớn (hàng trăm triệu rows):
- Query chậm do scan nhiều
- VACUUM/ANALYZE chậm
- Index lớn, khó maintain
- Archive data khó

**Partitioning** chia bảng thành nhiều phần nhỏ hơn, trong suốt với application .

### 5.2. Range Partitioning

```sql
-- Parent table
CREATE TABLE measurement (
  city_id int NOT NULL,
  logdate date NOT NULL,
  peaktemp int,
  unitsales int
) PARTITION BY RANGE (logdate);

-- Partitions
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
  FOR VALUES FROM ('2006-02-01') TO ('2006-03-01');

CREATE TABLE measurement_y2006m03 PARTITION OF measurement
  FOR VALUES FROM ('2006-03-01') TO ('2006-04-01');
```

**Use case:** Time-series data, logs, transactions theo tháng/năm .

### 5.3. List Partitioning

```sql
CREATE TABLE customers (
  id uuid NOT NULL,
  region text NOT NULL,
  name text
) PARTITION BY LIST (region);

CREATE TABLE customers_us PARTITION OF customers FOR VALUES IN ('US');
CREATE TABLE customers_eu PARTITION OF customers FOR VALUES IN ('EU', 'UK');
CREATE TABLE customers_asia PARTITION OF customers FOR VALUES IN ('VN', 'TH', 'SG');
```

**Use case:** Data theo vùng, category, tenant.

### 5.4. Hash Partitioning

```sql
CREATE TABLE users (
  id uuid NOT NULL,
  email text NOT NULL
) PARTITION BY HASH (id);

CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

**Use case:** Phân tán đều data, tránh hot partition.

### 5.5. Partition Pruning

PostgreSQL tự động loại bỏ partitions không cần thiết:

```sql
-- Query chỉ scan measurement_y2006m02
SELECT * FROM measurement WHERE logdate >= '2006-02-15' AND logdate < '2006-02-20';
```

**Config:** `enable_partition_pruning = on` (default) .

### 5.6. Sub-partitioning

```sql
CREATE TABLE measurement_y2006m02 PARTITION OF measurement
  FOR VALUES FROM ('2006-02-01') TO ('2006-03-01')
  PARTITION BY RANGE (peaktemp);

CREATE TABLE measurement_y2006m02_t1 PARTITION OF measurement_y2006m02
  FOR VALUES FROM (0) TO (10);
```

---

## 6. Replication

### 6.1. Streaming vs Logical Replication

| Tiêu chí | Streaming | Logical |
|---|---|---|
| **Scope** | Toàn bộ cluster | Table-level |
| **Format** | Physical WAL | Logical changes (INSERT/UPDATE/DELETE) |
| **Cross-version** | Không | Có (14 → 16) |
| **Selective** | Không | Có (chọn tables) |
| **Use case** | HA, read scaling | Migration, data distribution |

### 6.2. Streaming Replication Setup

**Primary (postgresql.conf):**
```bash
wal_level = replica
max_wal_senders = 5
wal_keep_size = '1GB'
hot_standby = on
```

**Tạo replication user:**
```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secure_password';
```

**Standby:**
```bash
pg_basebackup -h primary-host -U replicator -D /var/lib/postgresql/data \
  -Fp -Xs -P -R
```

### 6.3. Logical Replication Setup

**Publisher:**
```sql
-- wal_level = logical
CREATE PUBLICATION my_pub FOR TABLE orders, customers;
```

**Subscriber:**
```sql
-- Tables phải tồn tại trước
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=publisher-host dbname=mydb user=replicator password=...'
  PUBLICATION my_pub;
```

### 6.4. Monitoring Replication Lag

```sql
-- Trên primary
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Alert nếu lag_bytes > 10MB
```

### 6.5. Failover

```sql
-- Promote standby
SELECT pg_promote();
-- Hoặc: pg_ctl promote
```

> ⚠️ **PostgreSQL 17** cải thiện logical replication failover — logical slots sync qua physical replication .

---

## 7. Connection Pooling

### 7.1. Tại sao cần Connection Pooling?

PostgreSQL tạo **1 process per connection** — tốn memory. 1000 connections = 1000 processes = crash.

### 7.2. PgBouncer

**pgbouncer.ini:**
```ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_addr = 0.0.0.0
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 20
```

**Pool modes:**

| Mode | Mô tả | Use case |
|---|---|---|
| **Session** | Connection giữ đến khi client disconnect | Legacy apps, prepared statements |
| **Transaction** | Connection trả về pool sau mỗi transaction | **Recommended** — balance |
| **Statement** | Connection trả về sau mỗi statement | Ít dùng |

### 7.3. TypeORM + PgBouncer

```typescript
export const AppDataSource = new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,  // Trỏ đến PgBouncer port 6432
  entities: [User, Post],
  migrations: ['src/migrations/*.ts'],
  extra: {
    statement_timeout: 30_000,
    application_name: 'your-app',
  },
  poolSize: 10,  // Cap pool < PgBouncer default_pool_size
});
```

> ⚠️ **Prepared statements** phải tắt khi dùng PgBouncer transaction mode .

### 7.4. NestJS Config

```typescript
TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  inject: [ConfigService],
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    host: config.get('DB_HOST'),
    port: config.get('DB_PORT'),
    username: config.get('DB_USERNAME'),
    password: config.get('DB_PASSWORD'),
    database: config.get('DB_DATABASE'),
    entities: [__dirname + '/**/*.entity{.ts,.js}'],
    synchronize: false,  // ⚠️ LUÔN false ở production
    logging: config.get('NODE_ENV') === 'development',
    extra: {
      max: 20,                    // Pool size
      idleTimeoutMillis: 30_000,
      connectionTimeoutMillis: 5_000,
      statement_timeout: 30_000,
    },
  }),
})
```

---

## 8. Performance Tuning

### 8.1. EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT * FROM orders WHERE customer_id = 42 AND status = 'pending';
```

**Đọc output:**
- `Seq Scan` — full table scan (bad trên bảng lớn)
- `Index Scan` — dùng index (good)
- `Bitmap Heap Scan` — kết hợp nhiều index
- `Nested Loop` — join nhỏ
- `Hash Join` — join lớn
- `Buffers: shared hit=X read=Y` — cache hit ratio

### 8.2. Tìm Slow Queries

```sql
-- Bật pg_stat_statements
CREATE EXTENSION pg_stat_statements;

-- Top 10 slow queries
SELECT query, mean_exec_time, calls, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### 8.3. VACUUM & Autovacuum

```sql
-- Check dead tuples
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Manual vacuum
VACUUM (ANALYZE, VERBOSE) orders;
```

**Tune autovacuum cho high-churn tables:**
```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_analyze_scale_factor = 0.005
);
```

### 8.4. Statistics

```sql
-- Update statistics sau bulk changes
ANALYZE orders;

-- Check statistics target
ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;
```

### 8.5. Bloat Monitoring

```sql
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS total_size,
       pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) AS table_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;
```

---

## 9. TypeORM trong NestJS

### 9.1. Entity Base Class

```typescript
export abstract class BaseEntity {
  @PrimaryGeneratedColumn('uuid')
  id!: string;

  @CreateDateColumn({ name: 'created_at', type: 'timestamptz' })
  createdAt!: Date;

  @UpdateDateColumn({ name: 'updated_at', type: 'timestamptz' })
  updatedAt!: Date;

  @DeleteDateColumn({ name: 'deleted_at', type: 'timestamptz', nullable: true })
  deletedAt!: Date | null;
}
```

### 9.2. Migration System

```bash
# Generate migration
pnpm typeorm migration:generate src/migrations/AddProductIndexes -d src/data-source.ts

# Run migrations
pnpm typeorm migration:run -d src/data-source.ts

# Revert
pnpm typeorm migration:revert -d src/data-source.ts
```

**Production rules** :
- ❌ **KHÔNG BAO GIỜ** dùng `synchronize: true`
- ✅ Luôn dùng migrations
- ✅ `CREATE INDEX CONCURRENTLY` cho production
- ✅ `ANALYZE` sau bulk data changes

### 9.3. Transaction Pattern

```typescript
async createOrder(userId: string, items: OrderItem[]): Promise<Order> {
  return this.dataSource.transaction(async (manager) => {
    // 1. Lock products
    const productIds = items.map(i => i.productId);
    const products = await manager
      .createQueryBuilder(Product, 'p')
      .setLock('pessimistic_write')
      .whereInIds(productIds)
      .getMany();

    // 2. Validate stock
    for (const item of items) {
      const product = products.find(p => p.id === item.productId);
      if (!product || product.stock < item.quantity) {
        throw new BadRequestException(`Insufficient stock for ${item.productId}`);
      }
    }

    // 3. Create order
    const order = manager.create(Order, {
      userId,
      totalAmount: items.reduce((s, i) => s + i.price * i.quantity, 0),
      status: OrderStatus.PENDING,
    });
    await manager.save(order);

    // 4. Decrease stock (atomic)
    for (const item of items) {
      await manager.decrement(Product, { id: item.productId }, 'stock', item.quantity);
    }

    return order;
  });
}
```

### 9.4. Pessimistic Locking

```typescript
// SELECT ... FOR UPDATE
const product = await manager.findOne(Product, {
  where: { id },
  lock: { mode: 'pessimistic_write' },
});

// SELECT ... FOR SHARE
const product = await manager.findOne(Product, {
  where: { id },
  lock: { mode: 'pessimistic_read' },
});
```

---

## 10. Monitoring & Maintenance

### 10.1. Key Metrics

| Metric | Query | Threshold |
|---|---|---|
| **Cache hit ratio** | `SELECT sum(blks_hit)*100/sum(blks_hit+blks_read) FROM pg_stat_database` | > 99% |
| **Dead tuples** | `SELECT sum(n_dead_tup) FROM pg_stat_user_tables` | < 10% |
| **Replication lag** | `SELECT pg_wal_lsn_diff(sent_lsn, replay_lsn) FROM pg_stat_replication` | < 10MB |
| **Active connections** | `SELECT count(*) FROM pg_stat_activity WHERE state = 'active'` | < max_connections * 0.8 |

### 10.2. pg_stat_statements

```sql
-- Top slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
WHERE query NOT LIKE '%pg_stat%'
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### 10.3. Logging Slow Queries

```sql
-- postgresql.conf
log_min_duration_statement = 1000  -- Log queries > 1s
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
```

---

## 11. Checklist Production

### 11.1. Database Setup

- [ ] `synchronize: false` trong TypeORM
- [ ] Connection pooling (PgBouncer hoặc built-in)
- [ ] `max_connections` phù hợp (thường 100-200)
- [ ] `shared_buffers` = 25% RAM
- [ ] `effective_cache_size` = 50-75% RAM
- [ ] `work_mem` phù hợp workload
- [ ] `maintenance_work_mem` cho VACUUM/CREATE INDEX

### 11.2. Schema Design

- [ ] UUID primary keys (tránh sequence contention)
- [ ] Indexes cho foreign keys
- [ ] Partial indexes cho hot queries
- [ ] Composite indexes theo leftmost prefix
- [ ] Partitioning cho large tables

### 11.3. Operations

- [ ] Autovacuum tuned
- [ ] Monitoring (pg_stat_statements, pg_stat_activity)
- [ ] Backup strategy (pg_dump + WAL archiving)
- [ ] Replication setup (streaming hoặc logical)
- [ ] Failover procedure documented

### 11.4. Security

- [ ] SSL connections
- [ ] Strong passwords (scram-sha-256)
- [ ] Principle of least privilege (roles)
- [ ] `pg_hba.conf` restricted
- [ ] Audit logging (pgAudit)

---

## 12. Tóm tắt 1 trang (TL;DR)

```
┌─────────────────────────────────────────────────────────────────┐
│                    POSTGRESQL — TL;DR                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ACID: Atomicity, Consistency, Isolation, Durability            │
│                                                                  │
│  MVCC: Đọc không block ghi, ghi không block đọc                 │
│         → Cần VACUUM để cleanup dead tuples                     │
│                                                                  │
│  ISOLATION LEVELS:                                              │
│    Read Committed (default) → snapshot mỗi statement            │
│    Repeatable Read → snapshot mỗi transaction                   │
│    Serializable → như tuần tự, có thể retry                     │
│                                                                  │
│  INDEX TYPES:                                                   │
│    B-tree → default, equality + range                           │
│    GIN → JSONB, full-text, array                                │
│    GiST → geometric, range types                                │
│    BRIN → time-series, large tables                             │
│    Partial → subset of rows                                     │
│                                                                  │
│  PARTITIONING:                                                  │
│    Range → time-series (theo ngày/tháng)                        │
│    List → category, region                                      │
│    Hash → phân tán đều                                          │
│                                                                  │
│  REPLICATION:                                                   │
│    Streaming → HA, read scaling (toàn cluster)                  │
│    Logical → migration, selective tables                        │
│                                                                  │
│  CONNECTION POOLING:                                            │
│    PgBouncer → transaction mode (recommended)                   │
│    Pool size < default_pool_size                                │
│                                                                  │
│  PERFORMANCE:                                                   │
│    EXPLAIN ANALYZE → tìm bottleneck                             │
│    pg_stat_statements → top slow queries                        │
│    VACUUM/ANALYZE → cleanup + statistics                        │
│    CREATE INDEX CONCURRENTLY → không lock table                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 13. Tags

#postgresql #database #acid #mvcc #isolation-levels #index #partitioning #replication #connection-pooling #pgbouncer #typeorm #nestjs #performance #vacuum #explain-analyze #monitoring #obsidian-doc

---

> 📌 **Ghi chú cuối:**
> - Tài liệu này kết hợp lý thuyết + code thực tế từ dự án
> - Khi thêm index/partition mới → cập nhật §4, §5
> - **Nguyên tắc vàng:** *"EXPLAIN trước, index sau"* — không tạo index mà không analyze
> - **Production rule:** `synchronize: false` + migrations + `CREATE INDEX CONCURRENTLY`