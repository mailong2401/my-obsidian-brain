
---

# 🎮 Controller trong NestJS - Giải thích chi tiết
## 1. Controller là gì?

**Controller** có nhiệm vụ:
- **Nhận request** từ client (POST, GET, PUT, DELETE...)
- **Xử lý** (hoặc giao việc cho Service)
- **Trả về response** cho client

> Hãy tưởng tượng Controller là **người phục vụ** trong nhà hàng: nhận order (request) → gửi xuống bếp (Service) → mang món ăn (response) ra cho khách.

---

## 2. Các Decorator cơ bản

### 📌 `@Controller()`
Đánh dấu một class là Controller, đồng thời định nghĩa **route prefix**.

```typescript
// users.controller.ts
import { Controller, Get } from '@nestjs/common';

@Controller('users')  // Tất cả routes trong controller này sẽ bắt đầu bằng /users
export class UsersController {
  @Get()
  findAll() {
    return 'This will be called khi GET /users';
  }
}
```

### 📌 `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`
Xác định HTTP method và sub-route.

```typescript
@Controller('products')
export class ProductsController {
  @Get()              // GET /products
  getAll() {}

  @Get('featured')    // GET /products/featured
  getFeatured() {}

  @Post()             // POST /products
  create() {}

  @Put(':id')         // PUT /products/123
  update() {}

  @Delete(':id')      // DELETE /products/123
  remove() {}
}
```

---

## 3. Lấy dữ liệu từ Request

### 🎯 `@Param()` - Lấy tham số từ URL

```typescript
@Controller('users')
export class UsersController {
  // URL: GET /users/123
  @Get(':id')
  findOne(@Param('id') id: string) {
    return `User ID: ${id}`;
  }

  // Lấy nhiều param: GET /users/123/posts/456
  @Get(':userId/posts/:postId')
  findPost(
    @Param('userId') userId: string,
    @Param('postId') postId: string
  ) {
    return { userId, postId };
  }

  // Lấy tất cả params (dùng khi không biết trước key)
  @Get('advanced/:id/:slug')
  getAllParams(@Param() params: any) {
    return params; // { id: '123', slug: 'hello-world' }
  }
}
```

### 🎯 `@Query()` - Lấy query string

```typescript
@Controller('products')
export class ProductsController {
  // URL: GET /products?page=2&limit=10&category=phone
  @Get()
  findAll(@Query() query: any) {
    return query; // { page: '2', limit: '10', category: 'phone' }
  }

  // Lấy từng query cụ thể
  @Get('search')
  search(
    @Query('keyword') keyword: string,
    @Query('page') page: number,
    @Query('sort') sort: string = 'desc' // có default value
  ) {
    return { keyword, page, sort };
  }
}
```

### 🎯 `@Body()` - Lấy dữ liệu từ request body

```typescript
// Định nghĩa DTO (Data Transfer Object)
class CreateUserDto {
  name: string;
  email: string;
  age: number;
}

@Controller('users')
export class UsersController {
  // POST /users
  // Body: { "name": "Teo", "email": "teo@example.com", "age": 25 }
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return `Created user: ${createUserDto.name}`;
  }

  // Lấy một field cụ thể trong body
  @Post('check-email')
  checkEmail(@Body('email') email: string) {
    return `Checking email: ${email}`;
  }
}
```

### 🎯 `@Headers()` - Lấy headers

```typescript
@Controller('auth')
export class AuthController {
  @Get('token')
  getToken(@Headers('authorization') authHeader: string) {
    return { token: authHeader };
  }

  @Get('info')
  getAllHeaders(@Headers() headers: any) {
    return headers;
  }
}
```

### 🎯 `@Req()` / `@Res()` - Request/Response object gốc của Express

```typescript
import { Request, Response } from 'express';

@Controller()
export class AppController {
  @Get('express')
  expressWay(@Req() req: Request, @Res() res: Response) {
    // Cách của Express (không khuyến khích trong NestJS)
    res.status(200).json({ message: 'Hello' });
  }

  // Cách NestJS (khuyến khích)
  @Get('nestjs-way')
  nestjsWay() {
    return { message: 'Hello' }; // Nest tự động trả về JSON
  }
}
```

> ⚠️ **Lưu ý**: Tránh dùng `@Res()` trừ khi thực sự cần (ví dụ: stream file, download). Nest sẽ mất khả năng tự động xử lý response.

---

## 4. Ví dụ thực tế hoàn chỉnh

```typescript
// users.controller.ts
import { Controller, Get, Post, Body, Param, Query, Put, Delete, HttpStatus, HttpException } from '@nestjs/common';

// DTOs
class CreateUserDto {
  name: string;
  email: string;
  age: number;
}

class UpdateUserDto {
  name?: string;
  email?: string;
  age?: number;
}

@Controller('users')
export class UsersController {
  // Giả lập database
  private users = [
    { id: 1, name: 'Teo', email: 'teo@example.com', age: 25 },
    { id: 2, name: 'Ty', email: 'ty@example.com', age: 30 },
  ];

  // GET /users - Lấy tất cả users (có filter qua query)
  @Get()
  findAll(@Query('minAge') minAge?: number) {
    if (minAge) {
      return this.users.filter(user => user.age >= minAge);
    }
    return this.users;
  }

  // GET /users/1 - Lấy user theo ID
  @Get(':id')
  findOne(@Param('id') id: string) {
    const user = this.users.find(u => u.id === parseInt(id));
    if (!user) {
      throw new HttpException('User not found', HttpStatus.NOT_FOUND);
    }
    return user;
  }

  // POST /users - Tạo user mới
  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    const newUser = {
      id: this.users.length + 1,
      ...createUserDto,
    };
    this.users.push(newUser);
    return {
      statusCode: HttpStatus.CREATED,
      message: 'User created successfully',
      data: newUser
    };
  }

  // PUT /users/1 - Cập nhật user
  @Put(':id')
  update(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto
  ) {
    const index = this.users.findIndex(u => u.id === parseInt(id));
    if (index === -1) {
      throw new HttpException('User not found', HttpStatus.NOT_FOUND);
    }
    
    this.users[index] = { ...this.users[index], ...updateUserDto };
    return this.users[index];
  }

  // DELETE /users/1 - Xóa user
  @Delete(':id')
  remove(@Param('id') id: string) {
    const index = this.users.findIndex(u => u.id === parseInt(id));
    if (index === -1) {
      throw new HttpException('User not found', HttpStatus.NOT_FOUND);
    }
    
    const deletedUser = this.users[index];
    this.users.splice(index, 1);
    return {
      message: 'User deleted successfully',
      deletedUser
    };
  }
}
```

---

## 5. Status Code và Response

### Tự động (mặc định)
- `POST` → **201 Created**
- Các method khác → **200 OK**
- Không có dữ liệu → **204 No Content**

### Tùy chỉnh với `@HttpCode()`

```typescript
import { HttpCode, HttpStatus } from '@nestjs/common';

@Controller('payments')
export class PaymentsController {
  @Post()
  @HttpCode(202) // Accepted
  processPayment() {
    return { message: 'Processing...' };
  }

  @Delete('hard')
  @HttpCode(HttpStatus.NO_CONTENT)
  hardDelete() {
    // Không cần return gì cả
  }
}
```

---

## 6. Async / Promise (Quan trọng)

```typescript
@Controller('data')
export class DataController {
  @Get('async')
  async getAsyncData() {
    // Giả sử gọi database hoặc API
    const data = await this.fetchFromDatabase();
    return data; // Nest tự động trả về Promise
  }

  @Get('promise')
  getPromiseData() {
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve({ message: 'Promise resolved' });
      }, 2000);
    });
  }
}
```

---

## 7. Route wildcard và optional param

```typescript
@Controller('files')
export class FilesController {
  // Wildcard: GET /files/anything/you/want
  @Get('*')
  catchAll() {
    return 'This matches any route under /files';
  }

  // Optional param: GET /posts hoặc GET /posts/featured
  @Get('posts/:type?')
  getPosts(@Param('type') type?: string) {
    if (type) {
      return `Getting ${type} posts`;
    }
    return 'Getting all posts';
  }
}
```

---

## 8. Tổng kết nhanh

| Decorator | Lấy dữ liệu từ | Ví dụ URL |
|-----------|---------------|-----------|
| `@Param('id')` | URL parameter | `/users/123` → id = 123 |
| `@Query('page')` | Query string | `/users?page=2` → page = 2 |
| `@Body()` | Request body | `POST /users` với JSON body |
| `@Headers()` | HTTP Headers | Lấy token từ header |
| `@Req()` | Full Express Request object | Dùng khi cần low-level control |
