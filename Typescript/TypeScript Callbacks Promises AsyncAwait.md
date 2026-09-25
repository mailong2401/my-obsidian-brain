

> [!abstract] Mục tiêu  
> Hiểu cách JavaScript/TypeScript xử lý tác vụ bất đồng bộ (**asynchronous**) và quá trình phát triển:
> 
> **Callback → Promise → Async/Await**

---

## 1. Synchronous và Asynchronous

### Synchronous — Đồng bộ

Code chạy **lần lượt từ trên xuống dưới**.

```ts
console.log("1");
console.log("2");
console.log("3");
```

Kết quả:

```text
1
2
3
```

Mỗi công việc hoàn thành rồi chương trình mới tiếp tục công việc tiếp theo.

---

### Asynchronous — Bất đồng bộ

Một số công việc cần thời gian:

- Gọi API
    
- Query database
    
- Đọc file
    
- Upload file
    
- Timer
    
- Gửi email
    
- Truy cập network
    

JavaScript không nhất thiết phải đứng chờ chúng hoàn thành.

```ts
console.log("1");

setTimeout(() => {
  console.log("2");
}, 2000);

console.log("3");
```

Kết quả:

```text
1
3
2
```

`setTimeout()` đăng ký công việc cần thực hiện sau. Chương trình tiếp tục chạy `console.log("3")`.

> [!important]  
> `async` không đơn giản có nghĩa là "chạy song song".
> 
> Để hiểu sâu hơn, xem [[JavaScript Event Loop]].

---

# 2. Callback

## Callback là gì?

**Callback** là một function được truyền vào function khác để function đó gọi lại khi cần.

Ví dụ đơn giản:

```ts
function calculate(
  a: number,
  b: number,
  callback: (result: number) => void
) {
  const result = a + b;
  callback(result);
}

calculate(10, 20, (result) => {
  console.log(result);
});
```

Kết quả:

```text
30
```

Ở đây:

```ts
(result) => {
  console.log(result);
}
```

chính là **callback function**.

---

## Callback trong xử lý bất đồng bộ

```ts
function getUser(
  id: number,
  callback: (user: string) => void
) {
  setTimeout(() => {
    callback("Long");
  }, 1000);
}

getUser(1, (user) => {
  console.log(user);
});
```

Luồng hoạt động:

```text
getUser()
   │
   ├── bắt đầu công việc async
   │
   └── sau 1 giây
          │
          ▼
     callback("Long")
          │
          ▼
     console.log("Long")
```

Callback giúp chúng ta nói:

> "Khi công việc này xong thì hãy chạy function này."

---

# 3. Error-first Callback

Một pattern từng rất phổ biến trong Node.js:

```ts
callback(error, result);
```

Ví dụ:

```ts
function getUser(
  id: number,
  callback: (error: Error | null, user?: string) => void
) {
  setTimeout(() => {
    if (id <= 0) {
      callback(new Error("Invalid user ID"));
      return;
    }

    callback(null, "Long");
  }, 1000);
}
```

Sử dụng:

```ts
getUser(1, (error, user) => {
  if (error) {
    console.error(error);
    return;
  }

  console.log(user);
});
```

Đây là kiểu API rất phổ biến trong Node.js đời đầu.

---

# 4. Callback Hell

Giả sử backend cần:

1. Lấy user
    
2. Lấy orders của user
    
3. Lấy sản phẩm trong order
    
4. Lấy thông tin thanh toán
    

Với callback:

```ts
getUser(1, (user) => {
  getOrders(user.id, (orders) => {
    getProducts(orders[0].id, (products) => {
      getPayment(products[0].id, (payment) => {
        console.log(payment);
      });
    });
  });
});
```

Code bắt đầu bị lồng:

```text
getUser
   └── getOrders
          └── getProducts
                 └── getPayment
                        └── ...
```

Hiện tượng này thường được gọi là:

**Callback Hell**

hoặc:

**Pyramid of Doom**

### Nhược điểm

- Khó đọc
    
- Khó maintain
    
- Error handling phức tạp
    
- Logic bị lồng nhiều tầng
    

Đây là một trong những lý do **Promise** trở nên quan trọng.

---

# 5. Promise

## Promise là gì?

`Promise<T>` đại diện cho **một kết quả có thể chưa có ngay bây giờ nhưng sẽ được hoàn thành trong tương lai**.

Một Promise thường được hiểu qua 3 trạng thái:

```text
              ┌── fulfilled
              │
pending ──────┤
              │
              └── rejected
```

### Pending

Đang xử lý.

### Fulfilled

Xử lý thành công.

### Rejected

Xử lý thất bại.

---

# 6. Tạo Promise

```ts
const promise = new Promise<string>((resolve, reject) => {
  const success = true;

  if (success) {
    resolve("Thành công");
  } else {
    reject(new Error("Thất bại"));
  }
});
```

Hai function quan trọng:

```text
resolve(value)
```

→ hoàn thành Promise thành công.

```text
reject(error)
```

→ Promise thất bại.

---

# 7. Promise

TypeScript cho phép khai báo kiểu dữ liệu Promise sẽ trả về:

```ts
Promise<string>
Promise<number>
Promise<User>
Promise<Product[]>
```

Ví dụ:

```ts
interface User {
  id: number;
  name: string;
}

function getUser(): Promise<User> {
  return Promise.resolve({
    id: 1,
    name: "Long",
  });
}
```

`Promise<User>` có thể hiểu:

> Function này chưa trả `User` ngay. Nó trả một Promise mà khi thành công sẽ cung cấp `User`.

---

# 8. `.then()`

Dùng `.then()` để xử lý kết quả khi Promise thành công.

```ts
getUser().then((user) => {
  console.log(user);
});
```

Có thể chain:

```ts
getUser()
  .then((user) => {
    return getOrders(user.id);
  })
  .then((orders) => {
    return getProducts(orders[0].id);
  })
  .then((products) => {
    console.log(products);
  });
```

So với callback:

```text
Callback

getUser
   └── getOrders
          └── getProducts
                 └── getPayment
```

Promise chain phẳng hơn:

```text
getUser()
   ↓
.then(getOrders)
   ↓
.then(getProducts)
   ↓
.then(getPayment)
```

---

# 9. `.catch()`

Dùng để xử lý Promise bị rejected.

```ts
getUser()
  .then((user) => {
    console.log(user);
  })
  .catch((error) => {
    console.error(error);
  });
```

Một `.catch()` cuối chain có thể xử lý rejection/error phát sinh từ các bước trước:

```ts
getUser()
  .then((user) => getOrders(user.id))
  .then((orders) => getProducts(orders[0].id))
  .then((products) => console.log(products))
  .catch((error) => {
    console.error(error);
  });
```

---

# 10. `.finally()`

`finally()` chạy khi Promise kết thúc bất kể thành công hay thất bại.

```ts
fetchData()
  .then((data) => {
    console.log(data);
  })
  .catch((error) => {
    console.error(error);
  })
  .finally(() => {
    console.log("Finished");
  });
```

Ví dụ thực tế:

```ts
showLoading();

fetchData()
  .then(handleData)
  .catch(handleError)
  .finally(() => {
    hideLoading();
  });
```

Rất phù hợp cho:

- Tắt loading
    
- Cleanup
    
- Đóng connection/resource phù hợp
    

---

# 11. Async/Await

`async/await` là cú pháp giúp làm việc với Promise theo cách dễ đọc hơn.

Thay vì:

```ts
getUser()
  .then((user) => getOrders(user.id))
  .then((orders) => console.log(orders))
  .catch((error) => console.error(error));
```

Có thể viết:

```ts
async function main() {
  try {
    const user = await getUser();
    const orders = await getOrders(user.id);

    console.log(orders);
  } catch (error) {
    console.error(error);
  }
}
```

Nhìn gần giống code synchronous:

```text
user
 ↓
orders
 ↓
products
 ↓
payment
```

nhưng các operation vẫn sử dụng Promise/asynchronous mechanism.

---

# 12. `async`

Khi đặt `async` trước function:

```ts
async function getName() {
  return "Long";
}
```

Function này thực tế trả:

```ts
Promise<string>
```

Có thể viết rõ:

```ts
async function getName(): Promise<string> {
  return "Long";
}
```

Sử dụng:

```ts
getName().then((name) => {
  console.log(name);
});
```

hoặc:

```ts
const name = await getName();
```

> [!important]  
> Một `async function` **luôn trả về Promise**.

---

# 13. `await`

`await` chờ Promise settle và, nếu fulfilled, lấy giá trị của nó.

```ts
const user = await getUser();
```

Nếu:

```ts
getUser(): Promise<User>
```

thì:

```ts
const user = await getUser();
```

`user` có type:

```ts
User
```

Có thể hình dung:

```text
getUser()
   │
   ▼
Promise<User>
   │
 await
   ▼
 User
```

---

# 14. Error Handling với try/catch

Promise style:

```ts
getUser()
  .then((user) => {
    console.log(user);
  })
  .catch((error) => {
    console.error(error);
  });
```

Async/Await style:

```ts
try {
  const user = await getUser();

  console.log(user);
} catch (error) {
  console.error(error);
}
```

Trong TypeScript hiện đại, error trong `catch` thường nên được xử lý cẩn thận:

```ts
try {
  await getUser();
} catch (error) {
  if (error instanceof Error) {
    console.error(error.message);
  }
}
```

---

# 15. Ví dụ thực tế: gọi API

```ts
interface User {
  id: number;
  name: string;
  email: string;
}

async function getUser(id: number): Promise<User> {
  const response = await fetch(`/api/users/${id}`);

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`);
  }

  const user = (await response.json()) as User;

  return user;
}
```

Sử dụng:

```ts
async function main() {
  try {
    const user = await getUser(1);

    console.log(user.name);
  } catch (error) {
    if (error instanceof Error) {
      console.error(error.message);
    }
  }
}
```

Luồng:

```text
main()
  │
  ▼
getUser(1)
  │
  ▼
fetch()
  │
  ▼
Promise<Response>
  │
 await
  ▼
Response
  │
  ▼
response.json()
  │
 await
  ▼
User
```

---

# 16. `throw` trong async function

Có thể throw error:

```ts
async function getUser(id: number): Promise<User> {
  if (id <= 0) {
    throw new Error("Invalid ID");
  }

  // ...
}
```

Trong `async function`:

```ts
throw new Error("Something went wrong");
```

làm Promise được trả về bị **rejected**.

Ví dụ:

```ts
async function test(): Promise<string> {
  throw new Error("Failed");
}
```

tương đương về ý tưởng với:

```ts
function test(): Promise<string> {
  return Promise.reject(new Error("Failed"));
}
```

---

# 17. Sequential Await

Code:

```ts
const user = await getUser();

const orders = await getOrders(user.id);

const products = await getProducts();
```

Chạy theo thứ tự:

```text
getUser
   ↓
  wait
   ↓
getOrders
   ↓
  wait
   ↓
getProducts
```

Nếu bước sau **phụ thuộc bước trước**, đây thường là điều cần thiết.

Ví dụ:

```ts
const user = await getUser();

const orders = await getOrders(user.id);
```

Không thể lấy `orders` theo `user.id` trước khi biết `user`.

---

# 18. Concurrent Promises

Giả sử hai request độc lập:

```ts
const users = await getUsers();
const products = await getProducts();
```

Cách trên chờ users xong rồi mới bắt đầu/chờ products theo thứ tự biểu thức.

Nếu hai operation độc lập, có thể bắt đầu cả hai trước:

```ts
const usersPromise = getUsers();
const productsPromise = getProducts();

const users = await usersPromise;
const products = await productsPromise;
```

Hoặc thường dùng:

```ts
const [users, products] = await Promise.all([
  getUsers(),
  getProducts(),
]);
```

Luồng:

```text
             ┌── getUsers() ──────┐
START ───────┤                     ├── Promise.all()
             └── getProducts() ───┘
```

Hai công việc được khởi tạo mà không cần chờ nhau hoàn thành trước.

> [!warning]  
> Chỉ làm như vậy khi các operation **không phụ thuộc lẫn nhau**.

---

# 19. Promise.all()

```ts
const [user, products] = await Promise.all([
  getUser(),
  getProducts(),
]);
```

Nếu tất cả fulfilled:

```text
Promise 1 ── ✓
Promise 2 ── ✓
Promise 3 ── ✓
              │
              ▼
         Promise.all ✓
```

Nếu một Promise reject:

```text
Promise 1 ── ✓
Promise 2 ── ✗
Promise 3 ── ...
              │
              ▼
         Promise.all ✗
```

Promise được trả bởi `Promise.all()` sẽ reject khi một input Promise reject.

---

# 20. Promise.allSettled()

Nếu muốn chờ **tất cả hoàn thành**, kể cả có request thất bại:

```ts
const results = await Promise.allSettled([
  getUser(),
  getProducts(),
  getOrders(),
]);
```

Kết quả có thể giống:

```ts
[
  {
    status: "fulfilled",
    value: ...
  },
  {
    status: "rejected",
    reason: ...
  }
]
```

Hữu ích khi failure của một task không nên ngăn việc thu thập kết quả các task còn lại.

---

# 21. Promise.race()

```ts
const result = await Promise.race([
  requestA(),
  requestB(),
]);
```

Promise nào **settle trước** thì quyết định kết quả của Promise trả về.

```text
A ───────────── ✓

B ───── ✓
       │
       └── kết quả race
```

Lưu ý: "settle trước" có thể là fulfilled **hoặc rejected**.

---

# 22. Promise.any()

```ts
const result = await Promise.any([
  serverA(),
  serverB(),
  serverC(),
]);
```

Lấy Promise **fulfilled đầu tiên**.

Nếu tất cả đều reject, `Promise.any()` reject với `AggregateError`.

Khác với `race()`:

```text
race → settle đầu tiên
any  → fulfilled đầu tiên
```

---

# 23. Callback vs Promise vs Async/Await

|Đặc điểm|Callback|Promise|Async/Await|
|---|---|---|---|
|Async|✅|✅|✅|
|Dễ đọc|⚠️|✅|✅✅|
|Error handling|Khó hơn|`.catch()`|`try/catch`|
|Dễ chain|Khó|✅|✅|
|Nested code|Dễ xảy ra|Ít hơn|Ít|
|Modern TypeScript|Vẫn dùng|Rất quan trọng|Rất phổ biến|

Async/Await **không thay thế Promise ở tầng cơ chế**.

Có thể hình dung:

```text
Callback
   │
   ▼
Promise
   │
   ▼
Async / Await
```

`async/await` cung cấp syntax thuận tiện để tiêu thụ và tổ chức Promise.

---

# 24. Sai lầm phổ biến

## Quên `await`

Sai:

```ts
const user = getUser();

console.log(user.name);
```

Nếu:

```ts
getUser(): Promise<User>
```

thì `user` là:

```ts
Promise<User>
```

chứ không phải:

```ts
User
```

Đúng:

```ts
const user = await getUser();

console.log(user.name);
```

---

## Dùng `await` tuần tự không cần thiết

Không tối ưu khi hai request độc lập:

```ts
const users = await getUsers();
const products = await getProducts();
```

Có thể:

```ts
const [users, products] = await Promise.all([
  getUsers(),
  getProducts(),
]);
```

---

## Quên xử lý rejection

```ts
async function main() {
  const data = await fetchData();
}
```

Nếu operation có khả năng thất bại và tầng hiện tại chịu trách nhiệm xử lý:

```ts
async function main() {
  try {
    const data = await fetchData();
  } catch (error) {
    console.error(error);
  }
}
```

Tuy nhiên không phải function nào cũng cần `try/catch`; đôi khi nên để error propagate lên tầng xử lý phù hợp.

---

# 25. Ví dụ Backend TypeScript / NestJS

Trong [[NestJS]], bạn sẽ gặp `async/await` rất nhiều.

Ví dụ Service:

```ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  async findOne(id: number): Promise<User | null> {
    return await this.userRepository.findOne({
      where: { id },
    });
  }
}
```

Controller:

```ts
@Get(':id')
async findOne(
  @Param('id') id: number,
): Promise<User | null> {
  return await this.usersService.findOne(id);
}
```

Luồng:

```text
HTTP Request
     │
     ▼
Controller
     │
     ▼
UsersService
     │
     ▼
Repository
     │
     ▼
PostgreSQL
     │
     ▼
Promise<User | null>
     │
    await
     ▼
User / null
     │
     ▼
HTTP Response
```

> [!note]  
> Trong một số trường hợp `return await something()` là không cần thiết và có thể viết trực tiếp `return something()`. Nhưng `await` vẫn cần khi bạn phải dùng kết quả, phối hợp nhiều Promise hoặc xử lý lỗi tại vị trí đó.

---

# 26. Mental Model

Khi nhìn thấy:

```ts
const user = await getUser();
```

hãy nghĩ:

```text
getUser()
    │
    ▼
Promise<User>
    │
    │ chưa có kết quả
    │
    ▼
operation hoàn thành
    │
    ▼
Promise fulfilled
    │
   await
    ▼
User
```

Nếu thất bại:

```text
getUser()
    │
    ▼
Promise<User>
    │
    ▼
rejected
    │
    ▼
throw
    │
    ▼
catch
```

---

# 27. Tóm tắt

### Callback

Function được truyền vào function khác:

```ts
doSomething((result) => {
  console.log(result);
});
```

---

### Promise

Đại diện cho kết quả asynchronous:

```ts
Promise<User>
```

Xử lý bằng:

```ts
promise
  .then(...)
  .catch(...)
  .finally(...);
```

---

### Async

Biến function thành function trả Promise:

```ts
async function getUser(): Promise<User> {
  // ...
}
```

---

### Await

Lấy kết quả fulfilled của Promise theo cú pháp tuần tự:

```ts
const user = await getUser();
```

---

### Error

```ts
try {
  const user = await getUser();
} catch (error) {
  console.error(error);
}
```

---

### Nhiều Promise độc lập

```ts
const [users, products] = await Promise.all([
  getUsers(),
  getProducts(),
]);
```

---

# 28. Cheat Sheet

```ts
// Promise
const promise: Promise<string> =
  Promise.resolve("Hello");
```

```ts
// async
async function hello(): Promise<string> {
  return "Hello";
}
```

```ts
// await
const result = await hello();
```

```ts
// error handling
try {
  await doSomething();
} catch (error) {
  console.error(error);
}
```

```ts
// concurrent
const [a, b] = await Promise.all([
  getA(),
  getB(),
]);
```

```ts
// wait for every result
const results = await Promise.allSettled([
  getA(),
  getB(),
]);
```

---

# 29. Câu hỏi tự kiểm tra

1. Callback là gì?
    
2. Callback Hell xảy ra khi nào?
    
3. `Promise<T>` có ý nghĩa gì?
    
4. Promise có những trạng thái nào?
    
5. `resolve()` và `reject()` khác nhau thế nào?
    
6. `.then()`, `.catch()`, `.finally()` dùng khi nào?
    
7. `async function` trả về kiểu gì?
    
8. `await` làm gì?
    
9. `Promise<User>` khác `User` như thế nào?
    
10. Khi nào nên dùng `Promise.all()`?
    
11. `Promise.all()` khác `Promise.allSettled()` như thế nào?
    
12. `Promise.race()` khác `Promise.any()` như thế nào?
    
13. Tại sao `async/await` vẫn liên quan đến Promise?
    
14. Tại sao không nên `await` tuần tự các công việc độc lập?
    

---

## 🔗 Liên kết nên học tiếp

- [[JavaScript Event Loop]]
    
- [[TypeScript Functions]]
    
- [[TypeScript Generics]]
    
- [[TypeScript Error Handling]]
    
- [[NodeJS Event Loop]]
    
- [[NestJS]]
    
- [[NestJS Controller]]
    
- [[NestJS Service]]
    
- [[TypeORM]]
    
- [[REST API]]
    
- [[HTTP Request Response]]