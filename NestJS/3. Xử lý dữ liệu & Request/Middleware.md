Cảm ơn bạn đã cung cấp outline chi tiết! Dưới đây tôi sẽ đi sâu vào **từng mục một** với code mẫu hoàn chỉnh và giải thích cặn kẽ để bạn hiểu rõ và áp dụng được ngay.

---

## 1. Middleware là gì? (Chi tiết hơn)

Middleware trong NestJS **chính là Express Middleware** được "bọc" lại để tương thích với hệ thống Dependency Injection của NestJS.

**Cấu trúc cốt lõi:**
```typescript
(req: Request, res: Response, next: NextFunction) => void
```

**Ví dụ minh họa trực quan:**
```typescript
// Middleware chặn request từ IP bị cấm
import { Injectable, NestMiddleware, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class IpBlockerMiddleware implements NestMiddleware {
  private readonly blockedIps = ['192.168.1.100', '10.0.0.5'];
  
  use(req: Request, res: Response, next: NextFunction) {
    const clientIp = req.ip || req.connection.remoteAddress;
    
    if (this.blockedIps.includes(clientIp)) {
      // Kết thúc request ngay tại đây, không cho đi tiếp
      throw new HttpException('IP bị chặn', HttpStatus.FORBIDDEN);
    }
    
    // Cho phép request đi tiếp
    next();
  }
}
```

---

## 2. Flow request trong NestJS (Minh họa bằng code thực tế)

Để thấy rõ thứ tự, tôi sẽ tạo một ví dụ có **log ở từng layer**:

```typescript
// 1. MIDDLEWARE
@Injectable()
export class TimingMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log('🔵 [1] Middleware - Bắt đầu');
    req['startTime'] = Date.now();
    next();
    console.log('🔵 [7] Middleware - Kết thúc (sau khi response gửi đi)');
  }
}

// 2. GUARD
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    console.log('🟢 [2] Guard - Kiểm tra quyền truy cập');
    return true; // Cho phép đi tiếp
  }
}

// 3. INTERCEPTOR
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    console.log('🟡 [3] Interceptor - Trước khi vào controller');
    
    return next.handle().pipe(
      tap(() => console.log('🟡 [6] Interceptor - Sau khi có response'))
    );
  }
}

// 4. PIPE
@Injectable()
export class ValidationPipe implements PipeTransform {
  transform(value: any, metadata: ArgumentMetadata) {
    console.log('🟠 [4] Pipe - Validate và transform dữ liệu');
    return value;
  }
}

// 5. CONTROLLER
@Controller('users')
@UseGuards(AuthGuard)
@UseInterceptors(LoggingInterceptor)
export class UsersController {
  @Get(':id')
  getUser(@Param('id', ValidationPipe) id: string) {
    console.log('🔴 [5] Controller - Xử lý logic chính');
    return { id, name: 'John Doe' };
  }
}
```

**Kết quả console khi gọi GET /users/123:**
```
🔵 [1] Middleware - Bắt đầu
🟢 [2] Guard - Kiểm tra quyền truy cập
🟡 [3] Interceptor - Trước khi vào controller
🟠 [4] Pipe - Validate và transform dữ liệu
🔴 [5] Controller - Xử lý logic chính
🟡 [6] Interceptor - Sau khi có response
🔵 [7] Middleware - Kết thúc (sau khi response gửi đi)
```

---

## 3. Cách tạo Middleware (Chi tiết 2 cách)

### Cách 1: Class-based Middleware (Có DI)

**File: `middlewares/audit.middleware.ts`**
```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { AuditService } from '../audit/audit.service';

@Injectable()
export class AuditMiddleware implements NestMiddleware {
  // Có thể inject service vào đây
  constructor(private readonly auditService: AuditService) {}
  
  async use(req: Request, res: Response, next: NextFunction) {
    const auditData = {
      method: req.method,
      url: req.url,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      timestamp: new Date()
    };
    
    // Lưu log vào database (bất đồng bộ, không block request)
    this.auditService.logAccess(auditData).catch(err => {
      console.error('Lỗi ghi audit log:', err);
    });
    
    next();
  }
}
```

### Cách 2: Functional Middleware

**File: `middlewares/simple-logger.middleware.ts`**
```typescript
import { Request, Response, NextFunction } from 'express';

export const simpleLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  // Ghi log khi response kết thúc
  res.on('finish', () => {
    const duration = Date.now() - start;
    console.log(`${req.method} ${req.url} - ${res.statusCode} - ${duration}ms`);
  });
  
  next();
};
```

---

## 4 & 5. Áp dụng Middleware (Đăng ký chi tiết)

**File: `app.module.ts`**
```typescript
import { Module, NestModule, MiddlewareConsumer, RequestMethod } from '@nestjs/common';
import { AuditMiddleware } from './middlewares/audit.middleware';
import { simpleLogger } from './middlewares/simple-logger.middleware';
import { UsersModule } from './users/users.module';
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [UsersModule, AuthModule],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      // Áp dụng nhiều middleware (chạy theo thứ tự khai báo)
      .apply(simpleLogger, AuditMiddleware)
      
      // Áp dụng cho tất cả routes TRỪ auth
      .exclude(
        { path: 'auth/login', method: RequestMethod.POST },
        { path: 'auth/register', method: RequestMethod.POST },
        { path: 'health', method: RequestMethod.GET }
      )
      .forRoutes('*');
      
    // Áp dụng middleware riêng cho Users module
    consumer
      .apply(AuditMiddleware)
      .forRoutes('users');
      
    // Áp dụng cho controller cụ thể và method cụ thể
    consumer
      .apply(simpleLogger)
      .forRoutes(
        { path: 'users', method: RequestMethod.GET },
        { path: 'users/:id', method: RequestMethod.PUT }
      );
  }
}
```

---

## 6. Exclude route (Ví dụ thực tế)

```typescript
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(AuthMiddleware)
      .exclude(
        // Public routes không cần auth
        'auth/login',
        'auth/register',
        'auth/forgot-password',
        // Health check endpoint
        { path: 'health', method: RequestMethod.GET },
        // API docs
        'api-docs(.*)',
      )
      .forRoutes('*');
  }
}
```

---

## 7. Middleware vs Guard vs Interceptor (So sánh thực tế)

| Tiêu chí | Middleware | Guard | Interceptor |
|----------|------------|-------|-------------|
| **Thời điểm chạy** | Đầu tiên | Sau Middleware | Sau Guard |
| **Biết handler nào?** | ❌ Không | ✅ Có | ✅ Có |
| **Có thể dùng DI?** | ✅ (class) | ✅ | ✅ |
| **Dùng decorator?** | ❌ | ✅ `@UseGuards()` | ✅ `@UseInterceptors()` |
| **Chặn request?** | ✅ | ✅ | ✅ |
| **Biến đổi response?** | ✅ (khó) | ❌ | ✅ (dễ) |

**Ví dụ minh họa sự khác biệt:**

```typescript
// MIDDLEWARE: Không biết route có @Public() hay không
@Injectable()
export class BadAuthMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    // Vấn đề: Làm sao biết route này public hay private?
    // Không có cách nào để đọc @Public() decorator ở đây!
    const token = req.headers.authorization;
    if (!token) {
      // Sẽ chặn cả public routes!
      return res.status(401).send('Unauthorized');
    }
    next();
  }
}

// GUARD: Biết rõ metadata của route
@Injectable()
export class GoodAuthGuard implements CanActivate {
  constructor(private reflector: Reflector) {}
  
  canActivate(context: ExecutionContext): boolean {
    // Có thể đọc metadata từ decorator
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (isPublic) {
      return true; // Cho phép public routes
    }
    
    // Kiểm tra token cho private routes
    const request = context.switchToHttp().getRequest();
    const token = request.headers.authorization;
    return !!token;
  }
}
```

---

## 8. Middleware cho Authentication (Tại sao không nên)

**❌ Ví dụ SAI (dùng Middleware cho auth):**
```typescript
@Injectable()
export class WrongAuthMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const publicRoutes = ['/auth/login', '/auth/register', '/public'];
    
    if (publicRoutes.includes(req.path)) {
      return next();
    }
    
    const token = req.headers.authorization;
    if (!token) {
      return res.status(401).json({ message: 'Unauthorized' });
    }
    
    // Vấn đề:
    // 1. Phải maintain danh sách public routes thủ công
    // 2. Không thể dùng decorator @Public() linh hoạt
    // 3. Khó mở rộng cho role-based access control
    // 4. Không tận dụng được NestJS metadata system
    
    try {
      const decoded = jwt.verify(token, process.env.JWT_SECRET);
      req['user'] = decoded;
      next();
    } catch (error) {
      return res.status(401).json({ message: 'Invalid token' });
    }
  }
}
```

**✅ Ví dụ ĐÚNG (dùng Guard):**
```typescript
// decorators/public.decorator.ts
import { SetMetadata } from '@nestjs/common';
export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);

// guards/auth.guard.ts
@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private jwtService: JwtService,
    private reflector: Reflector
  ) {}
  
  canActivate(context: ExecutionContext): boolean {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);
    
    if (isPublic) return true;
    
    const request = context.switchToHttp().getRequest();
    const token = this.extractTokenFromHeader(request);
    
    if (!token) {
      throw new UnauthorizedException();
    }
    
    try {
      const payload = this.jwtService.verify(token);
      request['user'] = payload;
      return true;
    } catch {
      throw new UnauthorizedException();
    }
  }
}

// Sử dụng trong controller
@Controller('users')
export class UsersController {
  @Public() // Route này không cần auth
  @Get()
  findAll() {
    return this.usersService.findAll();
  }
  
  @Get('profile') // Route này yêu cầu auth
  getProfile(@Request() req) {
    return req.user;
  }
}
```

---

## 9 & 10. Global Middleware & Async Middleware

**Global Middleware (đăng ký trong main.ts):**
```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as helmet from 'helmet';
import * as compression from 'compression';
import * as cors from 'cors';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global middlewares - Chạy cho mọi request
  app.use(helmet()); // Security headers
  app.use(cors({ origin: 'https://example.com' })); // CORS
  app.use(compression()); // Gzip compression
  
  // Custom global middleware
  app.use((req: Request, res: Response, next: NextFunction) => {
    req['requestId'] = crypto.randomUUID();
    next();
  });
  
  await app.listen(3000);
}
bootstrap();
```

**Async Middleware (xử lý bất đồng bộ):**
```typescript
@Injectable()
export class GeoLocationMiddleware implements NestMiddleware {
  constructor(private readonly geoService: GeoLocationService) {}
  
  async use(req: Request, res: Response, next: NextFunction) {
    const clientIp = req.ip;
    
    try {
      // Gọi API bên ngoài để lấy vị trí địa lý
      const location = await this.geoService.getLocationByIp(clientIp);
      req['geoLocation'] = location;
      
      // Chặn request từ quốc gia bị cấm
      if (location.country === 'North Korea') {
        return res.status(403).json({ 
          message: 'Access denied from your location' 
        });
      }
      
      next();
    } catch (error) {
      // Nếu không lấy được location, vẫn cho đi tiếp
      console.error('Geo lookup failed:', error);
      next();
    }
  }
}
```

---

## 11. Dependency Injection trong Middleware (Chi tiết)

**File: `middlewares/user-context.middleware.ts`**
```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { UserService } from '../user/user.service';
import { TokenService } from '../auth/token.service';

@Injectable()
export class UserContextMiddleware implements NestMiddleware {
  constructor(
    private readonly userService: UserService,  // Inject UserService
    private readonly tokenService: TokenService, // Inject TokenService
  ) {}
  
  async use(req: Request, res: Response, next: NextFunction) {
    const token = req.headers.authorization?.replace('Bearer ', '');
    
    if (token) {
      try {
        const decoded = await this.tokenService.verifyToken(token);
        const user = await this.userService.findById(decoded.sub);
        req['currentUser'] = user; // Gán user vào request
      } catch (error) {
        // Token không hợp lệ, bỏ qua
      }
    }
    
    next();
  }
}

// Đăng ký trong module
@Module({
  imports: [UserModule, AuthModule],
  providers: [UserContextMiddleware], // Phải thêm vào providers
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(UserContextMiddleware).forRoutes('*');
  }
}
```

---

## 13. Các use case thực tế (Code hoàn chỉnh)

### Use Case 1: Request ID Tracing
```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const requestId = req.headers['x-request-id'] || uuidv4();
    req['requestId'] = requestId;
    res.setHeader('X-Request-Id', requestId);
    next();
  }
}
```

### Use Case 2: Rate Limiting
```typescript
import { Injectable, NestMiddleware, HttpException, HttpStatus } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class RateLimitMiddleware implements NestMiddleware {
  private requests = new Map<string, { count: number; resetTime: number }>();
  
  use(req: Request, res: Response, next: NextFunction) {
    const ip = req.ip;
    const now = Date.now();
    const windowMs = 60 * 1000; // 1 phút
    const maxRequests = 100; // 100 requests/phút
    
    const record = this.requests.get(ip) || { count: 0, resetTime: now + windowMs };
    
    if (now > record.resetTime) {
      record.count = 1;
      record.resetTime = now + windowMs;
    } else {
      record.count++;
    }
    
    this.requests.set(ip, record);
    
    res.setHeader('X-RateLimit-Limit', maxRequests);
    res.setHeader('X-RateLimit-Remaining', Math.max(0, maxRequests - record.count));
    
    if (record.count > maxRequests) {
      throw new HttpException('Too Many Requests', HttpStatus.TOO_MANY_REQUESTS);
    }
    
    next();
  }
}
```

### Use Case 3: Response Time Tracking
```typescript
@Injectable()
export class ResponseTimeMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const start = process.hrtime();
    
    res.on('finish', () => {
      const diff = process.hrtime(start);
      const time = (diff[0] * 1e3 + diff[1] * 1e-6).toFixed(2); // ms
      
      console.log(`${req.method} ${req.url} took ${time}ms`);
      
      // Gửi metrics đến monitoring system nếu response chậm
      if (parseFloat(time) > 1000) {
        this.sendSlowRequestAlert(req, time);
      }
    });
    
    next();
  }
  
  private sendSlowRequestAlert(req: Request, time: string) {
    // Gửi alert đến Slack/Email
    console.warn(`⚠️ Slow request detected: ${req.method} ${req.url} - ${time}ms`);
  }
}
```

---

## 14. Common Mistakes (Ví dụ minh họa)

### ❌ Mistake 1: Quên next()
```typescript
// SAI - Request sẽ bị treo mãi mãi
use(req: Request, res: Response, next: NextFunction) {
  console.log('Logging...');
  // QUÊN gọi next() -> Request timeout!
}

// ĐÚNG
use(req: Request, res: Response, next: NextFunction) {
  console.log('Logging...');
  next(); // Luôn gọi next() trừ khi muốn kết thúc request
}
```

### ❌ Mistake 2: Xử lý logic nặng đồng bộ
```typescript
// SAI - Block event loop
use(req: Request, res: Response, next: NextFunction) {
  const data = fs.readFileSync('large-file.csv'); // Đồng bộ, block tất cả requests khác!
  req['fileData'] = data;
  next();
}

// ĐÚNG
async use(req: Request, res: Response, next: NextFunction) {
  try {
    const data = await fs.promises.readFile('large-file.csv');
    req['fileData'] = data;
    next();
  } catch (error) {
    next(error); // Truyền lỗi cho error handler
  }
}
```

### ❌ Mistake 3: Không xử lý lỗi trong async middleware
```typescript
// SAI - Lỗi không được bắt, app crash
async use(req: Request, res: Response, next: NextFunction) {
  const user = await this.userService.findById(req.params.id); // Có thể throw exception!
  req['user'] = user;
  next();
}

// ĐÚNG
async use(req: Request, res: Response, next: NextFunction) {
  try {
    const user = await this.userService.findById(req.params.id);
    req['user'] = user;
    next();
  } catch (error) {
    next(error); // Chuyển lỗi cho NestJS error handler
  }
}
```

---

## Tổng kết Flow hoàn chỉnh với tất cả các layer

```typescript
// 1. Global Middleware (main.ts)
app.use(helmet());
app.use(cors());

// 2. Module Middleware (app.module.ts)
consumer.apply(RequestIdMiddleware, AuditMiddleware).forRoutes('*');

// 3. Guard
@UseGuards(JwtAuthGuard, RolesGuard)

// 4. Interceptor (Pre)
@UseInterceptors(LoggingInterceptor, CacheInterceptor)

// 5. Pipe
@UsePipes(ValidationPipe)

// 6. Controller
@Controller('users')
export class UsersController {
  @Get()
  findAll() {
    return this.usersService.findAll();
  }
}

// 7. Interceptor (Post)
// Tự động chạy sau controller

// 8. Exception Filter (nếu có lỗi)
@UseFilters(HttpExceptionFilter)
```

Hy vọng với phần giải thích chi tiết kèm code mẫu này, bạn đã nắm vững toàn bộ về Middleware trong NestJS! Nếu cần làm rõ thêm phần nào, cứ hỏi tôi nhé.