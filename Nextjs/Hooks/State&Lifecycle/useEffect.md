
---

# 🧠 1. `useEffect` là gì?

`useEffect` là hook dùng để xử lý **side effects** trong component.

👉 Side effect là:

- call API
    
- thao tác DOM
    
- setInterval / setTimeout
    
- addEventListener
    
- sync data bên ngoài
    

---

# ⚙️ 2. Cú pháp

```tsx
useEffect(() => {
  // code chạy

  return () => {
    // cleanup (optional)
  };
}, [dependencies]);
```

---

# 🔥 3. 3 kiểu useEffect quan trọng nhất

---

## 📌 1. Chạy 1 lần (mount)

```tsx
useEffect(() => {
  console.log("chạy 1 lần");
}, []);
```

👉 giống `componentDidMount`

---

## 📌 2. Chạy khi dependency thay đổi

```tsx
useEffect(() => {
  console.log("count thay đổi");
}, [count]);
```

---

## 📌 3. Chạy mỗi lần render

```tsx
useEffect(() => {
  console.log("chạy liên tục");
});
```

👉 ⚠️ hiếm khi dùng

---

# ⚡ 4. Cleanup function (RẤT QUAN TRỌNG)

```tsx
useEffect(() => {
  const id = setInterval(() => {
    console.log("running...");
  }, 1000);

  return () => {
    clearInterval(id);
  };
}, []);
```

👉 cleanup chạy khi:

- component unmount
    
- trước khi effect chạy lại
    

---

# 🧩 5. Nguyên lý hoạt động (hiểu sâu)

---

## 🔁 1. Flow

1. Render UI
    
2. Commit DOM
    
3. `useEffect` chạy
    

👉 luôn chạy **sau render**

---

## ⚠️ 2. Không block UI

`useEffect` là **async non-blocking**

---

## ⚠️ 3. Dependency array = trigger

```tsx
[count]
```

👉 React so sánh bằng **shallow compare**

---

# 💣 6. Lỗi phổ biến (rất quan trọng)

---

## ❌ Infinite loop

```tsx
useEffect(() => {
  setCount(count + 1);
}, [count]);
```

👉 loop vô hạn

---

## ❌ Missing dependency

```tsx
useEffect(() => {
  console.log(count);
}, []);
```

👉 bug logic (stale data)

3

## ❌ Object dependency

```tsx
useEffect(() => {}, [{ a: 1 }]);
```

👉 luôn chạy lại (object mới mỗi render)

---

# 🧠 7. Dependency chuẩn (rule vàng)

👉 Luôn include:

- state
    
- props
    
- function dùng bên trong effect
    

---

## ✅ Ví dụ đúng

```tsx
useEffect(() => {
  fetchData(id);
}, [id]);
```

---

# ⚡ 8. useEffect trong Next.js (QUAN TRỌNG)

---

## ❗ Chỉ dùng trong Client Component

```tsx
"use client";
```

👉 nếu không → lỗi

---

## ❗ Không nên fetch data bằng useEffect (App Router)

❌

```tsx
useEffect(() => {
  fetch("/api/data").then(res => res.json()).then(setData);
}, []);
```

✅

```tsx
const data = await fetch(...);
```

👉 dùng Server Component

---

# 🔄 9. Pattern thường dùng

---

## 📌 1. Call API

```tsx
useEffect(() => {
  const fetchData = async () => {
    const res = await fetch("/api");
    const data = await res.json();
    setData(data);
  };

  fetchData();
}, []);
```

---

## 📌 2. Event listener

```tsx
useEffect(() => {
  const handleResize = () => {
    console.log(window.innerWidth);
  };

  window.addEventListener("resize", handleResize);

  return () => {
    window.removeEventListener("resize", handleResize);
  };
}, []);
```

---

## 📌 3. Sync với localStorage

```tsx
useEffect(() => {
  localStorage.setItem("count", count.toString());
}, [count]);
```

---

## 📌 4. Animation loop

```tsx
useEffect(() => {
  let raf = requestAnimationFrame(function loop() {
    // animation logic
    raf = requestAnimationFrame(loop);
  });

  return () => cancelAnimationFrame(raf);
}, []);
```

---

# 🧪 10. Advanced useEffect

---

## 🔥 1. Cleanup trước khi chạy lại

```tsx
useEffect(() => {
  console.log("effect");

  return () => {
    console.log("cleanup");
  };
}, [count]);
```

👉 flow:

- cleanup cũ
    
- chạy effect mới
    

---

## 🔥 2. Avoid stale closure

```tsx
useEffect(() => {
  const id = setInterval(() => {
    setCount(prev => prev + 1);
  }, 1000);

  return () => clearInterval(id);
}, []);
```

---

## 🔥 3. useEffect + useRef

```tsx
const mounted = useRef(false);

useEffect(() => {
  if (!mounted.current) {
    mounted.current = true;
    return;
  }

  console.log("update only");
}, [count]);
```

---

# 🚀 11. Best Practices

---

## ✅ 1. Tách nhiều useEffect

❌

```tsx
useEffect(() => {
  fetchData();
  addEventListener();
}, []);
```

✅

```tsx
useEffect(() => {
  fetchData();
}, []);

useEffect(() => {
  addEventListener();
}, []);
```

---

## ✅ 2. Không nhét logic nặng

👉 nên tách function riêng

---

## ✅ 3. Luôn cleanup

👉 tránh memory leak

---

## ✅ 4. Không lạm dụng

👉 nhiều logic → dùng:

- custom hook
    
- server component
    

---

# ⚠️ 12. Khi KHÔNG nên dùng useEffect

---

## ❌ 1. Tính toán dữ liệu

👉 dùng:

- `useMemo`
    

---

## ❌ 2. Derived state

👉 không cần effect

```tsx
const fullName = firstName + lastName;
```

---

## ❌ 3. Fetch trong Next.js (App Router)

👉 nên fetch server

---

# 🧠 13. So sánh nhanh

|Hook|Mục đích|
|---|---|
|`useState`|lưu state|
|`useEffect`|side effect|
|`useMemo`|tối ưu tính toán|
|`useRef`|giữ giá trị|

---

# 🎯 14. Ví dụ thực tế (Next.js)

```tsx
"use client";

import { useEffect, useState } from "react";

export default function Example() {
  const [width, setWidth] = useState(0);

  useEffect(() => {
    const handleResize = () => {
      setWidth(window.innerWidth);
    };

    handleResize();

    window.addEventListener("resize", handleResize);

    return () => {
      window.removeEventListener("resize", handleResize);
    };
  }, []);

  return <div>Width: {width}</div>;
}
```
