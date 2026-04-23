
---

# 🧠 1. `useState` là gì?

`useState` là hook dùng để **quản lý state (dữ liệu thay đổi theo thời gian)** trong component.

👉 Khi state thay đổi → component **re-render**

---

# ⚙️ 2. Cú pháp cơ bản

```tsx
const [state, setState] = useState(initialValue);
```

- `state` → giá trị hiện tại
    
- `setState` → hàm cập nhật
    
- `initialValue` → giá trị ban đầu
    

---

# 📌 3. Ví dụ cơ bản

```tsx
"use client";

import { useState } from "react";

export default function Counter() {
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>Tăng</button>
    </div>
  );
}
```

---

# 🔥 4. Nguyên lý hoạt động (quan trọng)

### ❗ 1. State update là async

```tsx
setCount(count + 1);
console.log(count); // vẫn là giá trị cũ
```

👉 React **batch update** → không update ngay lập tức

---

### ❗ 2. Functional update (rất quan trọng)

Khi update dựa vào state cũ:

```tsx
setCount(prev => prev + 1);
```

👉 Luôn dùng cách này khi:

- update nhiều lần liên tiếp
    
- phụ thuộc state cũ
    

---

### ❗ 3. Re-render

- Khi `setState` → component chạy lại từ đầu
    
- Nhưng **React tối ưu (diff DOM)** → không render toàn bộ UI
    

---

# 🧩 5. Các kiểu dữ liệu với `useState`

---

## 🔹 1. Primitive

```tsx
const [name, setName] = useState("Naga");
const [age, setAge] = useState(20);
const [isOpen, setIsOpen] = useState(false);
```

---

## 🔹 2. Object

```tsx
const [user, setUser] = useState({
  name: "Naga",
  age: 20,
});
```

👉 Update phải **spread**:

```tsx
setUser(prev => ({
  ...prev,
  age: 21,
}));
```

---

## 🔹 3. Array

```tsx
const [list, setList] = useState<number[]>([]);
```

### ➕ Thêm phần tử

```tsx
setList(prev => [...prev, 1]);
```

### ❌ Xóa phần tử

```tsx
setList(prev => prev.filter(item => item !== 1));
```

---

# ⚡ 6. Lazy Initialization (ít người biết)

```tsx
const [value, setValue] = useState(() => {
  console.log("chạy 1 lần");
  return 100;
});
```

👉 Function chỉ chạy **1 lần duy nhất**

👉 Dùng khi:

- tính toán nặng
    
- đọc localStorage
    

---

# 🔁 7. Update nhiều lần trong 1 lần render

❌ Sai:

```tsx
setCount(count + 1);
setCount(count + 1);
```

👉 Kết quả: +1

✅ Đúng:

```tsx
setCount(prev => prev + 1);
setCount(prev => prev + 1);
```

👉 Kết quả: +2

---

# 🧱 8. useState trong Next.js (QUAN TRỌNG)

## ❗ Chỉ dùng trong Client Component

```tsx
"use client";
```

👉 Nếu không:

- ❌ lỗi ngay
    

---

## ❗ Không dùng trong Server Component

```tsx
// ❌ sai
export default function Page() {
  const [count, setCount] = useState(0);
}
```

---

# ⚡ 9. Khi nào KHÔNG nên dùng useState?

---

## ❌ 1. Dữ liệu từ API (Next.js)

👉 KHÔNG nên:

```tsx
useEffect(() => {
  fetchData().then(setData);
}, []);
```

👉 NÊN:

- fetch ở server (App Router)
    

---

## ❌ 2. State phức tạp

👉 dùng:

- `useReducer`
    

---

## ❌ 3. Global state

👉 dùng:

- Context
    
- Zustand
    
- Redux
    

---

# 🧠 10. Best Practices

---

## ✅ 1. Tách state nhỏ

❌

```tsx
const [state, setState] = useState({ a: 1, b: 2 });
```

✅

```tsx
const [a, setA] = useState(1);
const [b, setB] = useState(2);
```

---

## ✅ 2. Không mutate

❌

```tsx
user.age = 21;
setUser(user);
```

✅

```tsx
setUser(prev => ({ ...prev, age: 21 }));
```

---

## ✅ 3. Đặt tên rõ ràng

```tsx
const [isLoading, setIsLoading] = useState(false);
```

---

## ✅ 4. Reset state

```tsx
setCount(0);
```

---

# 🚨 11. Lỗi phổ biến

---

## ❌ Infinite render

```tsx
setCount(count + 1); // đặt ngoài event
```

---

## ❌ Stale state

```tsx
setCount(count + 1);
setCount(count + 1);
```

---

## ❌ Update object sai

```tsx
setUser({ age: 21 }); // mất name
```

---

# 🧪 12. Ví dụ thực tế (Next.js)

```tsx
"use client";

import { useState } from "react";

export default function TodoApp() {
  const [todos, setTodos] = useState<string[]>([]);
  const [input, setInput] = useState("");

  const addTodo = () => {
    if (!input) return;
    setTodos(prev => [...prev, input]);
    setInput("");
  };

  return (
    <div>
      <input
        value={input}
        onChange={e => setInput(e.target.value)}
      />
      <button onClick={addTodo}>Add</button>

      <ul>
        {todos.map((todo, i) => (
          <li key={i}>{todo}</li>
        ))}
      </ul>
    </div>
  );
}
```

---

# 🎯 13. Tư duy chuẩn khi dùng useState

- State = dữ liệu UI thay đổi
    
- Không lưu thứ có thể tính toán lại
    
- Tránh duplicate state
    

---

# 🧩 14. Khi kết hợp với hook khác

|Hook|Dùng với useState để|
|---|---|
|`useEffect`|gọi API / side effect|
|`useMemo`|tối ưu tính toán|
|`useCallback`|tránh re-render|
|`useRef`|giữ value không re-render|

---

# 🚀 Kết luận

👉 `useState` là:

- nền tảng của React/Next.js
    
- dùng cho UI state đơn giản → trung bình
    

👉 Muốn giỏi frontend:

- hiểu sâu **functional update**
    
- tránh **mutate**
    
- biết khi nào **KHÔNG dùng useState**
    

---

Nếu bạn muốn, mình có thể:

- đưa bài tập luyện `useState` từ dễ → khó
    
- hoặc review code của bạn để chỉ ra lỗi thực tế (cái này giúp lên trình rất nhanh)