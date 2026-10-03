# lecture18-2569-starter — fullstack: เชื่อม Frontend กับ Backend API

| โฟลเดอร์    | หน้าที่              | เทคโนโลยี                                  |
| ----------- | -------------------- | ------------------------------------------ |
| `backend/`  | REST API (port 3000) | Express + Prisma + MongoDB + JWT + Zod     |
| `frontend/` | หน้าเว็บ (port 5173) | React + Vite + shadcn/ui + Zustand + axios |

### สรุปไฟล์ที่เกี่ยวข้อง

| ขั้น  | ไฟล์                                                                            | สิ่งที่ทำ                                                                    |
| ----- | ------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| 0     | `backend/.env`, `frontend/.env`                                                 | เปลี่ยนฐานข้อมูลเป็น `lab18_2569`, เพิ่ม `CORS_ORIGIN`, สร้าง `VITE_API_URL` |
| 2     | `backend/src/index.ts`                                                          | เปิด CORS                                                                    |
| 3     | `backend/src/routes/usersRouters_v3.ts`                                         | Login ตอบ `token`, `role`, `studentId`                                       |
| 5     | `frontend/src/lib/api.ts`                                                       | axios instance + ฟังก์ชัน `api()`                                            |
| 6     | `frontend/src/lib/auth-store.ts`                                                | persist token ลง localStorage                                                |
| 7     | `frontend/src/pages/login.tsx`, `frontend/src/components/app-sidebar.tsx`       | Login / Logout                                                               |
| 8     | `backend/src/index.ts`                                                          | ทดลองปิด / เปิด CORS                                                         |
| 9     | `frontend/src/layouts/root-layout.tsx`, `frontend/src/layouts/require-role.tsx` | ตรวจ Login + โหลดข้อมูล + กันหน้าตาม role                                    |
| 10–12 | `frontend/src/lib/enrollment-store.ts`                                          | `getAll`, `enroll`, `addCourse`, `updateCourse`, `removeCourse`              |
| 11    | `frontend/src/pages/student/enrollments.tsx`                                    | ปุ่มลงทะเบียน (STUDENT)                                                      |
| 12    | `frontend/src/components/courses/course-form-dialog.tsx`, `course-table.tsx`    | เพิ่ม / แก้ / ลบ วิชา (ADMIN)                                                |

---

## ขั้นที่ 0 — สร้างฐานข้อมูล `lab18_2569` และแก้ `.env`

### 0.1 สร้างฐานข้อมูล `lab18_2569` ใน MongoDB Atlas

1. เข้า https://cloud.mongodb.com → เลือก Cluster ของตัวเอง → **Browse Collections**
2. กด **+ Create Database**
   - Database name: `lab18_2569`
   - Collection name: `Students` (ต้องใส่อย่างน้อย 1 collection ถึงจะสร้าง database ได้)
3. กด **Create**

### 0.2 แก้ `backend/.env`

```env
DATABASE_URL=mongodb+srv://<username>:<password>@<hostname>/lab18_2569?retryWrites=true&w=majority
```

ไปดูที่เมนู Security Quickstart

แล้ว **เพิ่ม** `CORS_ORIGIN` — `backend/.env` ทั้งไฟล์ควรมี 4 ค่า

```env
# port ของ Backend (ต้องตรงกับ VITE_API_URL ฝั่ง Frontend)
PORT=3000
# ใช้เซ็น/ตรวจ JWT — ห้ามเผยแพร่
JWT_SECRET=this_is_my_very_awesome_secret
# ที่อยู่ MongoDB + ชื่อฐานข้อมูล lab18_2569
DATABASE_URL=mongodb+srv://<username>:<password>@<hostname>/lab18_2569?retryWrites=true&w=majority
# URL ของ Frontend ที่อนุญาตให้เรียก API (ใช้ในขั้นที่ 2)
CORS_ORIGIN=http://localhost:5173
```

### 0.3 สร้าง `frontend/.env`

คัดลอก `frontend/.env.example` เป็น `frontend/.env`

```env
# URL ของ Backend API (ดูขั้นที่ 4)
VITE_API_URL=http://localhost:3000/api/v3
```

---

## ขั้นที่ 1 — ติดตั้งและรัน (เปิด 2 terminal)

```bash
# terminal 1 — backend → http://localhost:3000
cd backend
pnpm install
pnpm db:generate   # สร้าง Prisma Client (ทำใหม่ทุกครั้งที่แก้ schema.prisma)
pnpm db:push       # สร้าง collection / index ใน MongoDB (lab18_2569)
pnpm db:seed       # ใส่ข้อมูลตั้งต้นจาก src/db/*.json (รันซ้ำได้)
pnpm dev

# terminal 2 — frontend → http://localhost:5173
cd frontend
pnpm install
pnpm dev
```

---

## ภาพรวม: ข้อมูลเดินทางอย่างไร

```
Browser (React :5173)                          Backend (Express :3000)                 MongoDB
─────────────────────                          ───────────────────────                 ───────
① หน้า Login ── POST /users/login ──────────▶  ตรวจ password (bcrypt)
             ◀── { token, role, studentId } ──  jwt.sign() + เก็บลง user.tokens ─────▶  Users
   auth-store เก็บ token (localStorage)

② RootLayout → getAll()
   api("/courses")  ── GET + Authorization: Bearer <token> ──▶ authenticateToken  (JWT ถูกต้อง?)
   api("/enrollments")                                          → checkRoles / checkRoleAdmin (สิทธิ์?)
   api("/students")                                             → prisma.xxx.findMany() ───▶  อ่าน
             ◀── { success: true, data } ──────
   enrollment-store (global state) → ทุกหน้าอ่านจาก store

③ ฟอร์ม Add / Update / Delete
   validate ฝั่ง Frontend ผ่าน → POST/PUT/DELETE ─▶ Zod ฝั่ง Backend ตรวจซ้ำ
                                                  ไม่ผ่าน → 400 { errors } ─┐
             ◀── แสดงข้อความ error ในฟอร์ม ◀───────────────────────────────┘
                                                  ผ่าน → prisma.create/update/delete ─▶  เขียน
             ◀── 2xx { data } → อัปเดต state ใน store → UI เปลี่ยนเอง
```

---

## ขั้นที่ 2 — Backend: เปิด CORS

Frontend (`:5173`) กับ Backend (`:3000`) อยู่คนละ **origin** browser จะบล็อก request ถ้า Backend ไม่อนุญาต
(Insomnia / Postman ไม่โดนบล็อก เพราะไม่ใช่ browser)

📝 `backend/src/index.ts` — แทนที่ `TODO ขั้นที่ 2` ด้วย

```ts
import cors from "cors"; // ← ย้ายไปไว้กับ import อื่นด้านบนไฟล์

// CORS middleware: อนุญาตให้ Frontend (Vite dev server คนละ origin) เรียก API ได้
// ตั้งค่า origin ได้หลายค่าคั่นด้วย "," ผ่าน CORS_ORIGIN ใน .env
app.use(
  cors({
    origin: (process.env.CORS_ORIGIN || "http://localhost:5173").split(","),
  }),
);
```

ต้องอยู่ **ก่อน** `app.use(express.json())` และก่อน `app.use("/api/v3/...", router)` ทุกตัว
(แพ็กเกจ `cors` และ `@types/cors` ติดตั้งไว้ใน `package.json` แล้ว)

---

## ขั้นที่ 3 — Backend: Login ต้องตอบสิ่งที่ Frontend ใช้

Frontend ต้องรู้ 3 อย่างหลัง Login: **token** (แนบทุก request), **role** (เลือกเมนู), **studentId** (ของ STUDENT)
เดิมตอบแค่ `tokens` (array ทุก token) — Frontend ไม่รู้ว่าอันไหนล่าสุด และไม่รู้ role

📝 `backend/src/routes/usersRouters_v3.ts` — `POST /api/v3/users/login` ตรง `TODO ขั้นที่ 3`

```ts
// ก่อน
      data: {
        username: user.username,
        tokens: user.tokens,
      },

// หลัง
      data: {
        username: user.username,
        // Frontend ใช้ 3 ค่านี้: token ล่าสุด (แนบใน Authorization header) และ role/studentId
        token,
        role: user.role,
        studentId: user.studentId,
      },
```

`token` คือตัวแปรที่สร้างไว้แล้วด้านบนใน route เดียวกัน บันทัดที่ 111

```ts
// ใส่ข้อมูลผู้ใช้ไว้ใน payload ของ JWT (Frontend ถอดออกมาแสดงผลได้)
const token = jwt.sign(
  { username: user.username, studentId: user.studentId, role: user.role },
  jwt_secret,
  { expiresIn: "30m" }, // หมดอายุใน 30 นาที
);
```

ทดสอบด้วย Insomnia: `POST http://localhost:3000/api/v3/users/login` → `data` ต้องมี `token`, `role`, `studentId`

---

## ขั้นที่ 4 — Frontend: ตั้ง URL ของ API

ทำไปแล้วในขั้นที่ 0.3 — ตรวจว่า `frontend/.env` มี

```env
# ตัวแปรที่ขึ้นต้นด้วย VITE_ เท่านั้นที่ Vite ส่งให้โค้ดฝั่ง browser
VITE_API_URL=http://localhost:3000/api/v3
```

> เปลี่ยน `PORT` ของ Backend → แก้ `VITE_API_URL` ให้ตรง / เปลี่ยน port ของ Frontend → แก้ `CORS_ORIGIN` ให้ตรง
> แก้ `.env` แล้วต้อง **รัน `pnpm dev` ใหม่**

---

## ขั้นที่ 5 — Frontend: ตัวกลางยิง API (`src/lib/api.ts`)

ทุกหน้าเรียก API ผ่านฟังก์ชัน `api()` ตัวเดียว เพื่อไม่ต้องเขียนซ้ำ: ต่อ URL, แนบ token, แกะ `{ success, data }`, จัดการ error

📝 `frontend/src/lib/api.ts` — **แทนที่ทั้งไฟล์** ด้วย

```ts
import axios, { AxiosError, type Method } from "axios";

import { useAuthStore } from "@/lib/auth-store";

/**
 * ตัวกลางเรียก Backend API ด้วย axios
 *
 * - ต่อ URL จาก VITE_API_URL (ไฟล์ .env) เช่น http://localhost:3000/api/v3 → baseURL ของ axios
 * - แนบ Authorization: Bearer <token> ให้อัตโนมัติถ้า Login อยู่ (request interceptor)
 * - Backend ตอบรูปแบบ { success, data } หรือ { success: false, message, errors }
 *   ถ้าไม่สำเร็จจะ throw ApiError พร้อมข้อความจาก Backend ให้หน้าเว็บนำไปแสดง
 * - 401/403 = token หมดอายุ / ถูก logout ไปแล้ว → ล้างสถานะ Login ให้ไปหน้า Login ใหม่
 */
export const API_URL =
  import.meta.env.VITE_API_URL ?? "http://localhost:3000/api/v3";

export class ApiError extends Error {
  status: number;

  constructor(status: number, message: string) {
    super(message);
    this.status = status;
  }
}

type ApiResponse<T> = {
  success: boolean;
  data?: T;
  message?: string;
  errors?: string;
};

// ── 5.1 axios instance กลาง — ทุก request ใช้ baseURL เดียวกัน ──────────
// (axios ตั้ง Content-Type: application/json และแปลง body เป็น JSON ให้เอง)
export const http = axios.create({ baseURL: API_URL });

// request interceptor: ทำงาน "ก่อน" ส่งทุก request
// แนบ token ทุก request ยกเว้นที่ตั้ง skipAuth (เช่น /users/login)
http.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token; // อ่าน store นอก React component
  if (token && !config.skipAuth) {
    config.headers.Authorization = `Bearer ${token}`; // ← Backend อ่านตรงนี้ใน authenticateToken
  }
  return config;
});

// ให้ config รู้จัก field skipAuth (ใช้ภายใน interceptor ด้านบน)
declare module "axios" {
  interface AxiosRequestConfig {
    skipAuth?: boolean;
  }
}

// ── 5.2 ฟังก์ชัน api() — แกะข้อมูลและแปลง error ─────────────────────────
export async function api<T>(
  path: string,
  options: { method?: Method; body?: unknown; auth?: boolean } = {},
): Promise<T> {
  const { method = "GET", body, auth = true } = options;

  try {
    const res = await http.request<ApiResponse<T>>({
      url: path,
      method,
      data: body,
      skipAuth: !auth,
    });

    // บาง route ตอบ status 200 แต่ success: false
    if (res.data?.success === false) {
      throw new ApiError(
        res.status,
        res.data.errors ?? res.data.message ?? `Request failed (${res.status})`,
      );
    }
    return res.data.data as T; // ← คืนเฉพาะ data ให้หน้าเว็บใช้ต่อ
  } catch (err) {
    if (err instanceof ApiError) throw err;

    const axiosErr = err as AxiosError<ApiResponse<T>>;
    // ไม่มี response = ต่อ server ไม่ได้ (server ไม่ได้รัน / CORS ไม่ผ่าน)
    if (!axiosErr.response) {
      throw new ApiError(0, `เชื่อมต่อ Backend ไม่ได้ (${API_URL})`);
    }

    const { status, data } = axiosErr.response;
    // 401/403 = token หมดอายุ / ถูก logout → ล้าง token → RootLayout พาไปหน้า Login เอง
    if (auth && (status === 401 || status === 403)) {
      useAuthStore.getState().clear();
    }
    // errors = ข้อความจาก zod (Validation failed), message = ข้อความทั่วไป
    throw new ApiError(
      status,
      data?.errors ?? data?.message ?? `Request failed (${status})`,
    );
  }
}
```

### 5.3 เทียบ axios กับ `fetch()`

โปรเจคนี้ใช้ axios แต่ทำแบบเดียวกันด้วย `fetch()` ที่มีใน browser อยู่แล้วก็ได้ ต่างกันตรงที่ต้องทำเองหลายอย่าง
(ส่วนนี้ไว้อ่านเทียบ ไม่ต้องใส่ในโปรเจค)

```ts
// ── axios ──────────────────────────────────────────────
const courses = await api<Course[]>("/courses"); // GET
await api("/courses", { method: "POST", body: newCourse }); // POST

// ── fetch() (เขียนเองทั้งหมด) ─────────────────────────
const res = await fetch(`${API_URL}/courses`, {
  method: "POST",
  headers: {
    "Content-Type": "application/json", // axios ตั้งให้เอง
    Authorization: `Bearer ${token}`, // axios ใช้ interceptor แนบให้
  },
  body: JSON.stringify(newCourse), // axios แปลงเป็น JSON ให้เอง
});
const json = await res.json(); // axios แปลง response ให้เอง (res.data)
if (!res.ok) throw new Error(json.message); // fetch ไม่ throw เองเมื่อได้ 4xx/5xx — axios throw ให้
```

---

## ขั้นที่ 6 — Frontend: เก็บสถานะ Login (`src/lib/auth-store.ts`)

ตอนนี้ store ยังไม่ persist → **รีเฟรชหน้าแล้วต้อง Login ใหม่** ทำให้ persist ลง localStorage **เฉพาะ token**
ส่วน `username` / `role` / `studentId` ถอดจาก payload ของ JWT (ฟังก์ชัน `authFromToken` มีให้แล้ว)

📝 `frontend/src/lib/auth-store.ts`

1. เพิ่ม import ใต้ `import { create } from "zustand";`

```ts
import { persist } from "zustand/middleware";
```

2. แทนที่ `useAuthStore` ตรง `TODO ขั้นที่ 6` ด้วย

```ts
export const useAuthStore = create<AuthStore>()(
  persist(
    (set) => ({
      ...emptyAuth,
      setAuth: (token) => set(authFromToken(token)), // Login สำเร็จ → ถอด token เป็น state
      clear: () => set(emptyAuth), // Logout / token หมดอายุ
    }),
    {
      name: "lecture18-auth", // key ใน localStorage
      // เขียนลง localStorage แค่ token
      partialize: (state) => ({ token: state.token }),
      // ตอนโหลดกลับ (รีเฟรชหน้า) → ถอดค่าที่เหลือจาก token
      merge: (persisted, current) => ({
        ...current,
        ...authFromToken((persisted as Partial<AuthState> | undefined)?.token),
      }),
    },
  ),
);
```

> ถอด JWT ฝั่ง Frontend เพื่อ **แสดงผล** เท่านั้น ไม่ได้ verify ลายเซ็น — Backend ตรวจเองทุก request

---

## ขั้นที่ 7 — Frontend: Login / Logout

### 7.1 หน้า Login

📝 `frontend/src/pages/login.tsx`

1. แก้ / เพิ่ม import

```ts
import { Navigate, useNavigate } from "react-router"; // ← เพิ่ม useNavigate
// ...
import { api } from "@/lib/api"; // ← เพิ่ม
import { useAuthStore } from "@/lib/auth-store";
import type { User } from "@/lib/types"; // ← เพิ่ม
```

2. แทนที่ `TODO ขั้นที่ 7: type LoginResponse` ด้วย

```ts
// data ที่ POST /api/v3/users/login ตอบกลับมา (ขั้นที่ 3)
type LoginResponse = {
  username: string;
  token: string;
  role: User["role"];
  studentId?: string | null;
};
```

3. ในฟังก์ชัน `LoginPage` ใต้ `const token = ...` แทนที่ TODO ด้วย

```ts
const setAuth = useAuthStore((s) => s.setAuth);
const navigate = useNavigate();
```

4. แทนที่ `onSubmit` ด้วย

```ts
async function onSubmit(values: LoginValues) {
  try {
    const data = await api<LoginResponse>("/users/login", {
      method: "POST",
      body: values, // { username, password }
      auth: false, // ยังไม่มี token → ไม่ต้องแนบ
    });
    setAuth(data.token); // เก็บ token ลง auth-store
    navigate("/", { replace: true }); // ไปหน้าแรก
  } catch (err) {
    // เช่น 401 "Invalid username or password" → แสดงใต้ฟอร์ม
    form.setError("root", { message: (err as Error).message });
  }
}
```

### 7.2 ปุ่ม Logout

📝 `frontend/src/components/app-sidebar.tsx`

1. เพิ่ม import

```ts
import { api } from "@/lib/api";
```

2. แทนที่ `handleLogout` ตรง `TODO ขั้นที่ 7` ด้วย

```ts
// POST /users/logout ลบ token ทั้งหมดของ user ใน DB — ไม่ว่าสำเร็จหรือไม่ก็ล้างฝั่งเราด้วย
// (ล้างแล้ว RootLayout จะพาไปหน้า Login เอง)
const handleLogout = async () => {
  try {
    await api("/users/logout", { method: "POST" });
  } catch {
    // token หมดอายุไปแล้วก็ไม่เป็นไร
  } finally {
    clear();
  }
};
```

---

## ขั้นที่ 8 — ก่อนและหลังใช้ CORS

ตอนนี้ Login ได้แล้ว ลองให้เห็นเองว่าถ้าไม่มี CORS จะเกิดอะไรขึ้น

### 8.1 ก่อน — ปิด CORS ชั่วคราว

`backend/src/index.ts` คอมเมนต์ `app.use(cors(...))` ทิ้ง แล้ว save (nodemon รีสตาร์ต server ให้เอง)

```ts
// import cors from "cors";

// ❌ ปิด CORS ชั่วคราวเพื่อทดลอง
// app.use(
//   cors({
//     origin: (process.env.CORS_ORIGIN || "http://localhost:5173").split(","),
//   }),
// );

app.use(express.json());
```

ลอง Login ที่ http://localhost:5173 แล้วเปิด DevTools (F12)

| ดูที่    | ผลที่เห็น                                                                                                                                                                                                                   |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| หน้าเว็บ | ข้อความ **"เชื่อมต่อ Backend ไม่ได้ (http://localhost:3000/api/v3)"**                                                                                                                                                       |
| Console  | `Access to XMLHttpRequest at 'http://localhost:3000/api/v3/users/login' from origin 'http://localhost:5173' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.` |
| Network  | request ขึ้นสีแดง สถานะ **CORS error**                                                                                                                                                                                      |
| Insomnia | ยิง `POST /api/v3/users/login` ได้ **ปกติ** ← CORS เป็นกฎของ browser เท่านั้น                                                                                                                                               |

ทำไมเป็นแบบนี้

```
Browser (origin: http://localhost:5173)                 Backend (http://localhost:3000)
  ① OPTIONS /api/v3/users/login  (preflight: "ขอส่ง POST + JSON ได้ไหม?")  ──▶
                                ◀── ตอบกลับ แต่ไม่มี header Access-Control-Allow-Origin
  ② browser บล็อก ไม่ส่ง POST จริง → axios ไม่ได้ response
  ③ api.ts เห็นว่าไม่มี response → throw "เชื่อมต่อ Backend ไม่ได้"
```

> request ที่ส่ง `Content-Type: application/json` หรือมี header `Authorization` จะโดน browser ส่ง **preflight** (`OPTIONS`) ไปถามก่อนเสมอ

### 8.2 หลัง — เปิด CORS กลับ

เอาคอมเมนต์ออกให้เหมือนขั้นที่ 2 แล้ว save

```ts
import cors from "cors";

// ✅ อนุญาตเฉพาะ origin ของ Frontend
app.use(
  cors({
    origin: (process.env.CORS_ORIGIN || "http://localhost:5173").split(","),
  }),
);
```

Login ใหม่อีกครั้ง → สำเร็จ ดูใน **Network** → คลิก request `login` → แท็บ **Headers**

```
Request  (OPTIONS)  Origin: http://localhost:5173
Response (204)      Access-Control-Allow-Origin: http://localhost:5173     ← cors() ใส่ให้
                    Access-Control-Allow-Methods: GET,HEAD,PUT,PATCH,POST,DELETE
Request  (POST)     ส่งจริงตามมา → 200 { success: true, data: { token, ... } }
```

|                     | ก่อน (ไม่มี CORS)                   | หลัง (มี CORS)                                       |
| ------------------- | ----------------------------------- | ---------------------------------------------------- |
| Browser             | ❌ บล็อก — CORS error               | ✅ ผ่าน                                              |
| Insomnia / Postman  | ✅ ผ่าน                             | ✅ ผ่าน                                              |
| Response header     | ไม่มี `Access-Control-Allow-Origin` | `Access-Control-Allow-Origin: http://localhost:5173` |
| Preflight `OPTIONS` | ตอบแต่ไม่มี header อนุญาต           | `cors()` ตอบ 204 พร้อม header อนุญาต                 |

> ลองต่อ: ตั้ง `CORS_ORIGIN=http://localhost:9999` (origin ผิด) แล้วรีสตาร์ต Backend → โดนบล็อกเหมือนเดิม
> แปลว่า `cors()` อนุญาต **เฉพาะ origin ที่ระบุ** ไม่ได้เปิดให้ทุกเว็บ

---

## ขั้นที่ 9 — Frontend: กันหน้าและแยกเมนูตาม Role

### 9.1 Route (`src/main.tsx`) — มีให้แล้ว

```tsx
const router = createBrowserRouter([
  { path: "/login", element: <LoginPage /> }, // อยู่นอก RootLayout → ไม่มี Sidebar
  {
    path: "/",
    element: <RootLayout />, // ต้อง Login ก่อน (ดู 9.2)
    children: [
      { index: true, element: <HomePage /> },
      {
        path: "admin",
        element: <RequireRole role="ADMIN" />, // ADMIN เท่านั้น (ดู 9.3)
        children: [
          { path: "enrollments", element: <AdminEnrollmentsPage /> },
          { path: "courses", element: <AdminCoursesPage /> },
        ],
      },
      {
        path: "student",
        element: <RequireRole role="STUDENT" />, // STUDENT เท่านั้น
        children: [
          { path: "enrollments", element: <StudentEnrollmentsPage /> },
        ],
      },
    ],
  },
]);
```

### 9.2 RootLayout — ตรวจ Login + โหลดข้อมูลครั้งเดียว

📝 `frontend/src/layouts/root-layout.tsx`

1. แก้ import บรรทัดแรก

```ts
import { useEffect } from "react";
import { Navigate, Outlet } from "react-router";
```

2. แทนที่ส่วนบนของ `RootLayout` (ถึง `TODO ขั้นที่ 9.2`) ด้วย

```tsx
const token = useAuthStore((s) => s.token);
const role = useAuthStore((s) => s.role);
const studentId = useAuthStore((s) => s.studentId);
const { loading, error, getAll, reset } = useEnrollmentStore();

// Login แล้ว → โหลดข้อมูลจาก Backend ครั้งเดียว (ตาม role) ทุกหน้าใช้ store ร่วมกัน
// Logout / token หมดอายุ → ล้างข้อมูลของ user ก่อนหน้าทิ้ง
useEffect(() => {
  if (token && role) getAll(role, studentId);
  else reset();
}, [token, role, studentId, getAll, reset]);

// ยังไม่ Login (หรือ token หมดอายุ api.ts ล้างทิ้งแล้ว) → ไปหน้า Login
if (!token) return <Navigate to="/login" replace />;
```

แถบ "กำลังโหลดข้อมูลจาก Backend..." และแถบ error พร้อมปุ่ม **ลองใหม่** มีให้แล้วใน JSX ด้านล่าง

### 9.3 RequireRole

📝 `frontend/src/layouts/require-role.tsx` — **แทนที่ทั้งไฟล์** ด้วย

```tsx
import { Navigate, Outlet } from "react-router";

import { useAuthStore } from "@/lib/auth-store";
import type { User } from "@/lib/types";

/**
 * กันหน้าตาม role — ใช้เป็น element ของ route แม่ (ครอบกลุ่มหน้า /admin หรือ /student)
 * role ไม่ตรง → กลับหน้าแรก (เมนูใน Sidebar ก็ซ่อนหน้าที่ไม่มีสิทธิ์อยู่แล้ว
 * อันนี้กันกรณีพิมพ์ URL ตรงๆ) — Backend ตรวจสิทธิ์ซ้ำอีกชั้นเสมอ
 */
export default function RequireRole({ role }: { role: User["role"] }) {
  const currentRole = useAuthStore((s) => s.role);
  if (currentRole !== role) return <Navigate to="/" replace />;
  return <Outlet />;
}
```

### 9.4 เมนูแยกตาม role (`src/components/app-sidebar.tsx`) — มีให้แล้ว

```ts
const itemsByRole = {
  ADMIN: [
    { title: "หน้าแรก", url: "/", icon: Home },
    { title: "จัดการการลงทะเบียน", url: "/admin/enrollments", icon: BookOpen },
    { title: "จัดการวิชาเรียน", url: "/admin/courses", icon: Library },
  ],
  STUDENT: [
    { title: "หน้าแรก", url: "/", icon: Home },
    {
      title: "จัดการการลงทะเบียน",
      url: "/student/enrollments",
      icon: BookOpen,
    },
  ],
};
```

✅ **ตรวจผล:** Login ได้, รีเฟรชแล้วไม่หลุด, Logout แล้วกลับหน้า Login, แถบแดง "TODO ขั้นที่ 10.1" ขึ้นด้านบน (ทำต่อขั้นถัดไป)

---

## ขั้นที่ 10 — Frontend: Global State (`src/lib/enrollment-store.ts`)

store เดิมของ lecture17 — หน้าต่างๆ ยังเรียก `useEnrollmentStore()` เหมือนเดิม ที่เปลี่ยนคือ **แหล่งข้อมูล**

|                     | lecture17 (ก่อน)          | lecture18 (หลัง)                                       |
| ------------------- | ------------------------- | ------------------------------------------------------ |
| ข้อมูลตั้งต้น       | `mock-data.ts`            | `getAll()` → GET จาก API                               |
| เก็บถาวร            | `persist` ลง localStorage | ไม่ persist — DB คือแหล่งความจริง รีเฟรชก็โหลดใหม่     |
| action เพิ่ม/แก้/ลบ | sync ใส่ state ทันที      | `async`: ส่งไป Backend ก่อน สำเร็จแล้วค่อยอัปเดต state |
| สถานะเพิ่ม          | ไม่มี                     | `loading`, `error`                                     |

### 10.1 `getAll()` — GET หลาย endpoint พร้อมกัน แยกตาม role

📝 `frontend/src/lib/enrollment-store.ts`

1. เพิ่ม import ใต้ `import { create } from "zustand";`

```ts
import { api } from "@/lib/api";
```

2. แทนที่ `TODO ขั้นที่ 10.1: type ApiStudent ...` ด้วย type และฟังก์ชันแปลงข้อมูล

```ts
// รูปแบบข้อมูลที่ Backend ส่งมา (ต่างจาก type ฝั่ง Frontend เล็กน้อย)
type ApiStudent = Omit<Student, "emails"> & { emails?: string[] };
type ApiEnrollment = Enrollment & { createdAt?: string };

// Backend เก็บอีเมลเป็น string[] แต่ฟอร์ม (useFieldArray) ใช้ { address }[]
const fromApiStudent = (s: ApiStudent): Student => ({
  studentId: s.studentId,
  firstName: s.firstName,
  lastName: s.lastName,
  program: s.program,
  interests: s.interests ?? [],
  emails: (s.emails ?? []).map((address) => ({ address })),
});

// เลือกเฉพาะ field ที่ Frontend ใช้ (ตัด id / createdAt / updatedAt ทิ้ง)
const toCourse = ({ courseId, courseTitle, instructors }: Course): Course => ({
  courseId,
  courseTitle,
  instructors,
});

// Backend ใช้ชื่อ createdAt → Frontend ใช้ enrolledAt
const fromApiEnrollment = (e: ApiEnrollment): Enrollment => ({
  studentId: e.studentId,
  courseId: e.courseId,
  enrolledAt: e.createdAt,
});
```

3. แทนที่ `getAll` ด้วย

```ts
  getAll: async (role, studentId) => {
    set({ loading: true, error: null });
    try {
      // STUDENT เรียก GET /students (ทั้งหมด) ไม่ได้ → ดึงแค่ของตัวเอง
      const studentsRequest =
        role === "ADMIN"
          ? api<ApiStudent[]>("/students")
          : studentId
            ? api<ApiStudent>(`/students/${studentId}`).then((s) => [s])
            : Promise.resolve([] as ApiStudent[]);

      // Promise.all = ยิง 3 request พร้อมกัน รอจนครบทุกตัว (เร็วกว่ายิงทีละตัว)
      const [students, courses, enrollments] = await Promise.all([
        studentsRequest,
        api<Course[]>("/courses"),
        api<ApiEnrollment[]>("/enrollments"), // Backend กรองให้ STUDENT เห็นแค่ของตัวเอง
      ]);
      set({
        students: students.map(fromApiStudent),
        courses: courses.map(toCourse),
        enrollments: enrollments.map(fromApiEnrollment),
        loading: false,
      });
    } catch (err) {
      set({ loading: false, error: (err as Error).message }); // → แถบ error ใน RootLayout
    }
  },
```

✅ **ตรวจผล:** Login เป็น ADMIN → หน้า "จัดการวิชาเรียน" / "จัดการการลงทะเบียน" มีข้อมูลจาก DB

### 10.2 รูปแบบ action ที่เขียนข้อมูล (ใช้ซ้ำทุกตัวในขั้นที่ 11–12)

```ts
// 1) ส่งไป Backend  2) ถ้า error → api() throw ออกไปให้ฟอร์มแสดง (state ไม่เปลี่ยน)
// 3) สำเร็จ → เอา "ข้อมูลที่ Backend ตอบกลับ" มาอัปเดต state
someAction: async (input) => {
  const saved = await api<T>("/path", { method: "POST", body: input }); // 1, 2
  set((state) => ({ list: [...state.list, saved] }));                   // 3
},
```

---

## ขั้นที่ 11 — Role STUDENT: จัดการการลงทะเบียน (บาง endpoint)

หน้า `src/pages/student/enrollments.tsx` — นักศึกษาเห็นและลงทะเบียนได้ **เฉพาะของตัวเอง**

| การกระทำ                | Endpoint                   | Backend ทำอะไร                                        |
| ----------------------- | -------------------------- | ----------------------------------------------------- |
| **Get** ดูวิชาที่ลงแล้ว | `GET /api/v3/enrollments`  | STUDENT → กรอง `where: { studentId: user.studentId }` |
| **Get** รายวิชาให้เลือก | `GET /api/v3/courses`      | คืนทุกวิชา                                            |
| **Add** ลงทะเบียน       | `POST /api/v3/enrollments` | ตรวจ zod, กันลงให้คนอื่น (403), กันลงซ้ำ (409)        |

### 11.1 Backend (`backend/src/routes/enrollmentsRouters_v3.ts`) — มีให้แล้ว

```ts
// GET — ADMIN เห็นทั้งหมด, STUDENT เห็นแค่ของตัวเอง
router.get(
  "/",
  authenticateToken,
  checkRoles,
  async (req: CustomRequest, res) => {
    const user = req.user; // payload ของ JWT (ใส่มาโดย authenticateToken)
    const enrollments = await prisma.enrollment.findMany({
      where:
        user?.role === "STUDENT" ? { studentId: user.studentId ?? "" } : {},
      orderBy: { createdAt: "asc" },
    });
    return res.json({ success: true, data: enrollments });
  },
);

// POST — body = { studentId, courseId }
//   ① validate (400)  ② STUDENT ลงให้คนอื่นไม่ได้ (403)
//   ③ นักศึกษา/วิชาต้องมีจริง (404)  ④ ห้ามลงซ้ำ (409)
//   ⑤ prisma.enrollment.create() → 201 { data: created }
```

### 11.2 Store — action `enroll`

📝 `frontend/src/lib/enrollment-store.ts` — แทนที่ `enroll` ด้วย

```ts
  enroll: async (studentId, courseId) => {
    const created = await api<ApiEnrollment>("/enrollments", {
      method: "POST",
      body: { studentId, courseId },
    });
    // Backend บันทึกแล้ว → เพิ่มลง state (createdAt → enrolledAt)
    set((state) => ({
      enrollments: [...state.enrollments, fromApiEnrollment(created)],
    }));
  },
```

> action นี้ใช้ร่วมกับหน้า "จัดการการลงทะเบียน" ของ ADMIN ด้วย (มีให้แล้ว) — เขียนเสร็จ ADMIN ก็ลงทะเบียนให้นักศึกษาได้ทันที

### 11.3 หน้าเว็บ — เรียกใช้และแสดง error

📝 `frontend/src/pages/student/enrollments.tsx`

1. เพิ่ม `enroll` ใน destructuring

```ts
const { students, courses, enrollments, enroll } = useEnrollmentStore();
```

2. แทนที่ `handleEnroll` ด้วย

```ts
// POST /api/v3/enrollments — Backend กันลงทะเบียนซ้ำ (409) อีกชั้น
const handleEnroll = async () => {
  if (!studentId || !formCourse) return;
  setSubmitting(true); // ปิดปุ่มระหว่างรอ กันกดซ้ำ
  setServerError(null);
  try {
    await enroll(studentId, formCourse);
    handleOpenChange(false); // สำเร็จ → ปิด popup (ตารางอัปเดตเองเพราะ state เปลี่ยน)
  } catch (err) {
    setServerError((err as Error).message); // เช่น 409 "has already enrolled" → แสดงใน popup
  } finally {
    setSubmitting(false);
  }
};
```

✅ **ตรวจผล:** Login เป็น STUDENT → กด "ลงทะเบียนเรียน" → เลือกวิชา → ตารางมีวิชาเพิ่ม, รีเฟรชแล้วยังอยู่

---

## ขั้นที่ 12 — Role ADMIN: จัดการวิชาเรียน (ครบทุก endpoint)

หน้า `src/pages/admin/courses.tsx` = ปุ่ม **เพิ่มวิชา** (`CourseFormDialog`) + ตาราง (`CourseTable`)

| การกระทำ   | ปุ่มในหน้า        | Endpoint                 | store action             |
| ---------- | ----------------- | ------------------------ | ------------------------ |
| **Get**    | (โหลดตอนเข้าแอป)  | `GET /api/v3/courses`    | `getAll()` (ขั้นที่ 10)  |
| **Add**    | "+ เพิ่มวิชา"     | `POST /api/v3/courses`   | `addCourse(course)`      |
| **Update** | ดินสอ ✏️ ในตาราง  | `PUT /api/v3/courses`    | `updateCourse(course)`   |
| **Delete** | ถังขยะ 🗑️ ในตาราง | `DELETE /api/v3/courses` | `removeCourse(courseId)` |

ทุก endpoint ผ่าน `authenticateToken` + `checkRoleAdmin` (POST/PUT/DELETE ใช้ได้เฉพาะ ADMIN)

### 12.1 Store — 3 action ของวิชา

📝 `frontend/src/lib/enrollment-store.ts` — แทนที่ `addCourse`, `updateCourse`, `removeCourse` ด้วย

```ts
  // Add — POST /courses, body = { courseId, courseTitle, instructors }
  addCourse: async (course) => {
    const created = await api<Course>("/courses", {
      method: "POST",
      body: course,
    });
    set((state) => ({ courses: [...state.courses, toCourse(created)] })); // ต่อท้าย
  },

  // Update — PUT /courses (courseId แก้ไม่ได้ ใช้หาว่าจะแก้วิชาไหน)
  updateCourse: async (course) => {
    const updated = await api<Course>("/courses", {
      method: "PUT",
      body: course,
    });
    set((state) => ({
      courses: state.courses.map((c) =>
        c.courseId === updated.courseId ? toCourse(updated) : c, // แทนที่ตัวเดิม
      ),
    }));
  },

  // Delete — DELETE /courses, body = { courseId }
  removeCourse: async (courseId) => {
    await api<Course>("/courses", {
      method: "DELETE",
      body: { courseId },
    });
    // Backend ลบ enrollments ของวิชานี้ไปแล้ว — ฝั่งนี้ตัดออกให้ตรงกัน
    set((state) => ({
      courses: state.courses.filter((c) => c.courseId !== courseId),
      enrollments: state.enrollments.filter((e) => e.courseId !== courseId),
    }));
  },
```

### 12.2 ฟอร์ม Add / Update ใช้ component เดียว

📝 `frontend/src/components/courses/course-form-dialog.tsx`

```tsx
// <CourseFormDialog />                 → โหมดเพิ่ม  → POST
// <CourseFormDialog course={course} /> → โหมดแก้ไข → PUT (ช่องรหัสวิชา disabled)
```

1. แทนที่ `TODO ขั้นที่ 12.2: ดึง addCourse / updateCourse` ด้วย

```ts
const addCourse = useEnrollmentStore((s) => s.addCourse);
const updateCourse = useEnrollmentStore((s) => s.updateCourse);
```

2. ใน `handleSubmit` แทนที่ส่วน TODO (หลังบรรทัด `if (Object.keys(nextErrors).length > 0) return;`) ด้วย

```ts
    // ส่งไป Backend (POST หรือ PUT /api/v3/courses) — Backend ตรวจซ้ำ
    // รวมถึงกันชื่อวิชาซ้ำ ซึ่งฟอร์มฝั่งนี้ไม่ได้ตรวจ
    setSubmitting(true);
    setServerError(null);
    try {
      const payload = {
        courseId: values.courseId.trim(),
        courseTitle: values.courseTitle.trim(),
        instructors: values.instructors,
      };
      if (isEdit) await updateCourse(payload); // PUT
      else await addCourse(payload);           // POST
      resetForm();
      setOpen(false);                          // สำเร็จ → ปิด popup
    } catch (err) {
      setServerError((err as Error).message);  // Backend ปฏิเสธ → แสดงในฟอร์ม
    } finally {
      setSubmitting(false);
    }
  };
```

> Backend ตรวจซ้ำด้วย Zod เสมอ และตรวจบางอย่างที่ Frontend ไม่รู้ เช่น **ชื่อวิชาซ้ำ** → 400 `"Course Title or Course ID is already taken."` (POST) / `"Course Title is already taken."` (PUT)

### 12.3 ปุ่มลบในตาราง

📝 `frontend/src/components/courses/course-table.tsx`

1. แทนที่ `TODO ขั้นที่ 12.3: ดึง removeCourse` ด้วย

```ts
const removeCourse = useEnrollmentStore((s) => s.removeCourse);
```

2. แทนที่ `handleDelete` ด้วย

```ts
const handleDelete = async (courseId: string) => {
  setDeleteError(null);
  try {
    await removeCourse(courseId); // DELETE /courses
  } catch (err) {
    setDeleteError((err as Error).message); // แสดง "ลบไม่สำเร็จ: ..." เหนือตาราง
  }
};
```

ปุ่มดินสอ (`<CourseFormDialog course={course} />`) และถังขยะ (`<ConfirmDeleteButton ... />`) ในแต่ละแถวมีให้แล้ว

### 12.4 Backend — PUT / DELETE (`backend/src/routes/coursesRouters_v3.ts`) — มีให้แล้ว

```ts
// PUT — body = { courseId, courseTitle?, instructors? }
router.put("/", authenticateToken, checkRoleAdmin, async (req, res) => {
  const result = zCoursePutBody.safeParse(req.body); // validate
  // ... ไม่พบวิชา → 404, ชื่อวิชาซ้ำกับวิชาอื่น → 400
  const updated = await prisma.course.update({
    where: { courseId },
    data: {
      ...(courseTitle != null && { courseTitle }), // แก้เฉพาะ field ที่ส่งมา
      ...(instructors != null && { instructors }),
    },
  });
  return res.status(200).json({ success: true, data: updated });
});

// DELETE — body = { courseId }
router.delete("/", authenticateToken, checkRoleAdmin, async (req, res) => {
  // ... validate + ไม่พบวิชา → 404
  // $transaction = ทำทั้งคู่สำเร็จ หรือไม่ทำเลย
  // ต้องลบ enrollments ก่อน เพราะอ้างอิง courseId อยู่
  const [, deleted] = await prisma.$transaction([
    prisma.enrollment.deleteMany({ where: { courseId } }),
    prisma.course.delete({ where: { courseId } }),
  ]);
  return res.status(200).json({ success: true, data: deleted });
});
```

---

### รหัสสถานะที่เจอบ่อย

| Status             | ความหมาย                         | Frontend ทำอะไร            |
| ------------------ | -------------------------------- | -------------------------- |
| 200 / 201          | สำเร็จ / สร้างใหม่สำเร็จ         | อัปเดต state               |
| 400                | Validation ไม่ผ่าน / ข้อมูลซ้ำ   | แสดงข้อความในฟอร์ม         |
| 401                | ไม่มี token / Login ไม่ผ่าน      | ล้าง token → หน้า Login    |
| 403                | token หมดอายุ / ไม่มีสิทธิ์      | ล้าง token → หน้า Login    |
| 404                | ไม่พบข้อมูล                      | แสดงข้อความ                |
| 409                | ลงทะเบียนซ้ำ                     | แสดงข้อความ                |
| 0 (ไม่มี response) | Backend ไม่ได้รัน / CORS ไม่ผ่าน | "เชื่อมต่อ Backend ไม่ได้" |

---

## สรุป Endpoint ที่ Frontend ใช้

| หน้า               | Method + Path                     | สิทธิ์                                |
| ------------------ | --------------------------------- | ------------------------------------- |
| Login              | `POST /api/v3/users/login`        | ทุกคน                                 |
| Logout             | `POST /api/v3/users/logout`       | Login แล้ว                            |
| (โหลดข้อมูล)       | `GET /api/v3/students`            | ADMIN                                 |
| (โหลดข้อมูล)       | `GET /api/v3/students/:studentId` | STUDENT (ของตัวเอง)                   |
| จัดการวิชาเรียน    | `GET /api/v3/courses`             | ADMIN / STUDENT                       |
| จัดการวิชาเรียน    | `POST /api/v3/courses`            | ADMIN                                 |
| จัดการวิชาเรียน    | `PUT /api/v3/courses`             | ADMIN                                 |
| จัดการวิชาเรียน    | `DELETE /api/v3/courses`          | ADMIN                                 |
| จัดการการลงทะเบียน | `GET /api/v3/enrollments`         | ADMIN (ทั้งหมด) / STUDENT (ของตัวเอง) |
| จัดการการลงทะเบียน | `POST /api/v3/enrollments`        | ADMIN / STUDENT (ของตัวเอง)           |

