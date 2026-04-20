

## 📚 Giới thiệu về Interceptors

Interceptor trong NestJS là một thành phần đặc biệt được **inspire từ Spring AOP (Aspect Oriented Programming)**. Interceptor cho phép bạn can thiệp vào luồng xử lý request/response.

### Đặc điểm chính:
- Được kích hoạt **trước và sau** khi method handler được gọi
- Có thể **biến đổi kết quả** trả về
- Có thể **mở rộng behavior** cơ bản
- Có thể **xử lý lỗi** tập trung
- Có thể **log** hoạt động

## 🎯 Các use cases phổ biến

1. **Logging** - Ghi log request/response
2. **Transform data** - Chuẩn hóa response format
3. **Cache** - Cache response
4. **Timing** - Đo thời gian xử lý
5. **Error handling** - Xử lý lỗi tập trung
6. **Rate limiting** - Giới hạn request

## 🏗 Cấu trúc cơ bản

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';

@Injectable()
export class MyInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // 1. Xử lý TRƯỚC khi gọi handler
    console.log('Before handler...');
    
    const now = Date.now();
    
    return next
      .handle() // Gọi handler
      .pipe(
        // 2. Xử lý SAU khi handler trả về
        tap(() => console.log(`After handler: ${Date.now() - now}ms`)),
        map(data => {
          // 3. Biến đổi response data
          return {
            success: true,
            data: data,
            timestamp: new Date().toISOString()
          };
        })
      );
  }
}
```

## 📦 Các loại Interceptors phổ biến

### 1. **Logging Interceptor** - Ghi log request/response

```typescript
// interceptors/logging.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Logger,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';
import { Request, Response } from 'express';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  private readonly logger = new Logger(LoggingInterceptor.name);

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const ctx = context.switchToHttp();
    const request = ctx.getRequest<Request>();
    const response = ctx.getResponse<Response>();
    
    const { method, url, body, query, params, ip } = request;
    const userAgent = request.get('user-agent') || '';
    const startTime = Date.now();

    this.logger.log(`📥 ${method} ${url} - Started`);
    this.logger.debug(`Body: ${JSON.stringify(body)}`);
    this.logger.debug(`Query: ${JSON.stringify(query)}`);
    this.logger.debug(`Params: ${JSON.stringify(params)}`);

    return next.handle().pipe(
      tap({
        next: (data) => {
          const duration = Date.now() - startTime;
          this.logger.log(
            `📤 ${method} ${url} - ${response.statusCode} - ${duration}ms - ${ip} - ${userAgent}`
          );
          this.logger.debug(`Response: ${JSON.stringify(data)}`);
        },
        error: (error) => {
          const duration = Date.now() - startTime;
          this.logger.error(
            `❌ ${method} ${url} - ${error.status} - ${duration}ms - ${ip} - ${userAgent}`
          );
          this.logger.error(error.message);
        },
      })
    );
  }
}
```

### 2. **Transform Interceptor** - Chuẩn hóa response format

```typescript
// interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  statusCode: number;
  success: boolean;
  message: string;
  data: T;
  timestamp: string;
  path: string;
}

@Injectable()
export class TransformInterceptor<T> implements NestInterceptor<T, Response<T>> {
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    const ctx = context.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse();

    return next.handle().pipe(
      map(data => ({
        statusCode: response.statusCode,
        success: true,
        message: data?.message || 'Success',
        data: data?.result || data,
        timestamp: new Date().toISOString(),
        path: request.url,
      }))
    );
  }
}
```

### 3. **Timeout Interceptor** - Xử lý timeout

```typescript
// interceptors/timeout.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  RequestTimeoutException,
} from '@nestjs/common';
import { Observable, throwError, TimeoutError } from 'rxjs';
import { catchError, timeout } from 'rxjs/operators';

@Injectable()
export class TimeoutInterceptor implements NestInterceptor {
  constructor(private readonly timeoutMs: number = 30000) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      timeout(this.timeoutMs),
      catchError(err => {
        if (err instanceof TimeoutError) {
          return throwError(() => new RequestTimeoutException('Request timeout'));
        }
        return throwError(() => err);
      })
    );
  }
}
```

### 4. **Cache Interceptor** - Cache response

```typescript
// interceptors/cache.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  Inject,
} from '@nestjs/common';
import { Observable, of } from 'rxjs';
import { tap } from 'rxjs/operators';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';

@Injectable()
export class CacheInterceptor implements NestInterceptor {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const cacheKey = `${request.method}:${request.url}`;

    // Try to get cached data
    const cachedData = await this.cacheManager.get(cacheKey);
    
    if (cachedData) {
      console.log(`🔄 Cache hit for ${cacheKey}`);
      return of(cachedData);
    }

    console.log(`💾 Cache miss for ${cacheKey}`);

    return next.handle().pipe(
      tap(async (data) => {
        // Cache the response for 5 minutes
        await this.cacheManager.set(cacheKey, data, 300000);
      })
    );
  }
}
```

### 5. **Errors Interceptor** - Xử lý lỗi tập trung

```typescript
// interceptors/errors.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  BadRequestException,
  UnauthorizedException,
  NotFoundException,
  ConflictException,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Observable, throwError } from 'rxjs';
import { catchError } from 'rxjs/operators';
import { QueryFailedError } from 'typeorm';

@Injectable()
export class ErrorsInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      catchError(error => {
        console.error('Error caught in interceptor:', error);

        // Handle specific database errors
        if (error instanceof QueryFailedError) {
          if (error.message.includes('duplicate key')) {
            return throwError(() => new ConflictException('Duplicate entry'));
          }
        }

        // Handle validation errors
        if (error.name === 'ValidationError') {
          return throwError(() => new BadRequestException(error.message));
        }

        // Handle TypeORM errors
        if (error.code === 'ER_NO_REFERENCED_ROW_2') {
          return throwError(() => new NotFoundException('Related record not found'));
        }

        // If it's already an HttpException, just rethrow
        if (error instanceof HttpException) {
          return throwError(() => error);
        }

        // Default error
        return throwError(() => new HttpException(
          {
            status: HttpStatus.INTERNAL_SERVER_ERROR,
            error: 'Internal server error',
            message: error.message || 'Something went wrong',
          },
          HttpStatus.INTERNAL_SERVER_ERROR,
        ));
      })
    );
  }
}
```

### 6. **Rate Limit Interceptor** - Giới hạn request

```typescript
// interceptors/rate-limit.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class RateLimitInterceptor implements NestInterceptor {
  private requests = new Map<string, { count: number; timestamp: number }>();
  private readonly limit = 100; // Số request tối đa
  private readonly windowMs = 60000; // 1 phút

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const key = request.ip || request.headers['x-forwarded-for'] || 'unknown';
    const now = Date.now();

    const record = this.requests.get(key);
    
    if (record) {
      // Reset if window expired
      if (now - record.timestamp > this.windowMs) {
        this.requests.set(key, { count: 1, timestamp: now });
        return next.handle();
      }
      
      // Check limit
      if (record.count >= this.limit) {
        throw new HttpException(
          'Too many requests',
          HttpStatus.TOO_MANY_REQUESTS
        );
      }
      
      // Increment count
      record.count++;
      this.requests.set(key, record);
    } else {
      this.requests.set(key, { count: 1, timestamp: now });
    }

    return next.handle();
  }
}
```

### 7. **File Upload Interceptor** - Xử lý upload file

```typescript
// interceptors/file-upload.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
  BadRequestException,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class FileUploadInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    
    // Validate file before processing
    if (request.file) {
      const maxSize = 5 * 1024 * 1024; // 5MB
      const allowedTypes = ['image/jpeg', 'image/png', 'image/jpg'];
      
      if (request.file.size > maxSize) {
        throw new BadRequestException('File size exceeds 5MB');
      }
      
      if (!allowedTypes.includes(request.file.mimetype)) {
        throw new BadRequestException('Invalid file type. Only JPEG, PNG allowed');
      }
    }

    return next.handle().pipe(
      map(data => ({
        ...data,
        uploadedFile: request.file ? {
          filename: request.file.originalname,
          size: request.file.size,
          mimetype: request.file.mimetype,
          path: request.file.path,
        } : null,
      }))
    );
  }
}
```

## 🔧 Cách sử dụng Interceptors

### 1. **Global Level** - Áp dụng cho toàn bộ ứng dụng

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { TransformInterceptor } from './interceptors/transform.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // Global interceptors
  app.useGlobalInterceptors(
    new LoggingInterceptor(),
    new TransformInterceptor(),
  );
  
  await app.listen(3000);
}
bootstrap();
```

### 2. **Controller Level** - Áp dụng cho controller

```typescript
// auth.controller.ts
import { Controller, UseInterceptors } from '@nestjs/common';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { CacheInterceptor } from './interceptors/cache.interceptor';

@Controller('auth')
@UseInterceptors(LoggingInterceptor, CacheInterceptor) // Áp dụng cho tất cả methods
export class AuthController {
  // ...
}
```

### 3. **Method Level** - Áp dụng cho method cụ thể

```typescript
// auth.controller.ts
import { Controller, Post, UseInterceptors } from '@nestjs/common';
import { TimeoutInterceptor } from './interceptors/timeout.interceptor';

@Controller('auth')
export class AuthController {
  
  @Post('login')
  @UseInterceptors(new TimeoutInterceptor(5000)) // 5 seconds timeout
  async login() {
    // ...
  }
}
```

### 4. **Custom Provider Level** - DI Container

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { APP_INTERCEPTOR } from '@nestjs/core';
import { LoggingInterceptor } from './interceptors/logging.interceptor';
import { TransformInterceptor } from './interceptors/transform.interceptor';

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: LoggingInterceptor,
    },
    {
      provide: APP_INTERCEPTOR,
      useClass: TransformInterceptor,
    },
  ],
})
export class AppModule {}
```

## 🎨 Ví dụ thực tế - Kết hợp nhiều Interceptors

```typescript
// interceptors/combined.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map, tap } from 'rxjs/operators';

@Injectable()
export class CombinedInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    const ctx = context.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse();

    // Log request
    console.log(`[${request.method}] ${request.url} - Started`);

    return next.handle().pipe(
      // Log response time
      tap(() => {
        const duration = Date.now() - start;
        console.log(`[${request.method}] ${request.url} - ${duration}ms`);
      }),
      
      // Transform response
      map(data => ({
        success: true,
        data: data,
        timestamp: new Date().toISOString(),
        requestId: request.id,
        duration: `${Date.now() - start}ms`,
      })),
      
      // Set custom headers
      tap(() => {
        response.setHeader('X-Response-Time', `${Date.now() - start}ms`);
        response.setHeader('X-Powered-By', 'NestJS');
      })
    );
  }
}
```

## 📝 Interceptor với Dependency Injection

```typescript
// interceptors/configurable.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class ConfigurableInterceptor implements NestInterceptor {
  constructor(
    private configService: ConfigService,
    private options?: { excludePaths?: string[] }
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    
    // Skip interceptor for certain paths
    if (this.options?.excludePaths?.includes(request.url)) {
      return next.handle();
    }

    const environment = this.configService.get('NODE_ENV');
    
    return next.handle().pipe(
      map(data => ({
        data,
        environment,
        version: this.configService.get('APP_VERSION'),
      }))
    );
  }
}
```

## ✅ Best Practices

1. **Order matters** - Interceptors chạy theo thứ tự từ ngoài vào trong
2. **Don't forget to call next.handle()** - Luôn gọi để tiếp tục xử lý
3. **Use dependency injection** - Tận dụng DI của NestJS
4. **Keep interceptors focused** - Mỗi interceptor chỉ làm một việc
5. **Handle errors properly** - Luôn có error handling
6. **Use async/await** - Hỗ trợ async operations

```typescript
// Ví dụ về order của interceptors
app.useGlobalInterceptors(
  LoggingInterceptor,    // Chạy đầu tiên
  TransformInterceptor,  // Chạy thứ hai
  TimeoutInterceptor,    // Chạy cuối cùng
);
```