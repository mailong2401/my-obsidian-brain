# 🔧 Provider & Service trong NestJS - Giải thích chi tiết

Đây là phần **trái tim** của business logic trong NestJS. 

---

## 1. Provider là gì?

**Provider** là bất kỳ class nào được đánh dấu bằng `@Injectable()` và được **NestJS Dependency Injection** quản lý.

Các loại Provider phổ biến:
- **Service** (xử lý logic nghiệp vụ)
- **Repository** (tương tác database)
- **Factory** (tạo đối tượng phức tạp)
- **Helper/Utility** class

> Hình dung: Provider là **đầu bếp** trong nhà hàng - nhận order từ Controller (bồi bàn) và nấu món ăn (xử lý logic).

---

## 2. Service là gì?

**Service** là một loại Provider đặc biệt, chứa **business logic**:
- Xử lý dữ liệu
- Gọi database
- Gọi API bên ngoài
- Tính toán, validate phức tạp

### Tạo Service cơ bản

```typescript
// math.service.ts
import { Injectable } from '@nestjs/common';

@Injectable()  // Đánh dấu class này có thể được inject
export class MathService {
  add(a: number, b: number): number {
    return a + b;
  }

  multiply(a: number, b: number): number {
    return a * b;
  }
  
  isEven(num: number): boolean {
    return num % 2 === 0;
  }
}
```

### Sử dụng Service trong Controller

```typescript
// math.controller.ts
import { Controller, Get, Query } from '@nestjs/common';
import { MathService } from './math.service';

@Controller('math')
export class MathController {
  // Inject MathService qua constructor
  constructor(private readonly mathService: MathService) {}
  
  @Get('add')
  add(@Query('a') a: string, @Query('b') b: string) {
    const numA = parseInt(a);
    const numB = parseInt(b);
    
    // Gọi logic từ Service
    const result = this.mathService.add(numA, numB);
    return { result };
  }
  
  @Get('multiply')
  multiply(@Query('a') a: string, @Query('b') b: string) {
    return {
      result: this.mathService.multiply(parseInt(a), parseInt(b))
    };
  }
}
```

---

## 3. Ví dụ thực tế: User Management

### Bước 1: Tạo DTOs

```typescript
// dto/create-user.dto.ts
export class CreateUserDto {
  name: string;
  email: string;
  password: string;
  age: number;
}

// dto/update-user.dto.ts
export class UpdateUserDto {
  name?: string;
  email?: string;
  age?: number;
}
```

### Bước 2: Tạo Service (xử lý logic)

```typescript
// users.service.ts
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

// Interface cho User
export interface User {
  id: number;
  name: string;
  email: string;
  password: string;  // Trong thực tế sẽ hash
  age: number;
  createdAt: Date;
}

@Injectable()
export class UsersService {
  // Giả lập database
  private users: User[] = [];
  private currentId = 1;
  
  // Tìm tất cả users
  findAll(): User[] {
    return this.users;
  }
  
  // Tìm user theo ID
  findOne(id: number): User {
    const user = this.users.find(user => user.id === id);
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }
  
  // Tìm user theo email
  findByEmail(email: string): User | undefined {
    return this.users.find(user => user.email === email);
  }
  
  // Tạo user mới
  create(createUserDto: CreateUserDto): User {
    // Kiểm tra email đã tồn tại chưa
    const existingUser = this.findByEmail(createUserDto.email);
    if (existingUser) {
      throw new ConflictException('Email already exists');
    }
    
    const newUser: User = {
      id: this.currentId++,
      ...createUserDto,
      createdAt: new Date(),
    };
    
    this.users.push(newUser);
    return newUser;
  }
  
  // Cập nhật user
  update(id: number, updateUserDto: UpdateUserDto): User {
    const user = this.findOne(id); // Sẽ throw NotFoundException nếu không tìm thấy
    
    // Cập nhật từng field
    Object.assign(user, updateUserDto);
    
    return user;
  }
  
  // Xóa user
  remove(id: number): { message: string } {
    const index = this.users.findIndex(user => user.id === id);
    if (index === -1) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    
    this.users.splice(index, 1);
    return { message: `User ${id} deleted successfully` };
  }
  
  // Logic phức tạp: Lấy users theo độ tuổi
  findByAgeRange(minAge: number, maxAge: number): User[] {
    return this.users.filter(
      user => user.age >= minAge && user.age <= maxAge
    );
  }
  
  // Logic phức tạp: Thống kê
  getStatistics() {
    const total = this.users.length;
    const averageAge = total > 0 
      ? this.users.reduce((sum, user) => sum + user.age, 0) / total 
      : 0;
    
    return {
      totalUsers: total,
      averageAge: Math.round(averageAge * 100) / 100,
      oldestUser: this.users.reduce((oldest, user) => 
        user.age > oldest.age ? user : oldest, 
        this.users[0] || { age: 0 }
      ),
    };
  }
}
```

### Bước 3: Tạo Controller (gọi Service)

```typescript
// users.controller.ts
import { 
  Controller, 
  Get, 
  Post, 
  Put, 
  Delete, 
  Body, 
  Param, 
  Query,
  ParseIntPipe,
  HttpStatus,
  HttpCode
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')
export class UsersController {
  // Inject UsersService
  constructor(private readonly usersService: UsersService) {}
  
  // GET /users
  @Get()
  findAll() {
    return this.usersService.findAll();
  }
  
  // GET /users/statistics
  @Get('statistics')
  getStatistics() {
    return this.usersService.getStatistics();
  }
  
  // GET /users/age-range?min=18&max=30
  @Get('age-range')
  findByAgeRange(
    @Query('min', ParseIntPipe) min: number,
    @Query('max', ParseIntPipe) max: number
  ) {
    return this.usersService.findByAgeRange(min, max);
  }
  
  // GET /users/123
  @Get(':id')
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.findOne(id);
  }
  
  // POST /users
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }
  
  // PUT /users/123
  @Put(':id')
  update(
    @Param('id', ParseIntPipe) id: number,
    @Body() updateUserDto: UpdateUserDto
  ) {
    return this.usersService.update(id, updateUserDto);
  }
  
  // DELETE /users/123
  @Delete(':id')
  remove(@Param('id', ParseIntPipe) id: number) {
    return this.usersService.remove(id);
  }
}
```

### Bước 4: Đăng ký trong Module

```typescript
// users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],  // Đăng ký Controller
  providers: [UsersService],        // Đăng ký Service (Provider)
  exports: [UsersService]           // Export để module khác dùng
})
export class UsersModule {}
```

---

## 4. Dependency Injection (DI) chi tiết

### Cách 1: Constructor Injection (phổ biến nhất)

```typescript
@Controller('products')
export class ProductsController {
  // TypeScript sẽ tự động tạo property và gán giá trị
  constructor(
    private readonly productsService: ProductsService,
    private readonly loggerService: LoggerService,
    private readonly mailService: MailService
  ) {}
}
```

### Cách 2: Property Injection (ít dùng)

```typescript
@Controller('orders')
export class OrdersController {
  @Inject(OrdersService)
  private readonly ordersService: OrdersService;
  
  @Inject('CONFIG_OPTIONS')
  private readonly config: any;
}
```

### Cách 3: Custom Provider với token

```typescript
// Cung cấp giá trị cụ thể
const connectionProvider = {
  provide: 'DATABASE_CONNECTION',
  useValue: createConnection(),
};

// Factory provider
const configProvider = {
  provide: 'CONFIG',
  useFactory: () => {
    return {
      apiKey: process.env.API_KEY,
      env: process.env.NODE_ENV
    };
  },
};

// Class provider (mặc định)
@Module({
  providers: [
    UsersService,  // Tương đương { provide: UsersService, useClass: UsersService }
    connectionProvider,
    configProvider,
  ]
})
export class AppModule {}
```

### Sử dụng custom provider

```typescript
// Trong Service
@Injectable()
export class AppService {
  constructor(
    @Inject('DATABASE_CONNECTION') private connection: any,
    @Inject('CONFIG') private config: any
  ) {}
}
```

---

## 5. Service gọi Service khác

```typescript
// email.service.ts
@Injectable()
export class EmailService {
  sendWelcomeEmail(email: string, name: string) {
    console.log(`Sending welcome email to ${name} at ${email}`);
    // Logic gửi email thực tế
    return true;
  }
  
  sendGoodbyeEmail(email: string) {
    console.log(`Sending goodbye email to ${email}`);
    return true;
  }
}

// users.service.ts
@Injectable()
export class UsersService {
  constructor(
    private readonly emailService: EmailService,  // Inject EmailService
    private readonly loggerService: LoggerService
  ) {}
  
  create(createUserDto: CreateUserDto): User {
    // Logic tạo user...
    
    // Gửi email chào mừng
    this.emailService.sendWelcomeEmail(
      createUserDto.email, 
      createUserDto.name
    );
    
    this.loggerService.log(`User ${createUserDto.email} created`);
    
    return newUser;
  }
  
  remove(id: number): void {
    const user = this.findOne(id);
    
    // Gửi email tạm biệt
    this.emailService.sendGoodbyeEmail(user.email);
    
    // Xóa user...
  }
}
```

---

## 6. Scope của Provider

```typescript
// Singleton (mặc định) - Dùng chung 1 instance cho toàn app
@Injectable({ scope: Scope.DEFAULT })
export class SingletonService {}

// Request scope - Tạo mới mỗi request
@Injectable({ scope: Scope.REQUEST })
export class RequestScopedService {}

// Transient scope - Tạo mới mỗi lần inject
@Injectable({ scope: Scope.TRANSIENT })
export class TransientService {}
```

```typescript
// Ví dụ thực tế về Request scope (lấy user từ request)
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

@Injectable({ scope: Scope.REQUEST })
export class UserContextService {
  constructor(@Inject(REQUEST) private request: Request) {}
  
  getCurrentUserId(): string {
    return this.request.headers['user-id'] as string;
  }
  
  getAuthToken(): string {
    return this.request.headers.authorization?.split(' ')[1];
  }
}
```

---

## 7. Lifecycle Hooks (Quan trọng cho Service)

```typescript
@Injectable()
export class DatabaseService implements OnModuleInit, OnModuleDestroy, OnApplicationBootstrap, OnApplicationShutdown {
  
  // Khi module được khởi tạo
  async onModuleInit() {
    console.log('DatabaseService module initialized');
    await this.connectToDatabase();
  }
  
  // Sau khi tất cả modules đã khởi tạo xong
  async onApplicationBootstrap() {
    console.log('Application bootstrapped, ready to serve');
    await this.seedData(); // Seed dữ liệu mẫu
  }
  
  // Khi application đang shutdown (graceful shutdown)
  async onModuleDestroy() {
    console.log('Module destroying, cleaning up...');
    await this.closeConnections();
  }
  
  // Khi application shutdown hoàn toàn
  async onApplicationShutdown(signal?: string) {
    console.log(`Application shutdown with signal: ${signal}`);
    await this.cleanupResources();
  }
  
  private async connectToDatabase() {
    // Kết nối database
  }
  
  private async seedData() {
    // Thêm dữ liệu mẫu
  }
  
  private async closeConnections() {
    // Đóng kết nối database
  }
  
  private async cleanupResources() {
    // Dọn dẹp tài nguyên
  }
}
```

---

## 8. Optional Providers

```typescript
import { Injectable, Optional, Inject } from '@nestjs/common';

@Injectable()
export class CacheService {
  constructor(
    @Optional()  // Nếu không có provider này thì nhận undefined
    @Inject('CACHE_OPTIONS')
    private options?: any
  ) {
    if (options) {
      console.log('Cache enabled with options:', options);
    } else {
      console.log('Cache disabled');
    }
  }
}
```

---

## 9. Best Practices & Patterns

### ✅ DO - Nên làm

```typescript
// 1. Service chỉ chứa logic, không chứa HTTP-specific code
@Injectable()
export class GoodUserService {
  async createUser(userData: CreateUserDto) {
    // ✅ Chỉ xử lý data
    const hashedPassword = await bcrypt.hash(userData.password, 10);
    return this.userRepository.save({ ...userData, password: hashedPassword });
  }
}

// 2. Sử dụng interface cho type safety
export interface UserRepository {
  findById(id: number): Promise<User>;
  save(user: User): Promise<User>;
}

// 3. Tách biệt concerns
@Injectable()
export class OrderService {
  constructor(
    private readonly paymentService: PaymentService,
    private readonly emailService: EmailService,
    private readonly inventoryService: InventoryService
  ) {}
}

// 4. Xử lý error rõ ràng
async findUser(id: number): Promise<User> {
  const user = await this.userRepository.findOne(id);
  if (!user) {
    throw new NotFoundException(`User ${id} not found`);
  }
  return user;
}
```

### ❌ DON'T - Không nên làm

```typescript
// 1. KHÔNG gọi HTTP response trong Service
@Injectable()
export class BadUserService {
  createUser(@Res() res: Response) { // ❌ Sai - Service không nên biết đến @Res
    res.status(201).json({});
  }
}

// 2. KHÔNG hardcore logic quá phức tạp trong Controller
@Controller('users')
export class BadController {
  @Post()
  create(@Body() data: any) {
    // ❌ Sai - Logic quá phức tạp nên để trong Service
    if (!data.email.includes('@')) {
      throw new Error('Invalid email');
    }
    const hashed = bcrypt.hashSync(data.password, 10);
    // ... 50 dòng code khác
  }
}

// 3. KHÔNG tạo phụ thuộc vòng tròn
@Injectable()
export class ServiceA {
  constructor(private serviceB: ServiceB) {} // ❌ ServiceA -> ServiceB
}

@Injectable()
export class ServiceB {
  constructor(private serviceA: ServiceA) {} // ❌ ServiceB -> ServiceA (circular)
}
```

---

## 10. Testing Service (JUnit style)

```typescript
// users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersService } from './users.service';
import { NotFoundException, ConflictException } from '@nestjs/common';

describe('UsersService', () => {
  let service: UsersService;
  
  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [UsersService],
    }).compile();
    
    service = module.get<UsersService>(UsersService);
  });
  
  it('should be defined', () => {
    expect(service).toBeDefined();
  });
  
  describe('create', () => {
    it('should create a new user', () => {
      const createUserDto = {
        name: 'Teo',
        email: 'teo@example.com',
        password: '123456',
        age: 25
      };
      
      const user = service.create(createUserDto);
      
      expect(user).toHaveProperty('id');
      expect(user.name).toBe('Teo');
      expect(user.email).toBe('teo@example.com');
    });
    
    it('should throw ConflictException if email exists', () => {
      const createUserDto = {
        name: 'Ty',
        email: 'teo@example.com', // Email đã tồn tại
        password: '123456',
        age: 30
      };
      
      expect(() => service.create(createUserDto)).toThrow(ConflictException);
    });
  });
  
  describe('findOne', () => {
    it('should return user by id', () => {
      const user = service.findOne(1);
      expect(user).toBeDefined();
      expect(user.id).toBe(1);
    });
    
    it('should throw NotFoundException if user not found', () => {
      expect(() => service.findOne(999)).toThrow(NotFoundException);
    });
  });
});
```

---

## 11. Tổng kết nhanh

| Khái niệm | Vai trò | Decorator |
|-----------|---------|-----------|
| **Provider** | Bất kỳ class có thể inject | `@Injectable()` |
| **Service** | Chứa business logic | `@Injectable()` |
| **Dependency Injection** | Tự động quản lý dependencies | Constructor injection |
| **Scope** | Singleton (mặc định), Request, Transient | `@Injectable({ scope })` |
| **Lifecycle Hooks** | Chạy code tại các thời điểm | `onModuleInit()`, `onModuleDestroy()` |

---

## 📝 Bài tập thực hành

Tạo file `practice-service.md`:

```markdown
## Bài tập Service

### Bài 1: Todo Service
Tạo TodoService với các method:
- [ ] `findAll()` - lấy tất cả todos
- [ ] `findOne(id)` - lấy todo theo id
- [ ] `create(createTodoDto)` - tạo todo mới
- [ ] `update(id, updateTodoDto)` - cập nhật todo
- [ ] `remove(id)` - xóa todo
- [ ] `markAsCompleted(id)` - đánh dấu hoàn thành

### Bài 2: Product Service với logic phức tạp
- [ ] `applyDiscount(category, percent)` - giảm giá theo danh mục
- [ ] `getTopSelling(limit)` - lấy sản phẩm bán chạy nhất
- [ ] `calculateInventoryValue()` - tính tổng giá trị tồn kho

### Bài 3: Service gọi Service khác
Tạo AuthService gọi UserService để xác thực user

## Đáp án mẫu
[Code của bạn]
```

---

Bạn muốn mình giải thích thêm phần **Module** hoặc **Middleware** tiếp theo không? Hay muốn chuyển sang phần **Database với TypeORM/Prisma**?