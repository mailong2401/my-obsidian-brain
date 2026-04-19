
---

## 1. TypeORM là gì?

**TypeORM** là Object-Relational Mapping (ORM) cho phép bạn:
- Tương tác với database bằng TypeScript classes thay vì SQL thuần
- Tự động tạo bảng từ Entities
- Hỗ trợ Repository Pattern
- Quản lý quan hệ giữa các bảng (One-to-One, One-to-Many, Many-to-Many)
- Hỗ trợ transactions và migrations 

> Hình dung: TypeORM là **cầu nối** giữa TypeScript classes và database tables. Bạn viết code OOP, nó tự động chuyển thành SQL.

---

## 2. Cài đặt và Cấu hình ban đầu

### Bước 1: Cài đặt dependencies

```bash
# Chọn database bạn dùng:
# MySQL / MariaDB
npm install --save @nestjs/typeorm typeorm mysql2

# PostgreSQL
npm install --save @nestjs/typeorm typeorm pg

# SQL Server (Azure)
npm install --save @nestjs/typeorm typeorm mssql

# SQLite
npm install --save @nestjs/typeorm typeorm sqlite3
```

### Bước 2: Cấu hình trong AppModule

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',           // Loại database
      host: 'localhost',          // Host
      port: 5432,                 // Port
      username: 'postgres',       // Username
      password: 'password',       // Password
      database: 'nestjs_db',      // Tên database
      entities: [__dirname + '/**/*.entity{.ts,.js}'], // Đường dẫn entities
      synchronize: true,          // ⚠️ Chỉ dùng trong development
      logging: true,              // Log SQL queries
    }),
  ],
})
export class AppModule {}
```

> ⚠️ **Cảnh báo**: `synchronize: true` sẽ tự động đồng bộ schema mỗi khi chạy app. **Tuyệt đối không dùng trong production** vì có thể làm mất dữ liệu .

### Bước 3: Dùng file cấu hình riêng (best practice)

```typescript
// config/typeorm.config.ts
import { TypeOrmModuleOptions } from '@nestjs/typeorm';
import { ConfigService } from '@nestjs/config';

export const getTypeOrmConfig = async (
  configService: ConfigService,
): Promise<TypeOrmModuleOptions> => ({
  type: 'postgres',
  host: configService.get('DB_HOST'),
  port: configService.get('DB_PORT'),
  username: configService.get('DB_USERNAME'),
  password: configService.get('DB_PASSWORD'),
  database: configService.get('DB_DATABASE'),
  entities: [__dirname + '/../**/*.entity{.ts,.js}'],
  synchronize: configService.get('NODE_ENV') !== 'production',
  logging: configService.get('NODE_ENV') === 'development',
});

// app.module.ts
@Module({
  imports: [
    ConfigModule.forRoot(),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: getTypeOrmConfig,
    }),
  ],
})
export class AppModule {}
```

---

## 3. Entity - Định nghĩa bảng dữ liệu

**Entity** là class đại diện cho một bảng trong database .

### Entity cơ bản

```typescript
// user.entity.ts
import { 
  Entity, 
  PrimaryGeneratedColumn, 
  Column, 
  CreateDateColumn, 
  UpdateDateColumn 
} from 'typeorm';

@Entity('users')  // Tên bảng trong database
export class User {
  @PrimaryGeneratedColumn()  // Auto-increment ID
  id: number;

  @Column({ length: 100 })   // VARCHAR(100)
  name: string;

  @Column({ unique: true })  // UNIQUE constraint
  email: string;

  @Column({ select: false }) // Không trả về khi query (bảo mật)
  password: string;

  @Column({ type: 'int', default: 0 })
  age: number;

  @Column({ type: 'boolean', default: true })
  isActive: boolean;

  @Column({ type: 'text', nullable: true })
  bio: string;

  @CreateDateColumn()  // Tự động set khi tạo
  createdAt: Date;

  @UpdateDateColumn()  // Tự động update khi sửa
  updatedAt: Date;
}
```

### Các Column types phổ biến

```typescript
@Entity()
export class Product {
  @PrimaryGeneratedColumn('uuid')  // UUID thay vì số
  id: string;

  @Column('varchar', { length: 255 })
  name: string;

  @Column('decimal', { precision: 10, scale: 2 })
  price: number;

  @Column('json', { nullable: true })
  metadata: Record<string, any>;

  @Column('simple-array')
  tags: string[];  // Lưu dưới dạng 'tag1,tag2,tag3'

  @Column('timestamp', { default: () => 'CURRENT_TIMESTAMP' })
  publishedAt: Date;
}
```

---

## 4. Repository Pattern trong NestJS

**Repository Pattern** là cách tương tác với database thông qua các method có sẵn, giúp tách biệt business logic và data access logic .

### Đăng ký Entity trong Module

```typescript
// users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { User } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])], // 👈 Đăng ký repository
  controllers: [UsersController],
  providers: [UsersService],
  exports: [TypeOrmModule], // Export nếu module khác cần dùng User repository
})
export class UsersModule {}
```

### Inject Repository vào Service

```typescript
// users.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private usersRepository: Repository<User>, // 👈 TypeORM repository
  ) {}

  // Các method CRUD sẽ ở đây
}
```

---

## 5. CRUD Operations chi tiết

### CREATE - Tạo mới

```typescript
// users.service.ts
async create(userData: Partial<User>): Promise<User> {
  // Cách 1: Dùng create() + save()
  const newUser = this.usersRepository.create(userData);
  return this.usersRepository.save(newUser);
  
  // Cách 2: Dùng save() trực tiếp
  // return this.usersRepository.save(userData);
  
  // Cách 3: Dùng insert() (nhanh hơn nhưng không trigger hooks)
  // await this.usersRepository.insert(userData);
  // return this.usersRepository.findOneBy({ id: userData.id });
}

// users.controller.ts
@Post()
async create(@Body() createUserDto: CreateUserDto) {
  return this.usersService.create(createUserDto);
}
```

### READ - Đọc dữ liệu

```typescript
// Tìm tất cả
async findAll(): Promise<User[]> {
  return this.usersRepository.find();
}

// Tìm với điều kiện
async findActiveUsers(): Promise<User[]> {
  return this.usersRepository.find({
    where: { isActive: true },
    select: ['id', 'name', 'email'], // Chỉ lấy các field này
    order: { name: 'ASC' },
    skip: 0,
    take: 10, // Limit
  });
}

// Tìm một user theo ID
async findOne(id: number): Promise<User> {
  return this.usersRepository.findOneBy({ id });
}

// Tìm với nhiều điều kiện
async findByEmailAndAge(email: string, age: number): Promise<User | null> {
  return this.usersRepository.findOne({
    where: {
      email: email,
      age: age,
    },
  });
}

// Tìm và throw error nếu không thấy
async findOneOrFail(id: number): Promise<User> {
  return this.usersRepository.findOneByOrFail({ id });
  // Sẽ throw EntityNotFoundError nếu không tìm thấy
}
```

### UPDATE - Cập nhật

```typescript
// Cách 1: Tìm → sửa → lưu
async update(id: number, updateData: Partial<User>): Promise<User> {
  const user = await this.usersRepository.findOneBy({ id });
  if (!user) {
    throw new NotFoundException(`User ${id} not found`);
  }
  
  // Merge dữ liệu cũ và mới
  Object.assign(user, updateData);
  return this.usersRepository.save(user);
}

// Cách 2: Dùng update() (nhanh hơn nhưng không trigger hooks)
async updateFast(id: number, updateData: Partial<User>): Promise<void> {
  await this.usersRepository.update(id, updateData);
}

// Cách 3: Dùng preload (merge và save trong 1 bước)
async updateWithPreload(id: number, updateData: Partial<User>): Promise<User> {
  const user = await this.usersRepository.preload({
    id: id,
    ...updateData,
  });
  
  if (!user) {
    throw new NotFoundException(`User ${id} not found`);
  }
  
  return this.usersRepository.save(user);
}
```

### DELETE - Xóa

```typescript
// Cách 1: Dùng delete()
async remove(id: number): Promise<void> {
  const result = await this.usersRepository.delete(id);
  
  if (result.affected === 0) {
    throw new NotFoundException(`User ${id} not found`);
  }
}

// Cách 2: Tìm trước rồi xóa (có thể dùng hooks)
async removeWithHooks(id: number): Promise<void> {
  const user = await this.usersRepository.findOneBy({ id });
  if (!user) {
    throw new NotFoundException(`User ${id} not found`);
  }
  
  await this.usersRepository.remove(user); // Sẽ trigger @BeforeRemove hooks
}

// Xóa nhiều records
async removeMany(ids: number[]): Promise<void> {
  await this.usersRepository.delete(ids);
}
```

---

## 6. Relations - Quan hệ giữa các bảng

### One-to-Many & Many-to-One

**Ví dụ**: Một User có nhiều Post, mỗi Post thuộc về một User .

```typescript
// user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Post } from './post.entity';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];
}

// post.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne, JoinColumn } from 'typeorm';
import { User } from './user.entity';

@Entity()
export class Post {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  title: string;

  @Column('text')
  content: string;

  @ManyToOne(() => User, (user) => user.posts)
  @JoinColumn({ name: 'user_id' }) // Tên cột foreign key
  author: User;
}
```

### Truy vấn có relation

```typescript
// posts.service.ts
async findAllWithAuthor(): Promise<Post[]> {
  return this.postsRepository.find({
    relations: ['author'], // Load luôn thông tin author
  });
}

async findPostWithAuthor(postId: number): Promise<Post> {
  return this.postsRepository.findOne({
    where: { id: postId },
    relations: ['author'],
  });
}

// Lấy user kèm theo posts của họ
async getUserWithPosts(userId: number): Promise<User> {
  return this.usersRepository.findOne({
    where: { id: userId },
    relations: ['posts'],
  });
}
```

### Many-to-Many

**Ví dụ**: User có nhiều Role, Role có nhiều User .

```typescript
// user.entity.ts
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @ManyToMany(() => Role, (role) => role.users)
  @JoinTable({
    name: 'user_roles',           // Tên bảng trung gian
    joinColumn: { name: 'user_id', referencedColumnName: 'id' },
    inverseJoinColumn: { name: 'role_id', referencedColumnName: 'id' },
  })
  roles: Role[];
}

// role.entity.ts
@Entity()
export class Role {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ unique: true })
  name: string;

  @ManyToMany(() => User, (user) => user.roles)
  users: User[];
}
```

### One-to-One

```typescript
// user.entity.ts
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @OneToOne(() => Profile, (profile) => profile.user, { cascade: true })
  profile: Profile;
}

// profile.entity.ts
@Entity()
export class Profile {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  bio: string;

  @Column()
  avatar: string;

  @OneToOne(() => User, (user) => user.profile)
  @JoinColumn() // Bên nào có @JoinColumn sẽ lưu foreign key
  user: User;
}
```

---

## 7. Transactions - Xử lý giao dịch

Transaction đảm bảo một loạt các thao tác database được thực hiện **all-or-nothing** .

### Cách 1: Dùng QueryRunner (khuyến khích)

```typescript
// orders.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { DataSource, Repository } from 'typeorm';

@Injectable()
export class OrdersService {
  constructor(
    @InjectRepository(Order)
    private ordersRepository: Repository<Order>,
    @InjectRepository(Product)
    private productsRepository: Repository<Product>,
    private dataSource: DataSource,
  ) {}

  async createOrderWithTransaction(userId: number, productId: number, quantity: number): Promise<Order> {
    const queryRunner = this.dataSource.createQueryRunner();
    
    await queryRunner.connect();
    await queryRunner.startTransaction();
    
    try {
      // 1. Kiểm tra tồn kho
      const product = await queryRunner.manager.findOne(Product, {
        where: { id: productId },
      });
      
      if (!product || product.stock < quantity) {
        throw new Error('Insufficient stock');
      }
      
      // 2. Cập nhật tồn kho
      product.stock -= quantity;
      await queryRunner.manager.save(product);
      
      // 3. Tạo đơn hàng
      const order = queryRunner.manager.create(Order, {
        userId,
        productId,
        quantity,
        total: product.price * quantity,
      });
      
      const savedOrder = await queryRunner.manager.save(order);
      
      // 4. Commit transaction
      await queryRunner.commitTransaction();
      
      return savedOrder;
    } catch (error) {
      // Rollback nếu có lỗi
      await queryRunner.rollbackTransaction();
      throw error;
    } finally {
      // Giải phóng queryRunner
      await queryRunner.release();
    }
  }
}
```

### Cách 2: Dùng transactional helper (đơn giản hơn)

```typescript
import { DataSource } from 'typeorm';

async transferMoney(fromUserId: number, toUserId: number, amount: number) {
  await this.dataSource.transaction(async (manager) => {
    // manager là EntityManager trong transaction này
    
    const fromUser = await manager.findOne(User, {
      where: { id: fromUserId },
    });
    const toUser = await manager.findOne(User, {
      where: { id: toUserId },
    });
    
    if (fromUser.balance < amount) {
      throw new Error('Insufficient balance');
    }
    
    fromUser.balance -= amount;
    toUser.balance += amount;
    
    await manager.save(fromUser);
    await manager.save(toUser);
  });
}
```

---

## 8. Query Builder - Câu truy vấn phức tạp

Khi repository methods không đủ, dùng Query Builder .

```typescript
// users.service.ts
async searchUsers(keyword: string, minAge?: number): Promise<User[]> {
  const queryBuilder = this.usersRepository.createQueryBuilder('user');
  
  queryBuilder
    .where('user.name ILIKE :keyword', { keyword: `%${keyword}%` })
    .orWhere('user.email ILIKE :keyword', { keyword: `%${keyword}%` });
  
  if (minAge) {
    queryBuilder.andWhere('user.age >= :minAge', { minAge });
  }
  
  queryBuilder
    .leftJoinAndSelect('user.posts', 'post')
    .orderBy('user.createdAt', 'DESC')
    .limit(20);
  
  return queryBuilder.getMany();
}

// Query phức tạp với subquery
async getTopUsersWithMostPosts(limit: number = 10): Promise<User[]> {
  return this.usersRepository
    .createQueryBuilder('user')
    .leftJoin('user.posts', 'post')
    .select('user.id', 'id')
    .addSelect('user.name', 'name')
    .addSelect('COUNT(post.id)', 'postCount')
    .groupBy('user.id')
    .orderBy('postCount', 'DESC')
    .limit(limit)
    .getRawMany();
}

// Update bằng Query Builder
async incrementViews(postId: number): Promise<void> {
  await this.postsRepository
    .createQueryBuilder()
    .update(Post)
    .set({ views: () => 'views + 1' })
    .where('id = :id', { id: postId })
    .execute();
}
```

---

## 9. Advanced Features

### Soft Delete (Xóa mềm)

```typescript
// user.entity.ts
import { DeleteDateColumn } from 'typeorm';

@Entity()
export class User {
  // ... các column khác
  
  @DeleteDateColumn()
  deletedAt: Date | null;
}

// users.service.ts
async softDelete(id: number): Promise<void> {
  await this.usersRepository.softDelete(id);
}

async restore(id: number): Promise<void> {
  await this.usersRepository.restore(id);
}

// Tìm users chưa bị xóa (mặc định)
async findAllActive(): Promise<User[]> {
  return this.usersRepository.find(); // Tự động exclude soft-deleted
}

// Tìm kể cả soft-deleted
async findAllWithDeleted(): Promise<User[]> {
  return this.usersRepository.find({ withDeleted: true });
}
```

### Subscribers - Listeners tự động

```typescript
// user.subscriber.ts
import {
  EventSubscriber,
  EntitySubscriberInterface,
  InsertEvent,
  UpdateEvent,
} from 'typeorm';
import { User } from './user.entity';

@EventSubscriber()
export class UserSubscriber implements EntitySubscriberInterface<User> {
  listenTo() {
    return User;
  }
  
  async beforeInsert(event: InsertEvent<User>): Promise<void> {
    console.log(`Before insert user: ${event.entity.email}`);
    // Hash password, validate, etc.
  }
  
  async afterInsert(event: InsertEvent<User>): Promise<void> {
    console.log(`User created: ${event.entity.id}`);
    // Gửi email, log, etc.
  }
  
  async beforeUpdate(event: UpdateEvent<User>): Promise<void> {
    console.log(`Before update user: ${event.entity.id}`);
  }
}
```

### Indexes cho performance

```typescript
import { Index } from 'typeorm';

@Entity()
@Index(['email', 'isActive']) // Composite index
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  @Index() // Single column index
  name: string;

  @Column({ unique: true })
  email: string;

  @Column()
  isActive: boolean;
}
```

---

## 10. Best Practices & Common Pitfalls

### ✅ DO - Nên làm

```typescript
// 1. Dùng DTOs để validate data
import { IsEmail, IsString, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @MinLength(2)
  name: string;

  @IsEmail()
  email: string;
}

// 2. Xử lý duplicate key errors
async create(userData: CreateUserDto): Promise<User> {
  try {
    return await this.usersRepository.save(userData);
  } catch (error) {
    if (error.code === '23505') { // PostgreSQL duplicate key
      throw new ConflictException('Email already exists');
    }
    throw error;
  }
}

// 3. Dùng select: false cho sensitive fields
@Column({ select: false })
password: string;

// Khi cần lấy password (login)
const user = await this.usersRepository
  .createQueryBuilder('user')
  .addSelect('user.password')
  .where('user.id = :id', { id: userId })
  .getOne();

// 4. Pagination
async findAll(page: number = 1, limit: number = 10): Promise<PaginatedResult> {
  const [data, total] = await this.usersRepository.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
    order: { createdAt: 'DESC' },
  });
  
  return {
    data,
    total,
    page,
    totalPages: Math.ceil(total / limit),
  };
}
```

### ❌ DON'T - Không nên làm

```typescript
// 1. KHÔNG dùng synchronize trong production
// TypeOrmModule.forRoot({
//   synchronize: true, // ❌ Dễ mất dữ liệu
// })

// 2. KHÔNG query quá nhiều data không cần thiết
// ❌ Bad
async findAll() {
  return this.usersRepository.find(); // Lấy tất cả columns, tất cả rows
}

// ✅ Good
async findAll() {
  return this.usersRepository.find({
    select: ['id', 'name', 'email'],
    take: 100,
  });
}

// 3. KHÔNG để N+1 queries
// ❌ Bad - Mỗi user sẽ query thêm 1 lần để lấy posts
const users = await this.usersRepository.find();
for (const user of users) {
  user.posts = await this.postsRepository.find({ where: { authorId: user.id } });
}

// ✅ Good - Dùng relations
const users = await this.usersRepository.find({
  relations: ['posts'],
});

// 4. KHÔNG để lộ error details cho client
async findOne(id: number) {
  try {
    return await this.usersRepository.findOneByOrFail({ id });
  } catch (error) {
    // ❌ Trả luôn error message
    throw error;
    
    // ✅ Wrap lại
    throw new NotFoundException(`User ${id} not found`);
  }
}
```

---

## 11. Testing với TypeORM

```typescript
// users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UsersService } from './users.service';
import { User } from './user.entity';

describe('UsersService', () => {
  let service: UsersService;
  let repository: Repository<User>;
  
  const mockUser = {
    id: 1,
    name: 'Teo',
    email: 'teo@example.com',
  };
  
  const mockRepository = {
    find: jest.fn().mockResolvedValue([mockUser]),
    findOneBy: jest.fn().mockResolvedValue(mockUser),
    create: jest.fn().mockReturnValue(mockUser),
    save: jest.fn().mockResolvedValue(mockUser),
    delete: jest.fn().mockResolvedValue({ affected: 1 }),
  };
  
  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getRepositoryToken(User),
          useValue: mockRepository,
        },
      ],
    }).compile();
    
    service = module.get<UsersService>(UsersService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });
  
  it('should find all users', async () => {
    const result = await service.findAll();
    expect(result).toEqual([mockUser]);
    expect(repository.find).toHaveBeenCalled();
  });
});
```

---

## 12. Tổng kết nhanh

| Component | Decorator/Method | Mục đích |
|-----------|------------------|----------|
| **Entity** | `@Entity()`, `@Column()` | Định nghĩa bảng |
| **Repository** | `@InjectRepository()` | Inject repository vào service |
| **Module** | `TypeOrmModule.forFeature()` | Đăng ký entity trong module |
| **Find** | `find()`, `findOne()`, `findOneBy()` | Truy vấn dữ liệu |
| **Create** | `create()`, `save()` | Tạo record mới |
| **Update** | `update()`, `preload()`, `save()` | Cập nhật record |
| **Delete** | `delete()`, `softDelete()`, `remove()` | Xóa record |
| **Relations** | `@OneToMany`, `@ManyToOne`, `@ManyToMany` | Định nghĩa quan hệ |
| **Transaction** | `QueryRunner`, `dataSource.transaction()` | Giao dịch |
| **Query Builder** | `createQueryBuilder()` | Truy vấn phức tạp |

---

## 📝 Bài tập thực hành

Tạo file `practice-typeorm.md`:

```markdown
## Bài tập TypeORM

### Bài 1: Todo App với TypeORM
- [ ] Tạo entity `Todo` với các field: id, title, description, isCompleted, createdAt
- [ ] Tạo CRUD hoàn chỉnh với repository pattern
- [ ] Thêm filter lấy todos chưa hoàn thành

### Bài 2: Quan hệ One-to-Many
- [ ] Tạo entity `Category` và `Product` (1 category có nhiều products)
- [ ] Viết API lấy products kèm category info

### Bài 3: Many-to-Many
- [ ] Tạo `Student` và `Course` (nhiều-nhiều)
- [ ] Viết API enroll student vào course

### Bài 4: Transaction
- [ ] Tạo API transfer money giữa 2 users
- [ ] Đảm bảo rollback nếu có lỗi

### Bài 5: Advanced Query
- [ ] Dùng Query Builder tìm products theo price range
- [ ] Implement soft delete cho products
```

---

Bạn muốn mình giải thích thêm phần **Migrations** (quản lý thay đổi schema), **Relations nâng cao**, hay chuyển sang **Prisma** (ORM thay thế)?