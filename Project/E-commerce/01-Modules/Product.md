# Tài liệu chi tiết Module Product — Giải thích từng dòng code

Tài liệu này giải thích **từng file, từng dòng code** trong module Product của dự án Ecom-BE. Mục tiêu: sau khi đọc xong, bạn hiểu **tại sao** mỗi dòng tồn tại, không chỉ **cái gì** nó làm.

---

## 1. Tổng quan Module Product

Module Product là **trái tim của e-commerce** — nơi quản lý sản phẩm, cho phép user tìm kiếm, lọc, xem chi tiết, và admin CRUD.

### 1.1. Cấu trúc thư mục

```
src/modules/product/
├── dto/
│   ├── create-product.dto.ts    ← Validate input khi tạo sản phẩm
│   ├── query-product.dto.ts     ← Validate input khi list/search sản phẩm
│   └── update-product.dto.ts    ← Validate input khi update sản phẩm
├── product.controller.ts        ← HTTP endpoints
├── product.entity.ts            ← Định nghĩa bảng products trong DB
├── product.module.ts            ← Kết nối các thành phần
└── product.service.ts           ← Business logic
```

### 1.2. Luồng request đi qua module

```
Client (HTTP)
    │
    ▼
ProductController             ← Nhận request, validate input
    │
    ▼
ProductService                ← Xử lý business logic
    │
    ▼
ProductRepository (TypeORM)   ← Truy vấn database
    │
    ▼
PostgreSQL                    ← Lưu trữ dữ liệu
```

Mỗi tầng có trách nhiệm riêng — đây là **Separation of Concerns** trong thực tế.

---

## 2. File `product.entity.ts` — Định nghĩa bảng `products`

File này mô tả **cấu trúc bảng `products`** trong PostgreSQL. TypeORM đọc file này để biết cách map object `Product` trong code sang bảng trong DB.

### 2.1. Import

```typescript
import {
  Column,
  CreateDateColumn,
  Entity,
  Index,
  UpdateDateColumn,
} from 'typeorm';
import { ProductStatus } from 'src/common/enums/product-status.enum';
import { BaseEntity } from 'src/common/entities/base.entity';
```

| Import | Mục đích |
|:---|:---|
| `Column` | Đánh dấu field là column trong bảng |
| `CreateDateColumn` | Auto-set timestamp khi INSERT |
| `UpdateDateColumn` | Auto-update timestamp khi UPDATE |
| `Entity` | Đánh dấu class là entity (map tới bảng) |
| `Index` | Tạo index cho performance |
| `ProductStatus` | Enum trạng thái sản phẩm |
| `BaseEntity` | Class cha chứa `id` (UUIDv7) |

### 2.2. Decorator `@Entity` và `@Index`

```typescript
@Entity('products')
@Index(['status', 'createdAt'])
@Index(['category'])
export class Product extends BaseEntity {
```

**`@Entity('products')`**
- Map class `Product` → bảng `products` trong PostgreSQL.
- Nếu không truyền `'products'`, TypeORM sẽ dùng tên class viết thường → `product`. Đặt tên số nhiều `products` là convention.

**`@Index(['status', 'createdAt'])`**
- **Composite index** trên 2 cột `status` + `created_at`.
- **Tại sao?** Query phổ biến nhất là: "Lấy sản phẩm ACTIVE, sắp xếp theo ngày tạo mới nhất":
  ```sql
  SELECT * FROM products WHERE status = 'active' ORDER BY created_at DESC LIMIT 20;
  ```
- Index này giúp PostgreSQL **không cần full table scan** — cực nhanh.

**`@Index(['category'])`**
- **Single-column index** trên `category`.
- **Tại sao?** Filter theo category là query rất phổ biến: "Lấy sản phẩm thuộc danh mục áo thun".

**`extends BaseEntity`**
- `BaseEntity` cung cấp:
  - `id: string` là `@PrimaryColumn({ type: 'uuid' })`
  - `generateId()` chạy `@BeforeInsert()` để sinh UUIDv7 nếu chưa có.
- UUIDv7 có **timestamp ở đầu** → sắp xếp theo `id` chính là sắp xếp theo thời gian tạo. Đây là lý do cursor pagination có thể dùng `id` thay vì `created_at`.

### 2.3. Field `name`

```typescript
@Column({ name: 'name', type: 'varchar', length: 255 })
name!: string;
```

| Phần | Ý nghĩa |
|:---|:---|
| `@Column` | Field này là column trong bảng |
| `name: 'name'` | Tên column trong DB là `name` (trùng tên field) |
| `type: 'varchar'` | Kiểu dữ liệu là VARCHAR (chuỗi có độ dài giới hạn) |
| `length: 255` | Độ dài tối đa 255 ký tự |
| `name!: string` | Dấu `!` = TypeScript "definite assignment assertion" — báo cho compiler biết field này sẽ được gán giá trị (bởi TypeORM), đừng báo lỗi "chưa khởi tạo" |

### 2.4. Field `slug`

```typescript
@Column({ name: 'slug', type: 'varchar', length: 300, unique: true })
slug!: string;
```

**`slug`** là phiên bản URL-friendly của `name`. Ví dụ:
- `name`: "Áo thun nam Nike"
- `slug`: `"ao-thun-nam-nike"`

**`unique: true`**
- PostgreSQL tạo **UNIQUE constraint** trên cột này.
- **Tại sao?** Đảm bảo không có 2 sản phẩm cùng slug. Nếu cố INSERT trùng → lỗi `duplicate key value`.
- Slug được dùng cho SEO URL: `/products/ao-thun-nam-nike` thay vì `/products/uuid-xxx`.

**`length: 300`**
- Dài hơn `name` (255) vì slug có thể thêm suffix `-1`, `-2` khi trùng.

### 2.5. Field `description`

```typescript
@Column({ name: 'description', type: 'text', nullable: true })
description!: string | null;
```

**`type: 'text'`**
- TEXT không có giới hạn độ dài (khác VARCHAR).
- **Tại sao?** Description có thể dài (mô tả chi tiết sản phẩm).

**`nullable: true`**
- Cho phép `NULL` — sản phẩm có thể không có description.
- **`description!: string | null`** — TypeScript type tương ứng.

### 2.6. Field `price`

```typescript
@Column({ name: 'price', type: 'decimal', precision: 12, scale: 2 })
price!: number;
```

**`type: 'decimal'`**
- **Đây là điểm cực kỳ quan trọng.** KHÔNG dùng `float` hay `double` cho tiền!
- Float có **floating-point error**: `0.1 + 0.2 = 0.30000000000000004` — không chấp nhận được cho tiền.
- Decimal lưu **chính xác** giá trị thập phân.

**`precision: 12`**
- Tổng số chữ số (bao gồm cả phần thập phân).
- Ví dụ: `1234567890.12` — 10 chữ số nguyên + 2 chữ số thập phân = 12.

**`scale: 2`**
- Số chữ số sau dấu phẩy.
- VNĐ thường có đơn vị nhỏ nhất là đồng (0 chữ số thập phân), nhưng scale 2 để sẵn cho các trường hợp cần (khuyến mãi, tính thuế).

**Giá trị tối đa:** `9999999999.99` (~10 tỷ) — đủ cho hầu hết e-commerce.

### 2.7. Field `salePrice`

```typescript
@Column({
  name: 'sale_price',
  type: 'decimal',
  precision: 12,
  scale: 2,
  nullable: true,
})
salePrice!: number | null;
```

**Tại sao có cả `price` và `salePrice`?**
- `price`: giá gốc (giá niêm yết).
- `salePrice`: giá khuyến mãi (nếu đang giảm giá).
- Nếu `salePrice` là `null` → không có khuyến mãi, dùng `price`.
- Nếu `salePrice` có giá trị → giá hiệu lực là `salePrice`.

**Validate:** Ở tầng service, `salePrice` phải `< price` (nếu không, khuyến mãi vô nghĩa).

### 2.8. Field `stock`

```typescript
@Column({ name: 'stock', type: 'int', default: 0 })
stock!: number;
```

**`type: 'int'`**
- Số nguyên (không thập phân) — vì không bán nửa sản phẩm.

**`default: 0`**
- Nếu không truyền khi INSERT → mặc định `0`.
- **Tại sao?** Sản phẩm mới tạo có thể chưa có hàng.

### 2.9. Field `category`

```typescript
@Column({ name: 'category', type: 'varchar', length: 100 })
category!: string;
```

**Tại sao dùng string thay vì foreign key tới bảng `categories`?**
- Đây là **quyết định thiết kế** của Backend 1: category là **string đơn giản** thay vì bảng riêng.
- **Ưu điểm:** Đơn giản, không cần JOIN.
- **Nhược điểm:** Không normalize — đổi tên category phải UPDATE nhiều row.
- **Backend 2** dùng bảng `categories` riêng với foreign key — "chuẩn" hơn nhưng phức tạp hơn.

### 2.10. Field `brand`

```typescript
@Column({ name: 'brand', type: 'varchar', length: 100, nullable: true })
brand!: string | null;
```

Tương tự `category` — string đơn giản, không FK tới bảng `brands`. Có thể `null` (sản phẩm không brand).

### 2.11. Field `images`

```typescript
@Column({ name: 'images', type: 'jsonb', default: () => "'[]'::jsonb" })
images!: string[];
```

**`type: 'jsonb'`**
- Lưu mảng string dưới dạng JSONB (binary JSON) trong PostgreSQL.
- **Tại sao JSONB thay vì bảng `product_images` riêng?**
  - **Đơn giản:** 1 sản phẩm có vài ảnh (thường 3-5 cái), không cần bảng riêng.
  - **Nhanh:** Đọc 1 row là có hết ảnh, không cần JOIN.
  - **PostgreSQL JSONB hỗ trợ index** — có thể query `WHERE images @> '["url"]'` nếu cần.
- **Nhược điểm:** Không enforce được kiểu dữ liệu chặt chẽ, khó query phức tạp.

**`default: () => "'[]'::jsonb"`**
- **QUAN TRỌNG:** Phải dùng **arrow function** `() =>` để TypeORM gọi khi tạo schema.
- Nếu viết `default: "'[]'::jsonb"` (không có arrow) → TypeORM sẽ dùng chuỗi đó làm giá trị default, không phải SQL expression.

### 2.12. Field `status`

```typescript
@Column({
  name: 'status',
  type: 'enum',
  enum: ProductStatus,
  default: ProductStatus.DRAFT,
})
status!: ProductStatus;
```

**`type: 'enum'`**
- PostgreSQL tạo **ENUM type** riêng: `products_status_enum` với các giá trị `'draft', 'active', 'inactive', 'out_of_stock'`.

**`enum: ProductStatus`**
- TypeORM đọc enum từ file `product-status.enum.ts`:
  ```typescript
  export enum ProductStatus {
    DRAFT = 'draft',
    ACTIVE = 'active',
    INACTIVE = 'inactive',
    OUT_OF_STOCK = 'out_of_stock',
  }
  ```

**`default: ProductStatus.DRAFT`**
- Sản phẩm mới tạo mặc định là DRAFT — chưa public.
- Admin phải đổi sang `active` để user thấy.

### 2.13. Field `rating`

```typescript
@Column({
  name: 'rating',
  type: 'decimal',
  precision: 3,
  scale: 2,
  default: 0,
})
rating!: number;
```

**`precision: 3, scale: 2`**
- Tối đa `9.99` — đủ cho rating 0-5 với 2 chữ số thập phân.
- Ví dụ: `4.75` (4.75 sao).

**Field này có thể được update bởi Review module** (chưa có trong dự án).

### 2.14. Field `reviewCount`

```typescript
@Column({ name: 'review_count', type: 'int', default: 0 })
reviewCount!: number;
```

Số lượng review. Denormalized — được tính lại mỗi khi có review mới (thay vì `COUNT(*)` mỗi lần query).

### 2.15. Field `searchVector` — Cột FTS

```typescript
@Column({
  name: 'search_vector',
  type: 'tsvector',
  select: false,
  insert: false,
  update: false,
  nullable: true,
})
searchVector?: string;
```

**Đây là field đặc biệt — đọc kỹ từng option:**

| Option | Giá trị | Ý nghĩa |
|:---|:---|:---|
| `type: 'tsvector'` | | Cột kiểu `tsvector` của PostgreSQL FTS |
| `select: false` | | **KHÔNG** trả về trong query mặc định (`find`, `findOne`) — tiết kiệm băng thông |
| `insert: false` | | TypeORM **KHÔNG** ghi giá trị này khi INSERT — vì PostgreSQL tự sinh |
| `update: false` | | TypeORM **KHÔNG** ghi giá trị này khi UPDATE — vì PostgreSQL tự sinh |
| `nullable: true` | | Cho phép NULL (mặc dù thực tế generated column không null) |
| `searchVector?: string` | | Optional — vì `select: false`, nó không có trong kết quả query bình thường |

**Tại sao cần cột này?**
- Migration tạo column `search_vector` là **GENERATED ALWAYS AS ... STORED**:
  ```sql
  ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('simple', immutable_unaccent(name)), 'A') ||
    setweight(to_tsvector('simple', immutable_unaccent(description)), 'B')
  ) STORED;
  ```
- PostgreSQL **tự động** tính lại mỗi khi `name` hoặc `description` thay đổi.
- TypeORM **không biết** công thức này — nó chỉ cần biết column tồn tại để:
  1. Không cố DROP column khi generate migration mới.
  2. Có thể dùng trong query FTS (`@@`).

**Lưu ý quan trọng:** Nếu bạn KHÔNG khai báo field này trong entity, lần sau chạy `migration:generate`, TypeORM sẽ thấy DB có column `search_vector` mà entity không có → sinh migration **DROP COLUMN** → mất FTS!

### 2.16. Timestamp columns

```typescript
@CreateDateColumn({ name: 'created_at' })
createdAt!: Date;

@UpdateDateColumn({ name: 'updated_at' })
updatedAt!: Date;
```

**`@CreateDateColumn`**
- PostgreSQL: `created_at TIMESTAMP NOT NULL DEFAULT now()`.
- Tự động set khi INSERT — **bạn không cần gán trong code**.
- TypeORM tự bỏ qua field này khi save.

**`@UpdateDateColumn`**
- Tương tự, tự động update khi UPDATE.
- **Lưu ý:** TypeORM dùng `@UpdateDateColumn` cập nhật ở tầng application, không phải trigger DB. Nếu bạn UPDATE bằng raw SQL (`queryRunner.query`), field này **KHÔNG** tự update.

---

## 3. File `dto/create-product.dto.ts` — Validate input tạo sản phẩm

DTO (Data Transfer Object) định nghĩa **cấu trúc dữ liệu client gửi lên** và **validate** nó.

### 3.1. Field `name`

```typescript
@IsString()
@IsNotEmpty()
@MinLength(2)
@MaxLength(255)
@Transform(({ value }) => value?.trim())
name!: string;
```

| Decorator | Kiểm tra gì | Message lỗi |
|:---|:---|:---|
| `@IsString()` | Phải là string | "name must be a string" |
| `@IsNotEmpty()` | Không được rỗng (`''`) | "name should not be empty" |
| `@MinLength(2)` | Tối thiểu 2 ký tự | "name must be longer than or equal to 2 characters" |
| `@MaxLength(255)` | Tối đa 255 ký tự | "name must be shorter than or equal to 255 characters" |
| `@Transform(...)` | **Trim whitespace** trước khi validate | — |

**`@Transform`** chạy **trước** validation. Nếu client gửi `"  Áo thun  "` → transform thành `"Áo thun"` → mới validate. Nếu không có transform, `@MinLength(2)` sẽ pass với `"  "` (2 space).

### 3.2. Field `description`

```typescript
@IsString()
@IsOptional()
@MaxLength(5000)
description?: string;
```

**`@IsOptional()`** — nếu không truyền hoặc `undefined` → skip các validation khác. Nhưng nếu có truyền → phải pass `@IsString()` và `@MaxLength(5000)`.

**`?` sau `description`** — TypeScript optional property. Cho phép không truyền.

**Tại sao max 5000 mà không phải 10000?**
- Giới hạn độ dài để tránh abuse. 5000 ký tự ~1 trang A4, đủ cho mô tả sản phẩm.

### 3.3. Field `price`

```typescript
@Type(() => Number)
@IsNumber({ maxDecimalPlaces: 2 })
@Min(0)
price!: number;
```

**`@Type(() => Number)`**
- **QUAN TRỌNG:** HTTP request body là JSON → khi parse ra, `price: "350000"` có thể là string nếu client gửi string.
- `@Type(() => Number)` chuyển `"350000"` → `350000` (number) **trước khi validate**.
- Nếu thiếu dòng này → `@IsNumber()` sẽ fail vì value là string.

**`@IsNumber({ maxDecimalPlaces: 2 })`**
- Phải là number (không phải NaN).
- Tối đa 2 chữ số thập phân.

**`@Min(0)`**
- Không cho số âm.

### 3.4. Field `salePrice`

```typescript
@Type(() => Number)
@IsNumber({ maxDecimalPlaces: 2 })
@Min(0)
@IsOptional()
salePrice?: number;
```

Tương tự `price` nhưng optional.

**Lưu ý:** Validate `salePrice < price` **không** làm ở DTO mà ở service. Lý do: cần so sánh 2 field, DTO không hỗ trợ dễ dàng (cần custom validator).

### 3.5. Field `stock`

```typescript
@Type(() => Number)
@IsNumber()
@Min(0)
@IsOptional()
stock?: number;
```

**`@IsNumber()`** — không có `maxDecimalPlaces` → cho phép số thập phân? **Không**, `@IsNumber()` chỉ check number. Nhưng vì DB là `int`, nếu client gửi `10.5` → PostgreSQL sẽ round hoặc lỗi.

**Nên sửa thành `@IsInt()`** để chặt chẽ hơn — hiện tại là minor issue.

### 3.6. Field `category`

```typescript
@IsString()
@IsNotEmpty()
@MaxLength(100)
@Transform(({ value }) => value?.trim())
category!: string;
```

Bắt buộc, string, tối đa 100 ký tự, tự trim.

### 3.7. Field `brand`

```typescript
@IsString()
@IsOptional()
@MaxLength(100)
@Transform(({ value }) => value?.trim())
brand?: string;
```

Optional, string, tối đa 100 ký tự.

### 3.8. Field `images`

```typescript
@IsArray()
@IsString({ each: true })
@IsOptional()
images?: string[];
```

**`@IsArray()`** — phải là array.
**`@IsString({ each: true })`** — mỗi phần tử phải là string.

**Tại sao không validate URL?**
- Hiện tại chỉ check string. Có thể thêm `@IsUrl({}, { each: true })` để chặt hơn.
- Hiện tại cho phép cả URL tuyệt đối và relative path.

### 3.9. Field `status`

```typescript
@IsEnum(ProductStatus)
@IsOptional()
status?: ProductStatus;
```

**`@IsEnum(ProductStatus)`** — giá trị phải thuộc enum `ProductStatus`.
- Hợp lệ: `'draft'`, `'active'`, `'inactive'`, `'out_of_stock'`.
- Không hợp lệ: `'pending'`, `'deleted'` → 400 Bad Request.

---

## 4. File `dto/query-product.dto.ts` — Validate input list/search

File này **phức tạp nhất** trong các DTO vì nó handle nhiều tham số filter/pagination.

### 4.1. Enum `PaginationMode`

```typescript
export enum PaginationMode {
  CURSOR = 'cursor',
  OFFSET = 'offset',
}
```

2 chế độ phân trang:
- **CURSOR**: dùng `id` làm con trỏ, hiệu quả cho bảng lớn, không bị "page drift".
- **OFFSET**: dùng `LIMIT` + `OFFSET`, đơn giản nhưng chậm với OFFSET lớn.

### 4.2. Field `search`

```typescript
@IsString()
@IsOptional()
search?: string;
```

Từ khóa tìm kiếm. Optional — nếu không có → list bình thường.

### 4.3. Field `category`, `brand`

```typescript
@IsString()
@IsOptional()
category?: string;

@IsString()
@IsOptional()
brand?: string;
```

Filter theo category và brand. Đơn giản.

### 4.4. Field `minPrice`, `maxPrice`

```typescript
@Type(() => Number)
@IsNumber()
@Min(0)
@IsOptional()
minPrice?: number;

@Type(() => Number)
@IsNumber()
@Min(0)
@IsOptional()
maxPrice?: number;
```

Filter theo khoảng giá. Cả 2 đều optional, cho phép:
- Chỉ `minPrice` → "sản phẩm từ X trở lên".
- Chỉ `maxPrice` → "sản phẩm dưới X".
- Cả 2 → "sản phẩm từ X đến Y".

### 4.5. Field `status`

```typescript
@IsEnum(ProductStatus)
@IsOptional()
status?: ProductStatus;
```

Filter theo status. Admin có thể xem cả `draft`, user chỉ thấy `active`.

### 4.6. Field `sortBy` với default value

```typescript
@IsEnum(ProductSortBy)
@IsOptional()
sortBy?: ProductSortBy = ProductSortBy.CREATED_AT;
```

**Cú pháp `= ProductSortBy.CREATED_AT`**
- Đây là **default value của TypeScript**, không phải của class-validator.
- Khi `class-transformer` khởi tạo instance, field sẽ có giá trị default này.
- Nếu client không truyền `sortBy` → service sẽ dùng `'createdAt'`.

**Enum `ProductSortBy`:**
```typescript
export enum ProductSortBy {
  CREATED_AT = 'createdAt',
  PRICE = 'price',
  RATING = 'rating',
  NAME = 'name',
}
```

### 4.7. Field `order`

```typescript
@IsIn(['ASC', 'DESC'])
@IsOptional()
order?: 'ASC' | 'DESC' = 'DESC';
```

**`@IsIn(['ASC', 'DESC'])`**
- Chỉ chấp nhận 2 giá trị này.
- Nếu client gửi `'asc'` (lowercase) → 400 Bad Request.

**Default `'DESC'`** — sản phẩm mới nhất lên đầu.

### 4.8. Field `paginationMode`

```typescript
@IsEnum(PaginationMode)
@IsOptional()
paginationMode?: PaginationMode = PaginationMode.CURSOR;
```

**Default `CURSOR`** — chế độ hiệu quả cho bảng lớn.

### 4.9. Field `cursor`

```typescript
@IsUUID()
@IsOptional()
cursor?: string;
```

**`@IsUUID()`**
- Phải là UUID hợp lệ (format `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`).
- **Tại sao?** Vì cursor chính là `id` của sản phẩm cuối cùng trong page trước.

### 4.10. Field `page`, `limit`

```typescript
@Type(() => Number)
@IsInt()
@Min(1)
@IsOptional()
page?: number = 1;

@Type(() => Number)
@IsInt()
@Min(1)
@Max(100)
@IsOptional()
limit?: number = 20;
```

**`page`** — chỉ dùng khi `paginationMode = 'offset'`. Default `1`.
**`limit`** — số item mỗi page. Default `20`, max `100`.

**Tại sao max 100?**
- **Bảo vệ server.** Nếu client gửi `limit=1000000` → server query 1 triệu row → treo.
- **Bảo vệ client.** Trả về 1 triệu item → response nặng → client crash.
- **Bảo vệ UX.** Hiếm khi user cần xem hơn 100 item 1 lúc.

---

## 5. File `dto/update-product.dto.ts`

```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateProductDto } from './create-product.dto';

export class UpdateProductDto extends PartialType(CreateProductDto) { }
```

**Chỉ 3 dòng!** Nhưng cực kỳ mạnh mẽ.

**`PartialType(CreateProductDto)`**
- Tự động tạo DTO mới với **tất cả field của `CreateProductDto` đều là optional**.
- Không cần viết lại validation rules.

**Tại sao dùng `PartialType`?**
- **DRY (Don't Repeat Yourself):** Không lặp lại validation.
- **Consistency:** Update và Create cùng rules → không sợ lệch.
- **Maintainability:** Sửa `CreateProductDto` → `UpdateProductDto` tự động sync.

**Ví dụ:** Nếu `CreateProductDto` có `@MinLength(2) name`, thì `UpdateProductDto` cũng có `@MinLength(2)` nhưng `name?` optional.

---

## 6. File `product.controller.ts` — HTTP endpoints

Controller là **tầng mỏng nhất** — chỉ nhận request, gọi service, trả response. **Không có business logic** ở đây.

### 6.1. Import

```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Patch,
  Param,
  Delete,
  Query,
  UseGuards,
  ParseUUIDPipe,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { Throttle } from '@nestjs/throttler';
```

| Decorator | Mục đích |
|:---|:---|
| `@Controller` | Đánh dấu class là controller |
| `@Get`, `@Post`, `@Patch`, `@Delete` | Map HTTP method |
| `@Body` | Lấy request body |
| `@Param` | Lấy URL param (`:id`) |
| `@Query` | Lấy query string (`?search=x`) |
| `@UseGuards` | Áp dụng guard |
| `@ParseUUIDPipe` | Validate param là UUID |
| `@HttpCode`, `@HttpStatus` | Set HTTP status code |
| `@Throttle` | Override rate limit cho endpoint |

### 6.2. Class declaration

```typescript
@Controller('products')
export class ProductController {
  constructor(private readonly productService: ProductService) { }
```

**`@Controller('products')`**
- Tất cả route trong class này có prefix `/products` (sau global prefix `/api`).
- VD: `@Get() findAll()` → `GET /api/products`.

**`constructor(private readonly productService: ProductService)`**
- **Dependency Injection** — NestJS tự inject `ProductService` instance vào.
- `readonly` — không cho phép gán lại (best practice).
- **KHÔNG** cần `@Inject(ProductService)` vì `ProductService` đã được provide trong `ProductModule`.

### 6.3. Endpoint `GET /api/products` — List

```typescript
@Get()
@Throttle({ default: { ttl: 60_000, limit: 100 } })
async findAll(@Query() query: QueryProductDto) {
  return await this.productService.findAll(query);
}
```

**`@Get()`** — endpoint `GET /api/products`.

**`@Throttle({ default: { ttl: 60_000, limit: 100 } })`**
- Override rate limit global (mặc định 10,000 req/phút).
- Endpoint này: 100 req/phút.
- **Tại sao?** List sản phẩm là endpoint công khai (không cần login) → dễ bị spam. Giới hạn 100/phút/user hoặc /IP.

**`@Query() query: QueryProductDto`**
- Lấy tất cả query string, validate qua `QueryProductDto`.
- **Tự động transform:** `?minPrice=100` → `query.minPrice = 100` (number, nhờ `@Type`).

**`return await this.productService.findAll(query)`**
- Gọi service, trả kết quả.
- `await` vì service method là async (truy vấn DB).

### 6.4. Endpoint `GET /api/products/slug/:slug`

```typescript
@Get('slug/:slug')
@Throttle({ default: { ttl: 60_000, limit: 100 } })
async findBySlug(@Param('slug') slug: string) {
  return await this.productService.findBySlug(slug);
}
```

**Route:** `GET /api/products/slug/ao-thun-nam-nike`.

**Tại sao có route riêng thay vì `GET /products/:slug`?**
- **Conflict với `GET /products/:id`:**
  - Nếu viết `GET /products/:slug`, route `/products/abc-def` sẽ match cả 2 (id là UUID cũng là string).
  - **Giải pháp:** Tiền tố `/slug/` để phân biệt rõ ràng.

**Thứ tự route quan trọng:**
- Trong NestJS, route được match theo thứ tự khai báo.
- `@Get('search')` phải đặt **trước** `@Get(':id')`, nếu không `search` sẽ bị hiểu là `id = 'search'`.

### 6.5. Endpoint `GET /api/products/:id`

```typescript
@Get(':id')
@Throttle({ default: { ttl: 60_000, limit: 100 } })
async findOne(@Param('id', ParseUUIDPipe) id: string) {
  return await this.productService.findOne(id);
}
```

**`ParseUUIDPipe`**
- **Tự động validate** param `id` là UUID hợp lệ.
- Nếu client gửi `/products/abc` → 400 Bad Request ngay, không cần vào service.
- **Tại sao quan trọng?** Tránh query DB với id không hợp lệ → lãng phí tài nguyên.

### 6.6. Endpoint `POST /api/products` — Tạo (Admin)

```typescript
@Post()
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
@Throttle({ default: { ttl: 60_000, limit: 30 } })
@HttpCode(HttpStatus.CREATED)
async create(@Body() dto: CreateProductDto) {
  return await this.productService.create(dto);
}
```

**`@UseGuards(JwtAuthGuard, RolesGuard)`**
- **2 guards chạy tuần tự:**
  1. `JwtAuthGuard` — verify JWT, gắn `req.user`.
  2. `RolesGuard` — check role từ `@Roles` decorator.
- Nếu bất kỳ guard nào fail → 401/403, không vào controller.

**`@Roles(UserRole.ADMIN)`**
- Set metadata `roles = ['admin']`.
- `RolesGuard` đọc metadata này và so sánh với `req.user.role`.

**`@Throttle({ default: { ttl: 60_000, limit: 30 } })`**
- Chỉ admin gọi được, nhưng vẫn giới hạn 30/phút để tránh abuse.

**`@HttpCode(HttpStatus.CREATED)`**
- Mặc định POST trả 201 Created, nhưng explicit để rõ ràng.

### 6.7. Endpoint `GET /api/products/search` — FTS

```typescript
@Get('search')
@Throttle({ default: { ttl: 60_000, limit: 100 } })
async search(@Query('q') q: string, @Query('limit') limit?: number) {
  return await this.productService.searchWithRank(q, limit ? +limit : 20);
}
```

**`@Query('q')`** — lấy riêng param `q` (không dùng DTO).
**`@Query('limit')`** — lấy param `limit` optional.

**`limit ? +limit : 20`**
- Nếu `limit` có giá trị → parse sang number (`+limit`).
- Nếu không → default `20`.
- **Tại sao cần `+limit`?** Query string luôn là string, `+` chuyển `"10"` → `10`.

**Tại sao endpoint này đặt TRƯỚC `:id`?**
- Route được match theo thứ tự khai báo.
- Nếu `:id` đứng trước → `/products/search` sẽ match `:id = 'search'` → 400 UUID validation error.

### 6.8. Endpoint `PATCH /api/products/:id` — Update

```typescript
@Patch(':id')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
@Throttle({ default: { ttl: 60_000, limit: 30 } })
async update(
  @Param('id', ParseUUIDPipe) id: string,
  @Body() dto: UpdateProductDto,
) {
  return await this.productService.update(id, dto);
}
```

**`@Patch` thay vì `@Put`?**
- **PATCH** = update một phần (partial update).
- **PUT** = update toàn bộ (replace).
- Ở đây dùng PATCH vì `UpdateProductDto` cho phép chỉ truyền field cần sửa.

### 6.9. Endpoint `PATCH /api/products/:id/status`

```typescript
@Patch(':id/status')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
@Throttle({ default: { ttl: 60_000, limit: 30 } })
async updateStatus(
  @Param('id', ParseUUIDPipe) id: string,
  @Body('status') status: Product['status'],
) {
  return await this.productService.update(id, { status });
}
```

**Route:** `PATCH /api/products/:id/status`.

**`@Body('status')`**
- Lấy riêng field `status` từ body.
- Body có thể là: `{ "status": "active" }`.

**`Product['status']`**
- TypeScript **indexed access type** — lấy type của field `status` trong class `Product`.
- Tương đương `ProductStatus`. Cách viết này tiện khi muốn lấy type của field mà không cần import enum.

**Tại sao có endpoint riêng?**
- Admin thường xuyên đổi status (activate, deactivate) mà không cần đổi cả sản phẩm.
- Endpoint riêng → audit log dễ hơn, permission có thể khác.

### 6.10. Endpoint `DELETE /api/products/:id`

```typescript
@Delete(':id')
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles(UserRole.ADMIN)
@Throttle({ default: { ttl: 60_000, limit: 20 } })
@HttpCode(HttpStatus.NO_CONTENT)
async remove(@Param('id', ParseUUIDPipe) id: string) {
  await this.productService.remove(id);
}
```

**`@HttpCode(HttpStatus.NO_CONTENT)`** — trả 204 No Content (không có body).
**`await` nhưng không `return`** — vì 204 không có body.

**Tại sao 20 req/phút?**
- DELETE nguy hiểm hơn POST/PATCH → giới hạn chặt hơn.

---

## 7. File `product.service.ts` — Business logic

Đây là **file quan trọng nhất** — chứa toàn bộ logic nghiệp vụ.

### 7.1. Interfaces cho pagination

```typescript
export interface CursorPaginatedProducts {
  data: Product[];
  meta: {
    paginationMode: 'cursor';
    limit: number;
    nextCursor: string | null;
    hasNextPage: boolean;
  };
}

export interface OffsetPaginatedProducts {
  data: Product[];
  meta: {
    paginationMode: 'offset';
    total: number;
    page: number;
    limit: number;
    totalPages: number;
  };
}

export type PaginatedProducts =
  | CursorPaginatedProducts
  | OffsetPaginatedProducts;
```

**Discriminated Union:**
- 2 interface có field `paginationMode` khác nhau (`'cursor'` vs `'offset'`).
- TypeScript có thể **narrow down** type dựa vào field này:
  ```typescript
  if (result.meta.paginationMode === 'cursor') {
    // TypeScript biết result.meta có nextCursor, hasNextPage
  }
  ```

### 7.2. Constructor với Dependency Injection

```typescript
@Injectable()
export class ProductService {
  private readonly logger = new Logger(ProductService.name);

  constructor(
    @InjectRepository(Product)
    private readonly productRepository: Repository<Product>,
    private readonly cache: CacheService,
  ) { }
```

**`@Injectable()`** — cho NestJS biết class này có thể được inject.

**`@InjectRepository(Product)`**
- Inject TypeORM Repository cho entity `Product`.
- Repository có sẵn các method: `find`, `findOne`, `save`, `delete`, `createQueryBuilder`, ...

**`private readonly cache: CacheService`**
- `CacheService` được inject tự động (không cần `@Inject`).
- `CacheService` được `RedisModule` cung cấp global.

### 7.3. Method `slugify`

```typescript
private slugify(name: string): string {
  return name
    .toLowerCase()
    .normalize('NFD')
    .replace(/[\u0300-\u036f]/g, '')
    .replace(/đ/g, 'd')
    .replace(/[^a-z0-9\s-]/g, '')
    .trim()
    .replace(/\s+/g, '-')
    .replace(/-+/g, '-');
}
```

**Từng bước:**

1. **`.toLowerCase()`** — "Áo Thun" → "áo thun".
2. **`.normalize('NFD')`** — Unicode decomposition. "á" (1 ký tự) → "a" + "́" (2 ký tự: a + combining acute accent).
3. **`.replace(/[\u0300-\u036f]/g, '')`** — Xóa các combining diacritical marks (dấu). "a" + "́" → "a".
4. **`.replace(/đ/g, 'd')`** — "đ" không phải dấu mà là ký tự riêng → xử lý riêng.
5. **`.replace(/[^a-z0-9\s-]/g, '')`** — Xóa mọi ký tự không phải a-z, 0-9, space, hoặc dấu gạch.
6. **`.trim()`** — Xóa space đầu/cuối.
7. **`.replace(/\s+/g, '-')`** — Space → gạch. "ao thun" → "ao-thun".
8. **`.replace(/-+/g, '-')`** — Nhiều gạch liên tiếp → 1 gạch. "ao--thun" → "ao-thun".

**Ví dụ:** "Áo Thun Nam NIKE!!!" → "ao-thun-nam-nike".

### 7.4. Method `ensureUniqueSlug`

```typescript
private async ensureUniqueSlug(
  baseSlug: string,
  excludeId?: string,
): Promise<string> {
  let slug = baseSlug;
  let counter = 1;
  while (true) {
    const existing = await this.productRepository.findOne({
      where: { slug },
    });
    if (!existing || existing.id === excludeId) return slug;
    slug = `${baseSlug}-${counter++}`;
  }
}
```

**Mục đích:** Đảm bảo slug không trùng với sản phẩm khác.

**Logic:**
- Nếu slug chưa tồn tại → dùng luôn.
- Nếu slug đã tồn tại → thử `slug-1`, `slug-2`, ...
- **`excludeId`** — khi update sản phẩm, slug "trùng" với chính nó → bỏ qua.

**Ví dụ:**
- Sản phẩm A: name = "Áo thun" → slug = "ao-thun".
- Sản phẩm B: name = "Áo thun" → slug thử "ao-thun" → tồn tại → thử "ao-thun-1" → OK.

**Nhược điểm:** Nếu có 1000 sản phẩm cùng tên → loop 1000 lần query DB. Có thể optimize bằng `COUNT` query 1 lần.

### 7.5. Method `create`

```typescript
async create(dto: CreateProductDto): Promise<Product> {
  const baseSlug = this.slugify(dto.name);
  const slug = await this.ensureUniqueSlug(baseSlug);

  if (dto.salePrice && dto.salePrice >= dto.price) {
    throw new BadRequestException('Sale price must be less than price');
  }

  const product = this.productRepository.create({
    ...dto,
    slug,
    images: dto.images ?? [],
    stock: dto.stock ?? 0,
    status: dto.status ?? ProductStatus.DRAFT,
    salePrice: dto.salePrice ?? null,
    brand: dto.brand ?? null,
    description: dto.description ?? null,
  });

  const saved = await this.productRepository.save(product);
  this.logger.log(`Product created: ${saved.id} (${saved.slug})`);
  return saved;
}
```

**Từng bước:**

1. **`slugify(dto.name)`** — Tạo slug từ name.
2. **`ensureUniqueSlug(baseSlug)`** — Đảm bảo unique.
3. **Validate `salePrice < price`** — Không thể làm ở DTO vì cần 2 field.
4. **`this.productRepository.create({...})`**
   - **`create()` KHÔNG ghi vào DB** — chỉ tạo instance `Product` in-memory.
   - **`...dto`** — spread tất cả field từ DTO.
   - **`slug`** — override slug (không có trong DTO).
   - **`images: dto.images ?? []`** — nếu không truyền, dùng `[]`.
   - **`stock: dto.stock ?? 0`** — default 0.
   - **`status: dto.status ?? ProductStatus.DRAFT`** — default DRAFT.
   - **`salePrice: dto.salePrice ?? null`** — null nếu không có.
   - **`brand: dto.brand ?? null`** — null nếu không có.
   - **`description: dto.description ?? null`** — null nếu không có.

5. **`this.productRepository.save(product)`**
   - **`save()` mới thực sự ghi vào DB.**
   - Trả về instance `Product` với `id` đã được sinh (UUIDv7 từ `BaseEntity.generateId()`).
   - `created_at`, `updated_at` được tự động set.

6. **Log** — Log để debug/monitoring.

**Tại sao cần `?? null`?**
- DTO optional → field có thể `undefined`.
- DB column `nullable: true` → OK với `null`, nhưng TypeORM có thể chuyển `undefined` thành `null` hoặc lỗi.
- Explicit `?? null` để đảm bảo.

### 7.6. Method `findAll` — List với filter + pagination

Method **phức tạp nhất**. Đọc kỹ.

```typescript
async findAll(query: QueryProductDto): Promise<PaginatedProducts> {
  const {
    search,
    category,
    brand,
    minPrice,
    maxPrice,
    status,
    sortBy = ProductSortBy.CREATED_AT,
    order = 'DESC',
    paginationMode = PaginationMode.CURSOR,
    cursor,
    page = 1,
    limit = 20,
  } = query;

  const hasSearch = !!search && search.trim().length > 0;
```

**Destructuring với default values:**
- Lấy từng field từ `query`.
- `= ProductSortBy.CREATED_AT` — default nếu `undefined`.
- Lưu ý: default ở đây **chỉ áp dụng khi `undefined`**, không áp dụng khi `null`.

**`hasSearch`**
- `!!search` — chuyển `string | undefined` → `boolean`.
- `search.trim().length > 0` — đảm bảo không phải toàn space.
- Kết quả: `true` nếu có search thực sự.

#### 7.6.1. Offset mode

```typescript
if (paginationMode === PaginationMode.OFFSET) {
  const qb = this.productRepository.createQueryBuilder('p');
  this.applyFilters(qb, {
    search,
    category,
    brand,
    minPrice,
    maxPrice,
    status,
  });

  if (hasSearch) {
    qb.addSelect(
      `ts_rank(p.search_vector, websearch_to_tsquery('simple', immutable_unaccent(:rankQuery)))`,
      'rank',
    ).setParameter('rankQuery', search.trim());
    qb.orderBy('rank', 'DESC');
    const sortField = Object.values(ProductSortBy).includes(sortBy)
      ? sortBy
      : ProductSortBy.CREATED_AT;
    const sortDir = order === 'ASC' ? 'ASC' : 'DESC';
    qb.addOrderBy(`p.${sortField}`, sortDir).addOrderBy('p.id', 'DESC');
  } else {
    // ... không có search → sort thông thường
  }

  qb.skip((page - 1) * limit).take(limit);

  const [data, total] = await qb.getManyAndCount();

  return {
    data,
    meta: {
      paginationMode: 'offset',
      total,
      page,
      limit,
      totalPages: Math.ceil(total / limit),
    },
  };
}
```

**`createQueryBuilder('p')`**
- Tạo query builder với alias `p` (viết tắt của product).
- Mọi column phải reference qua `p.name`, `p.price`, ...

**`applyFilters(qb, {...})`** — helper apply tất cả filter (xem 7.7).

**Khi có search:**
- **`addSelect(ts_rank(...), 'rank')`** — thêm cột `rank` vào SELECT.
  - `ts_rank(search_vector, query)` — tính điểm liên quan (0-1).
  - `websearch_to_tsquery('simple', immutable_unaccent(:rankQuery))` — parse query.
  - `:rankQuery` — parameter binding (chống SQL injection).
  - `'rank'` — alias cho cột computed.
- **`setParameter('rankQuery', search.trim())`** — gán giá trị.
- **`orderBy('rank', 'DESC')`** — sắp xếp theo độ liên quan giảm dần.
- **`addOrderBy(sortField, sortDir).addOrderBy('p.id', 'DESC')`** — secondary sort.
  - Nếu 2 sản phẩm cùng rank → sort theo `price`/`created_at`/...
  - Nếu vẫn bằng → sort theo `id` để **stable** (tránh random giữa các lần query).

**Khi không có search:**
- Sort theo `sortBy` + `order`.
- Cũng có secondary sort `id` để stable.

**`qb.skip((page - 1) * limit).take(limit)`**
- **`skip(N)`** = `OFFSET N`.
- **`take(N)`** = `LIMIT N`.
- Page 1: skip 0, take 20.
- Page 2: skip 20, take 20.

**`getManyAndCount()`**
- **1 query duy nhất** trả về `[data, total]`?
- **KHÔNG** — thực tế TypeORM chạy 2 query:
  1. `SELECT ... LIMIT ... OFFSET ...` — lấy data.
  2. `SELECT COUNT(*) ...` — đếm tổng.
- Trả về `[data, total]` như tuple.

**`totalPages: Math.ceil(total / limit)`**
- Tổng số page = ceil(tổng item / số item mỗi page).
- VD: 100 item, limit 20 → 5 page.

#### 7.6.2. Cursor mode

```typescript
if (hasSearch) {
  throw new BadRequestException(
    'Cursor pagination không hỗ trợ search. Dùng paginationMode=offset.',
  );
}

if (sortBy !== ProductSortBy.CREATED_AT) {
  throw new BadRequestException(
    'Cursor pagination chỉ hỗ trợ sortBy=createdAt. ' +
    'Dùng paginationMode=offset cho sortBy khác.',
  );
}
```

**Tại sao cursor không hỗ trợ search?**
- **Lý do kỹ thuật:** Rank (điểm liên quan) thay đổi giữa các lần query nếu có data mới.
- Cursor giả định thứ tự **ổn định** giữa các page.
- Rank không ổn định → cursor có thể skip hoặc duplicate items.

**Tại sao cursor chỉ hỗ trợ sortBy=createdAt?**
- Cursor dùng `id` (UUIDv7) làm con trỏ.
- UUIDv7 sắp xếp theo **thời gian tạo**.
- Nếu sort theo `price` → `id` không phản ánh thứ tự `price` → cursor sai.

```typescript
const qb = this.productRepository.createQueryBuilder('p');
this.applyFilters(qb, { search, category, brand, minPrice, maxPrice, status });

const sortDir = order === 'ASC' ? 'ASC' : 'DESC';
qb.orderBy('p.id', sortDir);

if (cursor) {
  if (sortDir === 'DESC') {
    qb.andWhere('p.id < :cursor', { cursor });
  } else {
    qb.andWhere('p.id > :cursor', { cursor });
  }
}

qb.take(limit + 1);
const products = await qb.getMany();
const hasNextPage = products.length > limit;
if (hasNextPage) products.pop();

const nextCursor =
  hasNextPage && products.length > 0
    ? products[products.length - 1].id
    : null;
```

**Logic cursor:**

1. **`orderBy('p.id', sortDir)`** — sắp xếp theo `id`.
2. **Nếu có cursor:**
   - `DESC`: lấy `id < cursor` (các item cũ hơn).
   - `ASC`: lấy `id > cursor` (các item mới hơn).
3. **`take(limit + 1)`**
   - **Lấy dư 1 item** — để biết còn page sau không.
   - Nếu lấy đúng `limit`, không biết có next page không.
4. **`products.length > limit`** → có next page.
5. **`products.pop()`** — bỏ item dư.
6. **`nextCursor`** — id của item cuối cùng (dùng cho page sau).

**Tại sao cursor hiệu quả hơn offset?**
- **Offset:** `OFFSET 10000 LIMIT 20` → PostgreSQL phải scan 10000 row rồi mới lấy 20.
- **Cursor:** `WHERE id < 'xxx' LIMIT 20` → dùng index trên `id` → O(log n).

### 7.7. Helper `applyFilters`

```typescript
private applyFilters(
  qb: SelectQueryBuilder<Product>,
  f: {
    search?: string;
    category?: string;
    brand?: string;
    minPrice?: number;
    maxPrice?: number;
    status?: ProductStatus;
  },
): void {
  if (f.search && f.search.trim().length > 0) {
    qb.andWhere(
      `p.search_vector @@ websearch_to_tsquery('simple', immutable_unaccent(:search))`,
      { search: f.search.trim() },
    );
  }

  if (f.category)
    qb.andWhere('p.category = :category', { category: f.category });
  if (f.brand) qb.andWhere('p.brand = :brand', { brand: f.brand });
  if (f.status) qb.andWhere('p.status = :status', { status: f.status });
  if (f.minPrice !== undefined)
    qb.andWhere('p.price >= :minPrice', { minPrice: f.minPrice });
  if (f.maxPrice !== undefined)
    qb.andWhere('p.price <= :maxPrice', { maxPrice: f.maxPrice });
}
```

**Search dùng FTS:**
- **`p.search_vector @@ ...`** — toán tử match của PostgreSQL FTS.
- **`websearch_to_tsquery('simple', immutable_unaccent(:search))`**
  - `websearch_to_tsquery` — parse input tự nhiên (quotes, `-`, `OR`).
  - `'simple'` — text search configuration (không stemming).
  - `immutable_unaccent(...)` — bỏ dấu.
  - `:search` — parameter binding.

**Các filter khác dùng `=` đơn giản.**

**`!== undefined` cho minPrice/maxPrice**
- **Tại sao?** Vì `0` là giá trị hợp lệ.
- Nếu viết `if (f.minPrice)` → `0` bị coi là falsy → không apply filter.
- `!== undefined` chính xác hơn.

### 7.8. Method `searchWithRank` — FTS riêng

```typescript
async searchWithRank(query: string, limit = 20) {
  if (!query || query.trim().length === 0) {
    return { data: [], total: 0 };
  }

  const qb = this.productRepository
    .createQueryBuilder('p')
    .addSelect(
      `ts_rank(p.search_vector, websearch_to_tsquery('simple', immutable_unaccent(:q)))`,
      'rank',
    )
    .addSelect(
      `ts_headline('simple', immutable_unaccent(p.name), websearch_to_tsquery('simple', immutable_unaccent(:q)), 'StartSel=<mark>, StopSel=</mark>, MaxWords=20')`,
      'highlight',
    )
    .where(
      `p.search_vector @@ websearch_to_tsquery('simple', immutable_unaccent(:q))`,
      { q: query.trim() },
    )
    .andWhere('p.status = :status', { status: ProductStatus.ACTIVE })
    .orderBy('rank', 'DESC')
    .addOrderBy('p.created_at', 'DESC')
    .take(limit);

  const { entities, raw } = await qb.getRawAndEntities();

  const data = entities.map((product, idx) => ({
    ...product,
    rank: parseFloat(raw[idx].rank),
    highlight: raw[idx].highlight,
  }));

  return { data, total: data.length };
}
```

**Điểm mới so với `findAll`:**

1. **`addSelect(ts_rank(...), 'rank')`** — tính điểm liên quan.
2. **`addSelect(ts_headline(...), 'highlight')`**
   - **`ts_headline`** — tạo snippet với từ khớp được bôi đậm.
   - **`StartSel=<mark>, StopSel=</mark>`** — HTML tag bọc từ khớp.
   - **`MaxWords=20`** — giới hạn 20 từ trong snippet.
3. **`andWhere('p.status = :status', { status: ProductStatus.ACTIVE })`**
   - **Chỉ search sản phẩm ACTIVE** — user không thấy DRAFT.
4. **`getRawAndEntities()`**
   - **`entities`** — mảng `Product` object (entity).
   - **`raw`** — mảng raw row từ DB (chứa cả computed columns `rank`, `highlight`).
   - **Tại sao cần cả 2?** Vì `rank` và `highlight` là computed columns, không phải field của entity → TypeORM không tự gán vào entity.
5. **`entities.map((product, idx) => ({ ...product, rank: ..., highlight: ... }))`**
   - Merge computed values vào entity.
   - `raw[idx]` — match theo index (TypeORM đảm bảo thứ tự).

**Response:**
```json
{
  "data": [
    {
      "id": "...",
      "name": "Áo thun nam Nike",
      "rank": 0.6079271,
      "highlight": "<mark>Áo</mark> <mark>thun</mark> nam Nike",
      ...
    }
  ],
  "total": 3
}
```

### 7.9. Method `findOne` — Chi tiết + cache

```typescript
async findOne(id: string): Promise<Product> {
  const key = `product:${id}`;

  const cached = await this.cache.get<Product>(key);
  if (cached) return cached;

  const product = await this.productRepository.findOne({ where: { id } });
  if (!product) {
    throw new NotFoundException(`Product with ID "${id}" not found`);
  }

  await this.cache.set(key, product, 300);
  return product;
}
```

**Cache pattern:**

1. **Check cache trước:**
   - `cache.get(key)` → nếu có → trả luôn.
2. **Cache miss → query DB:**
   - Nếu không tìm thấy → throw `NotFoundException` (404).
3. **Set cache với TTL 300s (5 phút).**

**Tại sao cache 5 phút?**
- **Trade-off giữa freshness và performance.**
- Product detail ít thay đổi → cache 5 phút OK.
- Nếu update product → **phải invalidate cache** (xem `update`).

**Tại sao không cache list?**
- List phụ thuộc filter/search → vô số key.
- Cache list dễ stale (sản phẩm mới thêm không xuất hiện).

### 7.10. Method `findBySlug`

```typescript
async findBySlug(slug: string): Promise<Product> {
  const product = await this.productRepository.findOne({ where: { slug } });
  if (!product) {
    throw new NotFoundException(`Product with slug "${slug}" not found`);
  }
  return product;
}
```

**Tại sao không cache?**
- Slug ít được dùng hơn id (chỉ SEO).
- Có thể thêm cache sau nếu cần.

**Tại sao query riêng thay vì filter trong `findOne`?**
- `findOne` nhận `id`, `findBySlug` nhận `slug`.
- Có thể merge: `findOneBy({ id })` hoặc `findOneBy({ slug })`.
- Tách riêng cho rõ ràng.

### 7.11. Method `update`

```typescript
async update(id: string, dto: UpdateProductDto): Promise<Product> {
  const product = await this.findOne(id);

  const newPrice = dto.price ?? Number(product.price);
  const newSalePrice =
    dto.salePrice !== undefined ? dto.salePrice : product.salePrice;
  if (newSalePrice && newSalePrice >= newPrice) {
    throw new BadRequestException('Sale price must be less than price');
  }

  if (dto.name && dto.name !== product.name) {
    const baseSlug = this.slugify(dto.name);
    product.slug = await this.ensureUniqueSlug(baseSlug, id);
  }

  Object.assign(product, dto);
  const saved = await this.productRepository.save(product);
  await this.cache.del(`product:${id}`);
  this.logger.log(`Product updated: ${saved.id}`);
  return saved;
}
```

**Từng bước:**

1. **`findOne(id)`** — lấy product hiện tại.
   - **Lưu ý:** `findOne` có cache → có thể trả về stale data. Trong context update, nên query trực tiếp DB để chắc chắn fresh.
   - **Có thể là bug tiềm ẩn:** Nếu cache stale, validate salePrice có thể sai.

2. **Validate `salePrice < price` với giá trị mới:**
   - `dto.price ?? Number(product.price)` — dùng giá mới nếu có, không thì dùng giá cũ.
   - `dto.salePrice !== undefined ? dto.salePrice : product.salePrice` — tương tự.
   - **`Number(product.price)`** — vì `price` từ cache/DB là string (decimal), cần convert.

3. **Nếu name thay đổi → regenerate slug:**
   - `ensureUniqueSlug(baseSlug, id)` — exclude chính nó.
   - **Tại sao `dto.name !== product.name`?** — Tránh query DB không cần thiết.

4. **`Object.assign(product, dto)`**
   - Merge tất cả field từ DTO vào product.
   - Chỉ những field có trong DTO mới bị override.
   - **Cẩn thận:** Nếu `dto.field = undefined`, field đó **không** bị override (vì `Object.assign` skip undefined).

5. **`save(product)`** — UPDATE vào DB.
   - TypeORM sẽ update tất cả field (trừ `id`, `created_at`).
   - `updated_at` tự động set.

6. **`cache.del(key)`** — **invalidate cache** để lần sau `findOne` query DB mới.

### 7.12. Method `remove`

```typescript
async remove(id: string): Promise<void> {
  const product = await this.findOne(id);
  await this.productRepository.remove(product);
  await this.cache.del('product:${id}');
  this.logger.log(`Product removed: ${id}`);
}
```

**BUG nhỏ:** `'product:${id}'` dùng **nháy đơn** → là string literal, không phải template string.

**Sửa:**
```typescript
await this.cache.del(`product:${id}`);
```

**Tại sao bug này khó phát hiện?**
- `cache.del('product:${id}')` xóa key literal `"product:${id}"`.
- Key này không tồn tại → `del` return 0, không lỗi.
- Nhưng key thật `product:uuid` **không bị xóa** → cache stale.

**Hậu quả:**
- Xóa sản phẩm → cache vẫn còn → `findOne(id)` vẫn trả về product đã xóa.
- API trả 200 với product đã bị xóa → inconsistency.

**Bài học:** Luôn dùng template literal (`\``) khi cần interpolation.

### 7.13. Method `decreaseStock` — Trừ stock

```typescript
async decreaseStock(id: string, quantity: number): Promise<Product> {
  const product = await this.findOne(id);

  if (product.stock < quantity) {
    throw new BadRequestException(
      `Insufficient stock. Available: ${product.stock}, requested: ${quantity}`,
    );
  }

  product.stock -= quantity;
  if (product.stock === 0 && product.status === ProductStatus.ACTIVE) {
    product.status = ProductStatus.OUT_OF_STOCK;
  }

  await this.cache.del('product:${id}');
  return await this.productRepository.save(product);
}
```

**Method này có RACE CONDITION NGHIÊM TRỌNG.**

**Vấn đề:**
1. Request A: đọc `product.stock = 10`.
2. Request B: đọc `product.stock = 10`.
3. Request A: trừ 1 → `stock = 9` → save.
4. Request B: trừ 1 → `stock = 9` → save.
5. **Kết quả:** `stock = 9` thay vì `8`. **Đã oversell 1 sản phẩm.**

**Giải pháp:** Dùng **pessimistic lock** hoặc **optimistic lock**:

```typescript
async decreaseStock(id: string, quantity: number): Promise<Product> {
  return await this.dataSource.transaction(async (manager) => {
    const product = await manager.findOne(Product, {
      where: { id },
      lock: { mode: 'pessimistic_write' },
    });

    if (!product) throw new NotFoundException('Product not found');
    if (product.stock < quantity) {
      throw new BadRequestException('Insufficient stock');
    }

    product.stock -= quantity;
    if (product.stock === 0 && product.status === ProductStatus.ACTIVE) {
      product.status = ProductStatus.OUT_OF_STOCK;
    }

    await this.cache.del(`product:${id}`);
    return manager.save(product);
  });
}
```

**Cần inject `DataSource` vào `ProductService`:**
```typescript
constructor(
  @InjectRepository(Product)
  private readonly productRepository: Repository<Product>,
  private readonly cache: CacheService,
  private readonly dataSource: DataSource,
) {}
```

**Cách hoạt động:**
- `pessimistic_write` → PostgreSQL khóa row (`SELECT ... FOR UPDATE`).
- Request B phải **đợi** Request A commit xong.
- Request B đọc `stock = 9` (sau khi A trừ) → trừ tiếp → `stock = 8`. **Đúng.**

### 7.14. Method `increaseStock`

```typescript
async increaseStock(id: string, quantity: number): Promise<Product> {
  const product = await this.findOne(id);
  product.stock += quantity;
  if (product.stock > 0 && product.status === ProductStatus.OUT_OF_STOCK) {
    product.status = ProductStatus.ACTIVE;
  }
  await this.cache.del('product:${id}');
  return await this.productRepository.save(product);
}
```

**Cũng có race condition tương tự.**

**Cũng có bug nháy đơn** `'product:${id}'`.

**Logic status:**
- Nếu stock > 0 và status = OUT_OF_STOCK → chuyển về ACTIVE.
- **Hợp lý:** Hàng về → sản phẩm active lại.

---

## 8. File `product.module.ts` — Kết nối module

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { Product } from './product.entity';
import { ProductService } from './product.service';
import { ProductController } from './product.controller';

@Module({
  imports: [TypeOrmModule.forFeature([Product])],
  providers: [ProductService],
  controllers: [ProductController],
  exports: [ProductService],
})
export class ProductModule { }
```

### 8.1. `@Module({...})`

Định nghĩa module — đơn vị tổ chức code trong NestJS.

### 8.2. `imports: [TypeOrmModule.forFeature([Product])]`

**`TypeOrmModule.forFeature([Product])`**
- Đăng ký repository cho entity `Product`.
- Sau khi đăng ký, có thể `@InjectRepository(Product)` trong service.
- **Tại sao cần?** TypeORM cần biết entity nào thuộc module nào để inject repository.

### 8.3. `providers: [ProductService]`

- Đăng ký `ProductService` là **provider** (có thể được inject).
- NestJS sẽ tạo instance `ProductService` và quản lý lifecycle.

### 8.4. `controllers: [ProductController]`

- Đăng ký `ProductController` là controller.
- NestJS sẽ map các route từ controller này.

### 8.5. `exports: [ProductService]`

- **Cho phép module khác import `ProductService`.**
- Ví dụ: `CartModule` cần `ProductService` để validate product → import `ProductModule` → dùng được `ProductService`.
- **Nếu không có `exports`:** `ProductService` là private, chỉ dùng trong `ProductModule`.

---

## 9. Tổng kết — Những gì cần cải thiện

### 9.1. Bugs cần sửa ngay

| # | File | Dòng | Bug | Sửa |
|:---|:---|:---|:---|:---|
| 1 | `product.service.ts` | `remove` | `cache.del('product:${id}')` — nháy đơn | Đổi thành template literal |
| 2 | `product.service.ts` | `decreaseStock` | Nháy đơn + race condition | Dùng `pessimistic_write` |
| 3 | `product.service.ts` | `increaseStock` | Nháy đơn + race condition | Dùng `pessimistic_write` |
| 4 | `cart.service.ts` | `removeItem` | `console.log('userid: ', userId)` | Xóa hoặc dùng Logger |

### 9.2. Cải tiến nên làm

1. **Thêm `@IsUrl({}, { each: true })` cho `images`** trong `CreateProductDto`.
2. **Thêm `@IsInt()` cho `stock`** thay vì `@IsNumber()`.
3. **Cache invalidation cho list** — khi tạo/update/xóa product, invalidate cache list.
4. **Thêm `search` vào `ProductSortBy`** enum — hiện tại nếu client truyền `sortBy=search` sẽ bị 400.
5. **Optimize `ensureUniqueSlug`** — dùng 1 query `SELECT COUNT` thay vì loop.
6. **Thêm pagination cho `searchWithRank`** — hiện tại chỉ có `limit`, không có cursor/offset.

### 9.3. Features nên thêm

1. **Related products** — "Sản phẩm tương tự" dựa trên category/brand.
2. **Product reviews** — rating đã có field nhưng chưa có module.
3. **Product variants** — size, color (Backend 2 có `ProductVariant`).
4. **Bulk operations** — import/export CSV, update nhiều product 1 lúc.
5. **Product history** — track thay đổi giá, stock.

---

## 10. Câu hỏi tự kiểm tra

Sau khi đọc xong, bạn nên trả lời được:

1. **Tại sao `price` dùng `decimal` thay vì `float`?**
2. **`select: false` trên `searchVector` có tác dụng gì?**
3. **Tại sao cursor pagination không hỗ trợ search?**
4. **`getRawAndEntities()` khác gì `getMany()`?**
5. **Tại sao `take(limit + 1)` thay vì `take(limit)` trong cursor mode?**
6. **Tại sao `decreaseStock` có race condition? Cách sửa?**
7. **Tại sao route `@Get('search')` phải đặt trước `@Get(':id')`?**
8. **`PartialType(CreateProductDto)` làm gì?**
9. **Tại sao cần `@Type(() => Number)` trong DTO?**
10. **`@Transform(({ value }) => value?.trim())` chạy trước hay sau validation?**

Nếu bạn trả lời được hết → đã hiểu module Product.

Nếu cần tôi giải thích sâu hơn phần nào (ví dụ: FTS hoạt động thế nào, TypeORM query builder, hoặc cách viết test cho module này), cho tôi biết.