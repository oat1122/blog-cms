# CLAUDE.md

คู่มือสำหรับ Claude เมื่อทำงานในโค้ดเบส **blog-cms** นี้
อ่านไฟล์นี้ก่อนแก้โค้ดเสมอ และยึดเป็น "กติกาที่ห้ามผิด" ระหว่างการทำงานทุกครั้ง

> โปรเจคนี้เป็น **โจทย์ฝึก** (ดู `README.md`) เจ้าของโปรเจคต้องการ **ลงมือเขียนเอง**
> เมื่อถูกขอให้ช่วย ให้ค่อย ๆ อธิบายเหตุผลของแต่ละขั้น ไม่เทเฉลยทั้งหมดรวดเดียวถ้าไม่ได้ขอ
> ถ้าผู้ใช้ขอแค่ "ตรวจ/รีวิว" อย่าเขียนทับด้วยเฉลย — ชี้จุดและอธิบายหลักการแทน

---

## 1. โปรเจคนี้คืออะไร

ระบบ **Blog + CMS** แยกฝั่ง Public (อ่านบทความ) กับ Dashboard (จัดการเนื้อหา, ต้อง login)
สถาปัตยกรรม **Clean Architecture** แบบ feature-based (vertical slice) เป้าหมายคือแยก business logic ออกจาก framework ให้เด็ดขาด

Entities: `User` (role: ADMIN/AUTHOR), `Post` (DRAFT/PUBLISHED), `Category`, `Tag`
Relations: User→Post และ Category→Post (one-to-many), Post↔Tag (many-to-many ผ่าน join table)

---

## 2. Tech Stack (บังคับ — ห้ามสลับเป็นตัวอื่น)

- **Next.js 16** — App Router เท่านั้น
- **shadcn/ui** — UI components
- **Zod** — validation ทั้ง client และ server
- **Drizzle ORM** (+ drizzle-kit) — database + migration (ค่าเริ่มต้น: PostgreSQL)
- **dotenv** — โหลด env (ต่อด้วย validate ด้วย Zod ที่ `src/core/config`)
- **NextAuth v5 (Auth.js)** — authentication / session
- **bcrypt** — hash password

ภาษา: **TypeScript** (strict) เท่านั้น

---

## 3. กฎเหล็ก Clean Architecture (Dependency Rule)

> **ลูกศรการ import ชี้เข้าด้านในเสมอ ชั้นในห้ามรู้จักชั้นนอก**

ลำดับชั้นจากในสุด → นอกสุด: `domain` → `application` → `infrastructure` / `presentation`

ต้องทำ:
- `domain/` = pure TypeScript เท่านั้น — entity, value object, **repository interface**
- `application/` (use cases) เรียก database **ผ่าน interface ของ domain** เท่านั้น
- `infrastructure/` = implement interface ของ domain ด้วย Drizzle + mapper
- `presentation/` = route handler, component, React Query hook, Zod schema

ห้ามทำ (ถ้าเจอ ให้ทักท้วงและเสนอวิธีแก้):
- ❌ import `next`, `drizzle-orm`, `@/server/db`, `zod`, `bcrypt`, `next-auth` ภายในโฟลเดอร์ `domain/`
- ❌ `import { db }` ตรง ๆ ใน use case (`application/`)
- ❌ ใส่ business logic ใน `route.ts` หรือใน React component
- ❌ เรียก use case / repository ของ feature อื่นข้าม layer แบบมั่ว — สื่อสารผ่าน interface/use case ที่เปิดให้

ทิศทางที่ถูกต้อง: `presentation` → `application` → `domain` ← `infrastructure`
(infrastructure พึ่งพา domain เพื่อไป implement interface ของมัน แต่ domain ไม่รู้จัก infrastructure)

---

## 4. โครงสร้างไดเรกทอรี (ที่ต้องเคารพ)

```
src/
├── app/            Presentation: Next.js App Router
│   ├── (public)/   หน้าเว็บสาธารณะ
│   ├── (auth)/     login / register
│   ├── (dashboard)/ หลังบ้าน (ต้อง login)
│   └── api/        route handlers
├── core/           cross-cutting ที่ทุก feature ใช้ร่วม
│   ├── api/        withApiHandler, response shape, error mapping
│   ├── errors/     AppError + subclasses
│   ├── config/     env (dotenv + Zod)
│   ├── logger/
│   └── result/     (ตัวเลือก) Result<T,E>
├── features/<name>/
│   ├── domain/         entities / repositories(interface) / value-objects
│   ├── application/    use-cases / dtos
│   ├── infrastructure/ repositories(Drizzle impl) / mappers
│   └── presentation/   hooks / components / schemas(Zod)
├── server/
│   ├── db/         schema/ , migrations/ , index.ts (client)
│   └── auth/       NextAuth config
├── components/     ui/ (shadcn) , layout/
├── lib/            utils
└── providers/      React Query provider ฯลฯ
```

วาง feature ใหม่โดยก๊อปแพทเทิร์น 4 ชั้นจาก `features/posts/` (feature อ้างอิงหลัก)

---

## 5. Convention การตั้งชื่อ

- ไฟล์/โฟลเดอร์: `kebab-case` (เช่น `create-post.ts`, `post-repository.ts`)
- Entity / Class / Type: `PascalCase` (เช่น `Post`, `PostRepository`)
- Use case: ตั้งชื่อเป็น "กริยา + นาม" (เช่น `CreatePost`, `PublishPost`, `ListPublishedPosts`)
- Repository interface: `XxxRepository` (ใน domain) / implementation: `DrizzleXxxRepository` (ใน infrastructure)
- React Query hooks: `useXxx` / `useCreateXxx` / `useUpdateXxx` (ใน `presentation/hooks/`)
- Zod schema: `xxxSchema` (เช่น `createPostSchema`) วางใน `presentation/schemas/`
- API route: ไฟล์ `route.ts` ตามตำแหน่ง resource ของ App Router

---

## 6. กฎเฉพาะแต่ละชั้น (สรุปสั้น — รายละเอียดอยู่ใน skills)

**API route (`src/app/api/**/route.ts`)** — ต้องบาง (thin):
1. parse + validate input ด้วย Zod (body / query / params)
2. ตรวจ auth + role (ผ่าน guard ใน `core`/`server/auth`)
3. เรียก use case
4. คืน response รูปแบบกลาง
ห่อด้วย `withApiHandler` จาก `core/api` เพื่อให้ error handling/response สม่ำเสมอ — ดู skill `api-foundation`

**Use case (`application/`)** — รับ dependency (repository) ผ่าน constructor/argument (DI), คืน DTO ไม่คืน entity ดิบ, ไม่รู้จัก HTTP

**Repository** — interface อยู่ใน `domain/repositories`, implementation (Drizzle) อยู่ใน `infrastructure/repositories` + ใช้ mapper แปลง row ↔ entity

**Frontend** — Server Component ดึงข้อมูลผ่าน use case ได้โดยตรง (ไม่ต้องผ่าน HTTP), Client Component ใช้ React Query hooks; form ใช้ shadcn + Zod resolver

---

## 7. ความปลอดภัย (อ่าน skill `security-auth` ก่อนแตะ auth)

- รหัสผ่าน hash ด้วย bcrypt (cost ≥ 10) เสมอ — ห้ามเก็บ/log/return เป็น plain text
- **ห้าม** ส่ง field `password`/`passwordHash` กลับใน API response ใด ๆ
- ทุก route ในกลุ่ม dashboard/admin ต้องผ่าน auth guard ก่อน แล้วจึงตรวจ role
- ตรวจสิทธิ์ "เจ้าของทรัพยากร" สำหรับการแก้/ลบ (AUTHOR แก้ได้เฉพาะบทความตัวเอง; ADMIN แก้ได้ทั้งหมด)
- secret ทั้งหมดมาจาก `.env` → validate ที่ `core/config/env.ts` — ห้าม hardcode
- validate input จากภายนอกทุกจุดด้วย Zod ก่อนนำไปใช้

---

## 8. คำสั่งที่ใช้บ่อย (ปรับตาม package manager จริงของโปรเจค)

> โปรเจคยังเป็น skeleton — เมื่อ init แล้ว คำสั่งโดยทั่วไปคือ:

- dev server: `npm run dev`
- build: `npm run build`
- lint / typecheck: `npm run lint` / `npx tsc --noEmit`
- Drizzle generate migration: `npx drizzle-kit generate`
- Drizzle apply migration: `npx drizzle-kit migrate`
- Drizzle studio (ดูข้อมูล): `npx drizzle-kit studio`

หลังแก้โค้ดที่กระทบหลายไฟล์ ให้รัน typecheck + lint ก่อนถือว่าเสร็จ

---

## 9. มารยาทการทำงานของ Claude ในโปรเจคนี้

1. **เคารพ Dependency Rule เป็นอันดับแรก** — ถ้าสิ่งที่ผู้ใช้ขอจะละเมิดกฎ ให้ทักท้วงพร้อมเสนอทางที่ถูก
2. **ทำทีละชั้นและอธิบายเหตุผล** — เพราะนี่คือโจทย์ฝึก เน้นให้ผู้ใช้เข้าใจ ไม่ใช่แค่ผ่าน
3. **อย่าเพิ่ม dependency นอกเหนือ stack ที่กำหนด** โดยไม่ถามก่อน
4. **แก้ให้ตรงจุด** — อย่ารื้อโครงสร้างที่วางไว้แล้วโดยไม่จำเป็น
5. **ใช้ skills ที่เกี่ยวข้อง** ก่อนเขียนงานเฉพาะทาง (ดูตารางข้อ 10)
6. หลังทำงานเสร็จ ให้ตรวจ typecheck/lint และทวนว่าไม่ละเมิดกฎข้อ 3 และ 7

---

## 10. แผนที่ skills (อยู่ใน `.claude/skills/`)

| Skill | ขอบเขต |
|---|---|
| `clean-architecture` | ตัดสินใจวางโค้ดในชั้นที่ถูก, ตรวจ Dependency Rule, รูปแบบ DI |
| `api-foundation` | API route handler + React Query hook (withApiHandler, response shape, guard, pagination) |
| `next-be` | use case, repository, server action, business logic ฝั่ง server |
| `next-fe` | component, form (shadcn + Zod), data fetching, state ฝั่ง client |
| `database-drizzle` | schema, relations (1-N, N-N), migration, query ด้วย Drizzle |
| `security-auth` | NextAuth, bcrypt, auth guard, role/ownership check, ความปลอดภัย route |

> ความสัมพันธ์ของเอกสาร: **README = สร้างอะไร · skills = สร้างยังไงให้ถูกหลัก · CLAUDE.md (ไฟล์นี้) = กติกาที่ห้ามผิด**
