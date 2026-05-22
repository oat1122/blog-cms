---
name: security-auth
description: ใช้เมื่อทำงานเกี่ยวกับ authentication/authorization และความปลอดภัยของ blog-cms — ตั้งค่า NextAuth v5 (Auth.js) + Credentials provider, hash/verify รหัสผ่านด้วย bcrypt, register/login/logout, สร้าง auth guard (`requireAuth`/`requireRole`) ที่ `src/server/auth`, เช็คสิทธิ์ตาม role (ADMIN/AUTHOR) และสิทธิ์ "เจ้าของทรัพยากร", ป้องกัน route หลังบ้าน/middleware, จัดการ session/JWT callback, และกันข้อมูลลับรั่ว (passwordHash). Triggers — "ทำ login/register", "NextAuth", "Auth.js", "bcrypt hash", "verify password", "auth guard", "requireRole", "role check", "ownership check", "ป้องกัน route", "middleware auth", "session callback", "ห้าม password รั่ว". ข้ามไปใช้ `api-foundation` ถ้าเป็นโครง route ทั่วไป (แต่ guard อ้างอิงที่นี่), ใช้ `database-drizzle` ถ้าเป็นการออกแบบตาราง users เชิง schema.
---

# Security: Auth & Route Protection (blog-cms)

ใช้ NextAuth v5 (Credentials) + bcrypt config อยู่ `src/server/auth`, guard ถูกเรียกจาก route/use case

## หลักความปลอดภัยที่ห้ามผิด

1. รหัสผ่าน **hash ด้วย bcrypt เสมอ** (cost >= 10) — ห้ามเก็บ/log/return เป็น plain text
2. **ห้ามส่ง** `password`/`passwordHash` กลับใน response ใด ๆ — select เฉพาะ column ปลอดภัย
3. ทุก action หลังบ้าน **ตรวจ auth ก่อน แล้วจึงตรวจ role/ownership**
4. secret ทั้งหมดมาจาก `.env` ผ่าน `core/config/env` — ห้าม hardcode
5. validate input ของ login/register ด้วย Zod ก่อนใช้ และตอบ error แบบไม่บอกใบ้ว่า "อีเมลนี้มี/ไม่มี"

## bcrypt — hash & verify

```ts
// features/auth/infrastructure/password-hasher.ts
import bcrypt from "bcrypt";
const COST = 12;
export const hashPassword   = (plain: string) => bcrypt.hash(plain, COST);
export const verifyPassword = (plain: string, hash: string) => bcrypt.compare(plain, hash);
```

> วาง interface `PasswordHasher` ไว้ใน domain ถ้าต้องการให้ use case ทดสอบได้โดยไม่ผูก bcrypt ตรง (DI) — bcrypt จริงอยู่ชั้น infrastructure

## use case: RegisterUser (hash ก่อนเก็บ)

```ts
// features/auth/application/use-cases/register-user.ts
import type { UserRepository } from "@/features/users/domain/repositories/user-repository";
import { AppError } from "@/core/errors/app-error";
import { hashPassword } from "../../infrastructure/password-hasher";

export class RegisterUser {
  constructor(private readonly users: UserRepository) {}
  async execute(input: { email: string; name: string; password: string }) {
    const exists = await this.users.findByEmail(input.email);
    if (exists) throw new AppError("EMAIL_TAKEN", "อีเมลนี้ถูกใช้แล้ว", 409);
    const passwordHash = await hashPassword(input.password);
    const user = await this.users.create({ email: input.email, name: input.name, passwordHash, role: "AUTHOR" });
    return { id: user.id, email: user.email, name: user.name, role: user.role }; // ไม่มี passwordHash
  }
}
```

## NextAuth v5 config (Credentials provider)

```ts
// server/auth/index.ts
import NextAuth from "next-auth";
import Credentials from "next-auth/providers/credentials";
import { env } from "@/core/config/env";
import { loginSchema } from "@/features/auth/presentation/schemas/auth-schema";
import { makeAuthUseCases } from "@/features/auth/composition";

export const { handlers, auth, signIn, signOut } = NextAuth({
  secret: env.AUTH_SECRET,
  session: { strategy: "jwt" },
  pages: { signIn: "/login" },
  providers: [
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (raw) => {
        const creds = loginSchema.safeParse(raw);
        if (!creds.success) return null;
        const user = await makeAuthUseCases().validateLogin.execute(creds.data); // verify ด้วย bcrypt ข้างใน
        return user ? { id: user.id, name: user.name, email: user.email, role: user.role } : null;
      },
    }),
  ],
  callbacks: {
    // ยัด role เข้า token แล้วส่งต่อเข้า session เพื่อใช้ตรวจสิทธิ์
    jwt({ token, user }) { if (user) token.role = (user as any).role; return token; },
    session({ session, token }) { if (session.user) (session.user as any).role = token.role; return session; },
  },
});
```

```ts
// app/api/auth/[...nextauth]/route.ts  — handler ของ NextAuth (อย่าห่อด้วย withApiHandler)
import { handlers } from "@/server/auth";
export const { GET, POST } = handlers;
```

## auth guards (`server/auth/guards.ts`) — หัวใจของ route protection

```ts
import { auth } from "@/server/auth";
import { UnauthorizedError, ForbiddenError } from "@/core/errors/app-error";

type Role = "ADMIN" | "AUTHOR";
export type SessionUser = { id: string; email: string; name: string; role: Role };

export async function requireAuth(): Promise<{ user: SessionUser }> {
  const session = await auth();
  if (!session?.user) throw new UnauthorizedError();           // -> 401 ผ่าน withApiHandler
  return { user: session.user as SessionUser };
}

export async function requireRole(...allowed: Role[]) {
  const { user } = await requireAuth();
  if (!allowed.includes(user.role)) throw new ForbiddenError(); // -> 403
  return { user };
}
```

ใช้ใน route:

```ts
// app/api/users/route.ts  — เฉพาะ ADMIN
export const GET = withApiHandler(async () => {
  await requireRole("ADMIN");
  const users = await makeUserUseCases().listUsers.execute();  // use case select column ปลอดภัย
  return ok(users);
});
```

## ownership check — "เจ้าของทรัพยากร"

สิทธิ์แบบ "AUTHOR แก้ได้เฉพาะของตัวเอง / ADMIN แก้ได้หมด" เป็น **business rule -> อยู่ใน use case** ไม่ใช่ใน route (ดูตัวอย่าง `PublishPost` ใน skill `next-be`) route แค่ส่ง `actor: { id, role }` จาก session เข้าไป

```ts
// pattern ใน use case
const isOwner = resource.authorId === actor.id;
if (!isOwner && actor.role !== "ADMIN") throw new ForbiddenError();
```

## middleware (ทางเลือก) — กันทั้งกลุ่ม route

```ts
// middleware.ts
export { auth as middleware } from "@/server/auth";
export const config = { matcher: ["/dashboard/:path*"] };  // บังคับ login ก่อนเข้าหลังบ้าน
```

> middleware กันแบบหยาบ (login หรือยัง) แต่ **role/ownership ที่ละเอียดยังต้องเช็คใน use case/guard เสมอ** อย่าพึ่ง middleware อย่างเดียว

## ข้อผิดพลาดด้านความปลอดภัยที่พบบ่อย (ให้ทักท้วง)

- เก็บ/return passwordHash หรือ select `*` จากตาราง users -> ระบุ columns ปลอดภัย
- เช็ค role ใน UI อย่างเดียว แต่ route ไม่เช็ค -> ผู้ใช้ยิง API ตรงได้ ต้องเช็คฝั่ง server เสมอ
- ใช้ `==` เทียบรหัสผ่านเอง -> ต้องใช้ `bcrypt.compare`
- error login บอกชัดว่า "ไม่มีอีเมลนี้" vs "รหัสผิด" -> ช่วย attacker เดา ให้ตอบรวม ๆ
- hardcode AUTH_SECRET หรือ commit `.env` -> ต้องมาจาก env ที่ validate แล้ว และ .env อยู่ใน .gitignore

## checklist รีวิว auth/security

- [ ] รหัสผ่าน hash ด้วย bcrypt (cost >= 10) ทั้งตอน register และเทียบด้วย compare ตอน login
- [ ] ไม่มี passwordHash หลุดใน response/log ใด ๆ
- [ ] route หลังบ้านเรียก `requireAuth`/`requireRole` ก่อนทำงาน
- [ ] role/ownership ตรงตามตารางสิทธิ์ใน README และเช็คฝั่ง server (ไม่ใช่แค่ UI)
- [ ] route `/api/auth/[...nextauth]` ใช้ handlers ของ NextAuth ตรง ไม่ห่อ withApiHandler
- [ ] secret มาจาก `core/config/env` และ .env ไม่ถูก commit
