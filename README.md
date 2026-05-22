# Blog + CMS — โจทย์ฝึก Clean Architecture (Next.js 16)

> เอกสารนี้คือ **"โจทย์"** (assignment brief) ไม่ใช่เฉลย
> ไม่มีโค้ดให้ก๊อป — มีแต่คำอธิบายว่าจะสร้างอะไร ทำไม และเกณฑ์ว่าทำได้แล้ว
> หน้าที่ของคุณคือ **ลงมือเขียนโค้ดเอง** โดยมี `CLAUDE.md` + `.claude/skills/` เป็นผู้ช่วย/โค้ชระหว่างทาง

---

## 1. ภาพรวมโปรเจค

สร้างระบบ **Blog + CMS (Content Management System)** หนึ่งตัว ที่แยกเป็น 2 ฝั่งชัดเจน:

- **ฝั่ง Public (ผู้อ่านทั่วไป)** — หน้าเว็บบล็อกที่ใครก็เข้าดูได้ ดูรายการบทความ อ่านบทความ กรองตามหมวดหมู่/แท็ก
- **ฝั่ง Dashboard (หลังบ้าน)** — ต้อง login ก่อน ใช้จัดการเนื้อหา (สร้าง/แก้/ลบ บทความ หมวดหมู่ แท็ก ผู้ใช้) โดยสิทธิ์ขึ้นกับ **role**

หัวใจของโจทย์นี้ **ไม่ใช่ความสวยของ UI** แต่คือ **การวางสถาปัตยกรรมแบบ Clean Architecture** ให้โค้ดแยกชั้นชัดเจน ทดสอบง่าย เปลี่ยน database หรือ framework ได้โดยไม่พังทั้งระบบ

---

## 2. เป้าหมายการเรียนรู้ (Learning Objectives)

ฝึกจบโปรเจคนี้แล้วคุณควรทำสิ่งเหล่านี้ได้:

1. **เข้าใจ Dependency Rule ของ Clean Architecture** — รู้ว่าชั้นในไม่รู้จักชั้นนอก และทำไมถึงสำคัญ
2. **แยก business logic ออกจาก framework** — เขียน use case ที่ไม่รู้จัก Next.js, ไม่รู้จัก HTTP, ไม่รู้จัก Drizzle
3. **ออกแบบฐานข้อมูลแบบมีความสัมพันธ์ (relations)** — one-to-many และ many-to-many ด้วย Drizzle ORM
4. **เขียน API route ที่ปลอดภัยและสม่ำเสมอ** — มี auth guard, role check, validation, error handling แบบรวมศูนย์
5. **ต่อ frontend กับ backend อย่างเป็นระบบ** — ผ่าน React Query hooks + Zod schema ที่ใช้ร่วมกัน
6. **ทำ Authentication/Authorization จริง** — hash password ด้วย bcrypt, จัดการ session ด้วย NextAuth, จำกัดสิทธิ์ตาม role

---

## 3. Tech Stack ที่บังคับใช้

| เครื่องมือ      | เวอร์ชัน/ชนิด   | ใช้ทำอะไร                                              |
| --------------- | --------------- | ------------------------------------------------------ |
| **Next.js**     | 16 (App Router) | Framework หลัก ทั้ง frontend และ API route             |
| **shadcn/ui**   | latest          | ชุด UI component (ปุ่ม, ฟอร์ม, ตาราง, dialog)          |
| **Zod**         | latest          | Schema validation — ใช้ทั้งฝั่ง client และ server      |
| **Drizzle ORM** | latest          | คุยกับฐานข้อมูล + migration                            |
| **dotenv**      | latest          | โหลด environment variables (ต่อด้วย validate ด้วย Zod) |
| **NextAuth**    | v5 (Auth.js)    | ระบบ login / session                                   |
| **bcrypt**      | latest          | hash + เทียบรหัสผ่าน                                   |
| **Tiptap**      | latest          | Rich-text editor สำหรับเขียนเนื้อหา blog               |

> หมายเหตุ: ทั้ง 8 ตัวนี้คือสิ่งที่ต้องใช้ ห้ามสลับเป็นตัวอื่น (เพราะเป็นโจทย์ฝึกเฉพาะ stack นี้) ส่วน database engine เลือก PostgreSQL เป็นค่าเริ่มต้น 

---

## 4. ขอบเขตฟีเจอร์ (Scope: Standard)

### Entities หลัก

- **User** — ผู้ใช้ระบบ มี role (`ADMIN`, `AUTHOR`)
- **Post** — บทความ มีสถานะ `DRAFT` / `PUBLISHED` ผูกกับผู้เขียน 1 คน และหมวดหมู่ 1 หมวด
- **Category** — หมวดหมู่ บทความหนึ่งบทความอยู่ได้หนึ่งหมวด (one-to-many)
- **Tag** — แท็ก บทความหนึ่งบทความติดได้หลายแท็ก และแท็กหนึ่งติดได้หลายบทความ (many-to-many)

### ความสัมพันธ์ (Relations) ที่ต้องมี

- `User (1) → (หลาย) Post` — ผู้เขียนหนึ่งคนมีหลายบทความ
- `Category (1) → (หลาย) Post` — หนึ่งหมวดมีหลายบทความ
- `Post (หลาย) ↔ (หลาย) Tag` — ผ่านตารางกลาง (join table) เช่น `posts_to_tags`

### ฟีเจอร์ฝั่ง Public

- หน้ารวมบทความที่ `PUBLISHED` พร้อม **pagination**
- หน้าอ่านบทความเดี่ยวด้วย **slug** (เช่น `/blog/my-first-post`)
- กรองบทความตามหมวดหมู่ และตามแท็ก

### ฟีเจอร์ฝั่ง Dashboard (ต้อง login)

- **Auth**: สมัครสมาชิก (register), เข้าสู่ระบบ (login), ออกจากระบบ (logout)
- **Posts**: สร้าง / แก้ไข / ลบ / เปลี่ยนสถานะ draft↔published (CRUD)
- **Categories & Tags**: CRUD
- **Users**: ดูรายชื่อ + เปลี่ยน role (เฉพาะ `ADMIN`)

### กฎสิทธิ์ (Authorization) ขั้นต่ำ

| การกระทำ                         | AUTHOR | ADMIN |
| -------------------------------- | ------ | ----- |
| สร้าง/แก้/ลบ **บทความของตัวเอง** | ทำได้  | ทำได้ |
| แก้/ลบ **บทความของคนอื่น**       | ไม่ได้ | ทำได้ |
| จัดการ Category / Tag            | ทำได้  | ทำได้ |
| จัดการ User / เปลี่ยน role       | ไม่ได้ | ทำได้ |

---

## 5. Clean Architecture — แนวคิดหลักที่ต้องเข้าใจก่อนเริ่ม

Clean Architecture แบ่งโค้ดเป็น **วง (layer)** ซ้อนกัน โดยมี **กฎเหล็กข้อเดียว** ที่ห้ามผิด:

> **Dependency Rule:** โค้ดชั้นใน **ห้ามรู้จัก** ชั้นนอก
> ลูกศรของการพึ่งพา (import) ชี้เข้าด้านในเสมอ ไม่เคยชี้ออก

วงจากในสุด → นอกสุด มี 4 ชั้น:

```
        ┌──────────────────────────────────────────┐
        │  Presentation / Infrastructure (ชั้นนอก)   │  ← Next.js, Drizzle, NextAuth, shadcn
        │   ┌──────────────────────────────────┐    │
        │   │   Application (Use Cases)         │    │  ← business workflow
        │   │    ┌────────────────────────┐    │    │
        │   │    │   Domain (Entities)     │    │    │  ← กฎธุรกิจแกนกลาง (บริสุทธิ์ที่สุด)
        │   │    └────────────────────────┘    │    │
        │   └──────────────────────────────────┘    │
        └──────────────────────────────────────────┘
```

**1) Domain (ในสุด)** — "ความจริงทางธุรกิจ"
หน้าตาของ entity (Post, User…), value object, และ **interface ของ repository** (สัญญาว่า "ฉันต้องการเก็บ/ดึงข้อมูลแบบนี้" แต่ไม่สนว่าใครทำ)
ห้าม import อะไรจาก Next.js, Drizzle, Zod, หรือชั้นนอกใด ๆ ทั้งสิ้น

**2) Application** — "ขั้นตอนการทำงาน (use case)"
แต่ละ use case = หนึ่งเจตนาของผู้ใช้ เช่น `CreatePost`, `PublishPost`, `RegisterUser`
เรียกใช้ repository **ผ่าน interface** ของ domain เท่านั้น ไม่รู้ว่าข้างล่างเป็น Drizzle หรือไฟล์ text
ไม่รู้จัก HTTP / request / response

**3) Infrastructure** — "ของจริงที่ทำงานเบื้องหลัง"
ตัว implement ของ repository interface ด้วย Drizzle, การต่อ database, mapper แปลงข้อมูล DB ↔ entity
ชั้นนี้ **พึ่งพา** domain (ไป implement interface ของมัน) แต่ domain ไม่รู้จักชั้นนี้

**4) Presentation** — "หน้าตาที่ผู้ใช้/ระบบภายนอกเห็น"
API route (`route.ts`), React Server/Client Component, React Query hooks, Zod schema สำหรับ validate input
ชั้นนี้รับ request → เรียก use case → คืน response

**ทำไมต้องลำบากแบ่งขนาดนี้?** เพื่อให้:

- ทดสอบ business logic ได้โดยไม่ต้องมี database จริง (mock repository interface)
- เปลี่ยน Drizzle เป็น ORM อื่น หรือเปลี่ยน Next.js เป็นอย่างอื่น โดยแก้แค่ชั้นนอก
- อ่านโค้ดแล้วรู้ทันทีว่า "กฎธุรกิจ" อยู่ตรงไหน ไม่ปนกับโค้ด framework

---

## 6. โครงสร้างโฟลเดอร์ (อธิบายทีละส่วน)

```
blog-cms/
├── README.md                  ← ไฟล์นี้ (โจทย์)
├── CLAUDE.md                  ← คู่มือ/กติกาสำหรับให้ Claude ช่วยเขียนโค้ด
├── .env.example               ← ตัวอย่าง environment variables
├── .gitignore
│
├── .claude/
│   └── skills/                ← คลังความรู้ 6 ตัว (โค้ชเฉพาะทาง)
│       ├── clean-architecture/
│       ├── api-foundation/
│       ├── next-be/
│       ├── next-fe/
│       ├── database-drizzle/
│       └── security-auth/
│
└── src/
    ├── app/                   ← [Presentation] Next.js App Router
    │   ├── (public)/          route group หน้าเว็บสาธารณะ (blog, category)
    │   ├── (auth)/            route group หน้า login / register
    │   ├── (dashboard)/       route group หลังบ้าน (ต้อง login)
    │   └── api/               API route handlers (posts, categories, tags, users, auth)
    │
    ├── core/                  ← โค้ดกลางที่ทุก feature ใช้ร่วมกัน (cross-cutting)
    │   ├── api/               withApiHandler, รูปแบบ response, การ map error
    │   ├── errors/            AppError + ชนิดย่อย (NotFound, Unauthorized, ...)
    │   ├── config/            โหลด + validate env ด้วย Zod
    │   ├── logger/            logger กลาง
    │   ├── result/            (ตัวเลือก) Result<T,E> helper
    │   └── types/             type กลาง
    │
    ├── features/              ← หัวใจของโปรเจค: 1 feature = 1 vertical slice ครบ 4 ชั้น
    │   ├── posts/
    │   │   ├── domain/            entities/ , repositories/ (interface) , value-objects/
    │   │   ├── application/       use-cases/ , dtos/
    │   │   ├── infrastructure/    repositories/ (Drizzle impl) , mappers/
    │   │   └── presentation/      hooks/ (React Query) , components/ , schemas/ (Zod)
    │   ├── categories/        (โครงเดียวกับ posts)
    │   ├── tags/
    │   ├── users/
    │   └── auth/
    │
    ├── server/
    │   ├── db/
    │   │   ├── schema/        ตาราง Drizzle (users, posts, categories, tags, posts_to_tags)
    │   │   ├── migrations/    ไฟล์ migration ที่ drizzle-kit สร้าง
    │   │   └── index.ts       db client (สร้างเอง)
    │   └── auth/              config ของ NextAuth
    │
    ├── components/
    │   ├── ui/                shadcn components
    │   └── layout/            navbar, sidebar ฯลฯ
    │
    ├── lib/                   utility ทั่วไป
    └── providers/             React Query provider ฯลฯ
```

> โฟลเดอร์ตอนนี้ยังว่าง (มี `.gitkeep`) — เป็น **skeleton** ให้คุณเติมโค้ดเอง
> เริ่มจาก feature `posts` ให้ครบทั้ง 4 ชั้นก่อน แล้วค่อยก๊อปแพทเทิร์นไป feature อื่น

---

## 7. กฎเหล็กของโปรเจค (อ่านซ้ำก่อนเขียนทุกครั้ง)

1. **domain ห้าม import จากชั้นนอก** — ไม่มี `next`, `drizzle`, `zod`, `bcrypt` ในโฟลเดอร์ `domain/`
2. **application เรียก database ผ่าน interface เท่านั้น** — ไม่ `import { db }` ตรง ๆ ใน use case
3. **route.ts ต้องบาง (thin)** — แค่ validate input → เรียก use case → คืน response ห้ามมี business logic
4. **business logic อยู่ใน use case เท่านั้น** — ไม่กระจายอยู่ใน component หรือ route
5. **validate input ทุกจุดที่รับข้อมูลจากภายนอก** ด้วย Zod (body, query, params)
6. **ห้ามเก็บรหัสผ่านเป็น plain text** — hash ด้วย bcrypt เสมอ และห้ามส่ง field password กลับใน response
7. **ทุก secret อยู่ใน .env** และต้องผ่านการ validate ที่ `core/config` — ห้าม hardcode

---

## 8. แผนการฝึกแบบทีละขั้น (Milestones)

ทำตามลำดับนี้ จะค่อย ๆ เห็นภาพ Clean Architecture เป็นรูปเป็นร่าง

- **M0 — ตั้งโปรเจค:** init Next.js 16, ติดตั้ง dependency ทั้ง 7 ตัว, ตั้งค่า shadcn, ตั้งค่า env + validate ด้วย Zod
- **M1 — Database:** ออกแบบ schema Drizzle ครบทุกตารางและ relations, สร้าง migration แรก, ต่อ db client ได้
- **M2 — Domain + Application ของ Posts:** เขียน entity `Post`, interface `PostRepository`, และ use case `CreatePost` / `ListPublishedPosts` (ยังไม่ต้องมี DB จริง — mock ได้)
- **M3 — Infrastructure ของ Posts:** implement `PostRepository` ด้วย Drizzle + mapper
- **M4 — API foundation:** สร้าง `withApiHandler`, รูปแบบ response กลาง, error mapping, แล้วทำ route `GET/POST /api/posts`
- **M5 — Auth:** ตั้ง NextAuth + bcrypt, ทำ register/login, สร้าง auth guard และ role check
- **M6 — Presentation:** React Query hooks (`usePosts`, `useCreatePost`) + หน้า Dashboard CRUD ด้วย shadcn + Zod form
- **M7 — ขยายไป feature อื่น:** ก๊อปแพทเทิร์นจาก posts ไป categories, tags, users (รวม many-to-many ของ tags)
- **M8 — Public site:** หน้ารวมบทความ + pagination + หน้าอ่านด้วย slug + กรองตามหมวด/แท็ก

---

## 9. เกณฑ์ความสำเร็จ (Definition of Done)

ถือว่าผ่านโจทย์เมื่อ:

- [ ] โครงสร้างโฟลเดอร์ตรงตามข้อ 6 และไม่มีการละเมิด Dependency Rule (ข้อ 7)
- [ ] ออกแบบ relations ครบ: User→Post, Category→Post (one-to-many) และ Post↔Tag (many-to-many ผ่าน join table)
- [ ] ทุก API route ผ่าน `withApiHandler` เดียวกัน — มี auth guard, validation, error shape เหมือนกันทั้งระบบ
- [ ] register/login ใช้งานได้จริง, รหัสผ่าน hash ด้วย bcrypt, response ไม่เคยมี field password
- [ ] role check ทำงานถูกต้องตามตารางสิทธิ์ในข้อ 4
- [ ] หน้า Public แสดงเฉพาะบทความ `PUBLISHED` พร้อม pagination และเปิดอ่านด้วย slug ได้
- [ ] Dashboard ทำ CRUD ของ Posts/Categories/Tags ได้ครบ ผ่าน React Query hooks
- [ ] business logic ทดสอบได้โดยไม่ต้องมี database จริง (use case รับ repository ผ่าน interface)

---

## 10. เริ่มต้นยังไง (Getting Started)

> ส่วนนี้คือ "คำสั่งติดตั้ง" ไม่ใช่โค้ดแอป — รันเพื่อเตรียมเครื่องมือ

1. สร้างโปรเจค Next.js 16 ทับโครงนี้ (หรือ init แยกแล้วย้าย `src/`, `.claude/`, README, CLAUDE.md เข้ามา)
2. ติดตั้ง dependency: `drizzle-orm`, `drizzle-kit`, `zod`, `dotenv`, `next-auth`, `bcrypt` (+ `@types/bcrypt`)
3. ตั้งค่า shadcn/ui และเพิ่ม component ที่ต้องใช้ (button, input, form, table, dialog, …)
4. คัดลอก `.env.example` เป็น `.env` แล้วใส่ค่าจริง (`DATABASE_URL`, `AUTH_SECRET`, …)
5. ออกแบบ schema → `drizzle-kit generate` → `drizzle-kit migrate`
6. เริ่มไล่ตาม Milestones ในข้อ 8

---

## 11. ผู้ช่วยระหว่างทาง: ใช้ skills ให้เป็น

ในโฟลเดอร์ `.claude/skills/` มีคู่มือเฉพาะทาง 6 ตัว ที่ Claude จะหยิบมาใช้อัตโนมัติเมื่อคุณทำงานที่เกี่ยวข้อง (หรือคุณเรียกดูเองเพื่อทบทวนได้):

| Skill                  | ใช้ตอนไหน                                                            |
| ---------------------- | -------------------------------------------------------------------- |
| **clean-architecture** | ตัดสินใจว่าโค้ดชิ้นนี้ควรอยู่ชั้นไหน, ตรวจการละเมิด Dependency Rule  |
| **api-foundation**     | สร้าง/แก้ API route handler หรือ React Query hook                    |
| **next-be**            | งานฝั่ง backend: use case, repository, server action, business logic |
| **next-fe**            | งานฝั่ง frontend: component, form (shadcn + Zod), data fetching      |
| **database-drizzle**   | ออกแบบ schema, relations, migration, query ด้วย Drizzle              |
| **security-auth**      | NextAuth, bcrypt, auth guard, role check, ความปลอดภัยของ route       |

> วิธีคิด: **README บอกว่าจะสร้างอะไร / skills บอกว่าจะสร้างยังไงให้ถูกหลัก / CLAUDE.md คือกติกาที่ห้ามผิด**

ขอให้สนุกกับการฝึก — โฟกัสที่ "ความสะอาดของการแยกชั้น" มากกว่าความเร็วในการทำเสร็จ
