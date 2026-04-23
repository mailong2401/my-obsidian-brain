Tuyệt vời! Đây là phần giải thích **cực kỳ chi tiết** về Dependency Injection trong NestJS, từ khái niệm cơ bản đến cách NestJS triển khai IoC Container.

---

## 1. Dependency Injection là gì? (Hiểu đúng bản chất)

### Vấn đề khi KHÔNG dùng DI

Hãy tưởng tượng bạn có một `UserService` cần gửi email và lưu database.

```typescript
// ❌ CÁCH VIẾT TRUYỀN THỐNG (Tight Coupling - Ràng buộc chặt)
class UserService {
  private emailService: EmailService;
  private databaseService: DatabaseService;
  
  constructor() {
    // Vấn đề 1: UserService PHẢI BIẾT cách tạo EmailService
    this.emailService = new EmailService('smtp.gmail.com', 587, 'user', 'pass');
    
    // Vấn đề 2: UserService PHẢI BIẾT cách tạo DatabaseService
    this.databaseService = new DatabaseService('localhost', 5432, 'mydb', 'admin', 'pass');
  }
  
  async createUser(userData: any) {
    await this.databaseService.save(userData);
    await this.emailService.sendWelcome(userData.email);
  }
}

// Hậu quả:
// 1. KHÓ TEST: Muốn test UserService, bạn PHẢI có EmailService và DatabaseService THẬT
// 2. KHÓ THAY ĐỔI: Muốn đổi sang dùng SendGrid thay vì SMTP? Phải sửa code UserService
// 3. KHÓ TÁI SỬ DỤNG: Mỗi nơi cần UserService phải tự tạo instance phức tạp
```

### Giải pháp với Dependency Injection

```typescript
// ✅ CÁCH VIẾT VỚI DI (Loose Coupling - Ràng buộc lỏng)
interface IEmailService {
  sendWelcome(email: string): Promise<void>;
}

interface IDatabaseService {
  save(data: any): Promise<void>;
}

class UserService {
  // UserService KHÔNG CẦN BIẾT cách tạo dependencies
  // Nó chỉ cần biết "TÔI CẦN" cái gì
  constructor(
    private emailService: IEmailService,    // Ai đó hãy đưa cho tôi
    private databaseService: IDatabaseService // Ai đó hãy đưa cho tôi
  ) {}
  
  async createUser(userData: any) {
    await this.databaseService.save(userData);
    await this.emailService.sendWelcome(userData.email);
  }
}

// LỢI ÍCH:
// 1. DỄ TEST: Có thể inject MockEmailService và MockDatabaseService
// 2. DỄ THAY ĐỔI: Muốn đổi email provider? Tạo class mới implement IEmailService
// 3. DỄ TÁI SỬ DỤNG: Container sẽ tự động tạo và inject dependencies
```

---

## 2. NestJS IoC Container hoạt động như thế nào?

### Sơ đồ hoạt động của DI Container

```
┌─────────────────────────────────────────────────────────────┐
│                     NESTJS IOC CONTAINER                      │
│  (Inversion of Control - "Đảo ngược điều khiển")             │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  📦 Token Registry (Sổ đăng ký)                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Token: 'UserService'      → Class: UserService       │    │
│  │ Token: 'EmailService'     → Class: SmtpEmailService  │    │
│  │ Token: 'DatabaseService'  → Class: PostgresService   │    │
│  │ Token: 'CONFIG_OPTIONS'   → Value: { host: '...' }   │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                   │
│  🔧 Instance Cache (Bể chứa instances - Singleton)           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ UserService instance (đã được khởi tạo)              │    │
│  │ EmailService instance (đã được khởi tạo)             │    │
│  │ DatabaseService instance (đã được khởi tạo)          │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ↓                                   │
│  🏗️ Dependency Graph (Sơ đồ phụ thuộc)                      │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ AppController                                          │    │
│  │    └── cần → UserService                              │    │
│  │              ├── cần → EmailService                   │    │
│  │              └── cần → DatabaseService                │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Code minh họa cách NestJS Container làm việc

```typescript
// 1️⃣ ĐĂNG KÝ DEPENDENCY VÀO CONTAINER

// email.service.ts
import { Injectable } from '@nestjs/common';

@Injectable() // 👈 Decorator này nói với Container: "Hãy quản lý class này"
export class EmailService {
  send(email: string, message: string) {
    console.log(`Sending email to ${email}: ${message}`);
  }
}

// user.service.ts
import { Injectable } from '@nestjs/common';

@Injectable() // 👈 Container cũng quản lý class này
export class UserService {
  // Container sẽ tự động tìm EmailService và inject vào đây
  constructor(private emailService: EmailService) {}
  
  createUser(name: string, email: string) {
    console.log(`Creating user ${name}`);
    this.emailService.send(email, 'Welcome!');
  }
}

// user.controller.ts
import { Controller, Post, Body } from '@nestjs/common';

@Controller('users')
export class UserController {
  // Container cũng tự động inject UserService
  constructor(private userService: UserService) {}
  
  @Post()
  create(@Body() body: any) {
    return this.userService.createUser(body.name, body.email);
  }
}

// 2️⃣ ĐĂNG KÝ VÀO MODULE

// app.module.ts
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';
import { EmailService } from './email.service';

@Module({
  controllers: [UserController], // 👈 Đăng ký controller
  providers: [UserService, EmailService], // 👈 Đăng ký providers (services)
})
export class AppModule {}
```

**Điều gì xảy ra khi app chạy?**

```typescript
// NestJS Container sẽ tự động làm các bước sau:
// (Bạn không phải viết code này, NestJS làm cho bạn)

class NestJSContainer {
  private instances = new Map();
  
  async bootstrap() {
    // 1. Phân tích dependency graph
    // UserController cần UserService
    // UserService cần EmailService
    
    // 2. Khởi tạo từ dưới lên (Bottom-up)
    const emailService = new EmailService();           // Không có dependency
    const userService = new UserService(emailService); // Cần EmailService
    const userController = new UserController(userService); // Cần UserService
    
    // 3. Lưu vào cache để tái sử dụng (Singleton pattern)
    this.instances.set(EmailService, emailService);
    this.instances.set(UserService, userService);
    this.instances.set(UserController, userController);
  }
}
```

---

## 3. Các cách Inject Dependency trong NestJS

### Cách 1: Constructor Injection (Phổ biến nhất)

```typescript
@Injectable()
export class OrderService {
  constructor(
    private userService: UserService,        // TypeScript tự động suy luận type
    private productService: ProductService,
    private paymentService: PaymentService,
    private readonly configService: ConfigService // readonly để tránh sửa đổi
  ) {}
  
  async createOrder(userId: string, productId: string) {
    const user = await this.userService.findById(userId);
    const product = await this.productService.findById(productId);
    // ...
  }
}
```

### Cách 2: Property Injection (Ít dùng hơn)

```typescript
import { Inject } from '@nestjs/common';

@Injectable()
export class ReportService {
  @Inject(UserService)
  private userService: UserService; // Inject qua property
  
  @Inject('CUSTOM_TOKEN')
  private customValue: any;
}
```

### Cách 3: Custom Provider với Token

```typescript
// Khi bạn cần inject KHÔNG PHẢI LÀ CLASS (ví dụ: config object, string, function)

// constants.ts
export const CONFIG_OPTIONS = 'CONFIG_OPTIONS';
export const API_CLIENT = 'API_CLIENT';

// app.module.ts
@Module({
  providers: [
    UserService,
    {
      provide: CONFIG_OPTIONS, // 👈 Token là string
      useValue: {
        apiUrl: 'https://api.example.com',
        timeout: 5000,
        retries: 3
      }
    },
    {
      provide: API_CLIENT,
      useFactory: (config: any) => {
        return new ApiClient(config.apiUrl, config.timeout);
      },
      inject: [CONFIG_OPTIONS] // 👈 Inject dependency vào factory
    },
    {
      provide: 'DatabaseConnection', // Có thể dùng string làm token
      useClass: process.env.NODE_ENV === 'test' 
        ? MockDatabaseService 
        : PostgresDatabaseService
    }
  ]
})
export class AppModule {}

// Sử dụng trong service
@Injectable()
export class UserService {
  constructor(
    @Inject(CONFIG_OPTIONS) private config: any, // 👈 Inject bằng token
    @Inject(API_CLIENT) private apiClient: ApiClient
  ) {}
}
```

---

## 4. Các Scope của Provider (Vòng đời của Dependency)

### Singleton Scope (Mặc định)
```typescript
@Injectable() // Mặc định là Singleton
export class CounterService {
  private count = 0;
  
  increment() {
    return ++this.count;
  }
}

// Kết quả: Mọi nơi inject CounterService đều dùng CHUNG một instance
// UserController: count = 1
// OrderController: count = 2 (tăng tiếp)
// ProductController: count = 3 (tăng tiếp)
```

### Request Scope (Mỗi request một instance riêng)
```typescript
import { Injectable, Scope } from '@nestjs/common';

@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  private requestId: string;
  private userId: string;
  
  setContext(requestId: string, userId: string) {
    this.requestId = requestId;
    this.userId = userId;
  }
  
  getContext() {
    return { requestId: this.requestId, userId: this.userId };
  }
}

// Mỗi HTTP request sẽ có một instance RequestContextService RIÊNG
// Request A: requestId = 'abc-123', userId = 'user1'
// Request B: requestId = 'def-456', userId = 'user2'
// Hai request không ảnh hưởng lẫn nhau
```

### Transient Scope (Mỗi lần inject là một instance mới)
```typescript
@Injectable({ scope: Scope.TRANSIENT })
export class LoggerService {
  private id = Math.random();
  
  log(message: string) {
    console.log(`[Logger ${this.id}] ${message}`);
  }
}

// Mỗi lần inject vào một class khác sẽ tạo instance mới
// UserService có Logger với id = 0.123
// OrderService có Logger với id = 0.456
```

---

## 5. Custom Provider Patterns (Nâng cao)

### Factory Provider
```typescript
@Module({
  providers: [
    {
      provide: 'REDIS_CLIENT',
      useFactory: async (configService: ConfigService) => {
        const client = redis.createClient({
          host: configService.get('REDIS_HOST'),
          port: configService.get('REDIS_PORT'),
        });
        await client.connect();
        return client;
      },
      inject: [ConfigService], // Inject ConfigService vào factory
    }
  ]
})
```

### Alias Provider (Đặt bí danh)
```typescript
@Module({
  providers: [
    EmailService,
    {
      provide: 'IEmailService', // Interface token
      useExisting: EmailService // Dùng chung instance với EmailService
    }
  ]
})
```

### Value Provider (Inject giá trị tĩnh)
```typescript
@Module({
  providers: [
    {
      provide: 'APP_NAME',
      useValue: 'MyNestJSApp'
    },
    {
      provide: 'SUPPORTED_LANGUAGES',
      useValue: ['en', 'vi', 'ja', 'ko']
    }
  ]
})
```

---

## 6. Dependency Injection trong Testing (Lợi ích lớn nhất)

### Unit Test với Mock Dependencies

```typescript
// user.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserService } from './user.service';
import { EmailService } from './email.service';

describe('UserService', () => {
  let userService: UserService;
  let mockEmailService: jest.Mocked<EmailService>;
  
  beforeEach(async () => {
    // Tạo mock object
    mockEmailService = {
      send: jest.fn().mockResolvedValue(undefined),
    };
    
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: EmailService,
          useValue: mockEmailService, // 👈 Inject mock thay vì thật
        },
      ],
    }).compile();
    
    userService = module.get<UserService>(UserService);
  });
  
  it('should send welcome email when creating user', async () => {
    await userService.createUser('John', 'john@example.com');
    
    // Kiểm tra mock đã được gọi đúng cách
    expect(mockEmailService.send).toHaveBeenCalledWith(
      'john@example.com',
      'Welcome!'
    );
  });
});
```

### E2E Test với Override Provider

```typescript
// app.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from './../src/app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication;
  const mockPaymentService = {
    processPayment: jest.fn().mockResolvedValue({ success: true }),
  };
  
  beforeEach(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    })
      .overrideProvider(PaymentService) // 👈 Ghi đè provider thật
      .useValue(mockPaymentService)      // bằng mock
      .compile();
    
    app = moduleFixture.createNestApplication();
    await app.init();
  });
  
  it('/orders (POST) should use mock payment service', () => {
    return request(app.getHttpServer())
      .post('/orders')
      .send({ productId: '123' })
      .expect(201)
      .expect(() => {
        expect(mockPaymentService.processPayment).toHaveBeenCalled();
      });
  });
});
```

---

## 7. Module System và DI Scope

### Module tổ chức dependencies

```typescript
// user.module.ts
@Module({
  controllers: [UserController],
  providers: [UserService, UserRepository],
  exports: [UserService] // 👈 Export để module khác có thể inject
})
export class UserModule {}

// order.module.ts
@Module({
  imports: [UserModule], // 👈 Import UserModule để dùng UserService
  controllers: [OrderController],
  providers: [OrderService],
})
export class OrderModule {}

// order.service.ts
@Injectable()
export class OrderService {
  constructor(
    private userService: UserService // 👈 Inject từ UserModule đã import
  ) {}
}
```

### Dynamic Module (Cấu hình linh hoạt)

```typescript
// database.module.ts
import { Module, DynamicModule } from '@nestjs/common';

@Module({})
export class DatabaseModule {
  static forRoot(options: DatabaseOptions): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'DATABASE_OPTIONS',
          useValue: options,
        },
        {
          provide: 'DATABASE_CONNECTION',
          useFactory: (options: DatabaseOptions) => {
            return createConnection(options);
          },
          inject: ['DATABASE_OPTIONS'],
        },
      ],
      exports: ['DATABASE_CONNECTION'],
    };
  }
  
  static forFeature(entities: any[]): DynamicModule {
    return {
      module: DatabaseModule,
      providers: [
        {
          provide: 'REPOSITORIES',
          useFactory: (connection: Connection) => {
            return entities.map(entity => connection.getRepository(entity));
          },
          inject: ['DATABASE_CONNECTION'],
        },
      ],
      exports: ['REPOSITORIES'],
    };
  }
}

// Sử dụng trong app.module.ts
@Module({
  imports: [
    DatabaseModule.forRoot({
      host: 'localhost',
      port: 5432,
      username: 'admin',
      password: 'secret',
      database: 'myapp',
    }),
    DatabaseModule.forFeature([User, Order, Product]),
  ],
})
export class AppModule {}
```

---

## 8. Advanced DI Patterns trong NestJS

### Circular Dependency (Phụ thuộc vòng tròn)

```typescript
// ❌ Lỗi: Circular dependency
@Injectable()
export class UserService {
  constructor(private orderService: OrderService) {} // UserService cần OrderService
}

@Injectable()
export class OrderService {
  constructor(private userService: UserService) {} // OrderService cần UserService
}

// ✅ Giải pháp: Dùng forwardRef
import { forwardRef, Inject } from '@nestjs/common';

@Injectable()
export class UserService {
  constructor(
    @Inject(forwardRef(() => OrderService))
    private orderService: OrderService
  ) {}
}

@Injectable()
export class OrderService {
  constructor(
    @Inject(forwardRef(() => UserService))
    private userService: UserService
  ) {}
}
```

### Optional Dependencies

```typescript
import { Optional } from '@nestjs/common';

@Injectable()
export class LoggerService {
  constructor(
    @Optional() private analyticsService?: AnalyticsService
  ) {}
  
  log(message: string) {
    console.log(message);
    this.analyticsService?.track('log', { message }); // Gọi nếu tồn tại
  }
}
```

### Multiple Providers với cùng Token

```typescript
// Định nghĩa custom decorator
export const STRATEGIES = 'STRATEGIES';

export function Strategy(name: string) {
  return applyDecorators(
    Injectable(),
    SetMetadata('strategy:name', name),
  );
}

// Module
@Module({
  providers: [
    EmailStrategy,
    SmsStrategy,
    PushStrategy,
    {
      provide: STRATEGIES,
      useFactory: (...strategies: NotificationStrategy[]) => strategies,
      inject: [EmailStrategy, SmsStrategy, PushStrategy],
    },
  ],
})
export class NotificationModule {}

// Service sử dụng
@Injectable()
export class NotificationService {
  constructor(
    @Inject(STRATEGIES) private strategies: NotificationStrategy[]
  ) {}
  
  async send(type: string, message: string) {
    const strategy = this.strategies.find(s => s.canHandle(type));
    await strategy.send(message);
  }
}
```

---

## 9. DI Performance & Best Practices

### ✅ NÊN làm

```typescript
// 1. Dùng constructor injection
@Injectable()
export class BestPracticeService {
  constructor(
    private readonly dep1: Dep1Service,  // ✅ readonly để tránh sửa đổi
    private readonly dep2: Dep2Service   // ✅ TypeScript tự suy luận type
  ) {}
}

// 2. Tách module hợp lý
@Module({
  imports: [UserModule, OrderModule], // Mỗi module quản lý một domain
  controllers: [AppController],
  providers: [AppService],
})

// 3. Dùng custom provider cho config
@Module({
  providers: [
    {
      provide: 'MAIL_CONFIG',
      useFactory: (config: ConfigService) => config.get('mail'),
      inject: [ConfigService],
    }
  ]
})
```

### ❌ KHÔNG NÊN làm

```typescript
// 1. Không nên dùng property injection trừ khi bắt buộc
@Injectable()
export class BadService {
  @Inject()
  private dep1: Dep1Service; // ❌ Khó test và không rõ ràng
}

// 2. Không nên tạo circular dependency
// Nếu có circular dependency, nghĩa là thiết kế module chưa tốt

// 3. Không nên inject quá nhiều dependencies (>5)
@Injectable()
export class GodService {
  constructor(
    private s1: Service1,
    private s2: Service2,
    private s3: Service3,
    private s4: Service4,
    private s5: Service5,
    private s6: Service6, // ❌ Quá nhiều! Nên tách nhỏ class
  ) {}
}
```

---

## 10. Debug DI Issues

### Debug xem NestJS đang inject cái gì

```typescript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['debug', 'error', 'warn'], // Bật debug log
  });
  
  // Xem toàn bộ dependency graph
  const container = (app as any).container;
  console.log(container.getModules());
  
  await app.listen(3000);
}
```

### Xử lý lỗi phổ biến

```typescript
// Lỗi 1: Nest can't resolve dependencies
// Nguyên nhân: Quên thêm @Injectable() hoặc quên thêm vào providers

// ❌ Sai
export class EmailService { // Quên @Injectable()
  send() {}
}

@Module({
  providers: [UserService], // Quên thêm EmailService
})

// ✅ Đúng
@Injectable() // Phải có
export class EmailService {
  send() {}
}

@Module({
  providers: [UserService, EmailService], // Phải khai báo
})

// Lỗi 2: Dependency not found
// Nguyên nhân: Module không export hoặc không import

// ❌ Sai
// user.module.ts
@Module({
  providers: [UserService],
  // Quên exports: [UserService]
})

// order.module.ts
@Module({
  // Quên imports: [UserModule]
})

// ✅ Đúng
// user.module.ts
@Module({
  providers: [UserService],
  exports: [UserService], // Phải export
})

// order.module.ts
@Module({
  imports: [UserModule], // Phải import
})
```

---

## Tổng kết: Flow hoàn chỉnh của DI trong NestJS

```
1. KHỞI TẠO APP
   └── NestFactory.create(AppModule)
       └── Container quét tất cả @Module()

2. PHÂN TÍCH DEPENDENCIES
   └── Duyệt qua providers trong mỗi module
       └── Xây dựng dependency graph
           └── Phát hiện circular dependency

3. KHỞI TẠO INSTANCES (Bottom-up)
   └── Tạo instance cho class không có dependency
       └── Inject vào class phụ thuộc nó
           └── Lưu vào singleton cache
               └── Tiếp tục cho đến khi hoàn thành graph

4. XỬ LÝ REQUEST
   └── Request đến Controller
       └── Container lấy Controller instance từ cache
           └── Đã có sẵn tất cả dependencies được inject

5. KẾT THÚC APP
   └── Container gọi lifecycle hooks (@OnApplicationShutdown)
       └── Đóng connections, cleanup resources
```

DI trong NestJS không chỉ là một tính năng, nó là **triết lý thiết kế cốt lõi** giúp code của bạn:
- **Dễ test**: Mock mọi thứ dễ dàng
- **Dễ maintain**: Thay đổi implementation không ảnh hưởng code dùng nó
- **Dễ scale**: Tách module, quản lý dependencies rõ ràng
- **Dễ hiểu**: Constructor thể hiện rõ class cần những gì
