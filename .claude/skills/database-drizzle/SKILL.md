---
name: database-drizzle
description: ใช้เมื่อทำงานกับฐานข้อมูลของ blog-cms ผ่าน Drizzle ORM — ออกแบบ/แก้ table schema (`src/server/db/schema`), สร้างความสัมพันธ์ one-to-many และ many-to-many (เช่น User→Post, Category→Post, Post↔Tag ผ่าน join table), เขียน `relations()`, ตั้งค่า db client, generate/apply migration ด้วย drizzle-kit, เขียน query (findMany/findFirst/with relations/join/count), ทำ unique/index/cascade, หรือ seed ข้อมูล. Triggers — "ออกแบบ schema", "สร้างตาราง", "relation", "one-to-many", "many-to-many", "join table", "foreign key", "migration", "drizzle-kit generate/migrate", "query ด้วย drizzle", "with relations", "index/unique", "cascade", "seed". ข้ามไปใช้ `next-be` ถ้าเป็นการห่อ query ใน repository/use case, ใช้ `security-auth` ถ้าเป็นตาราง user เชิงรหัสผ่าน/auth โดยเฉพาะ.
---

# Database & Relations ด้วย Drizzle ORM (blog-cms)

ค่าเริ่มต้น: **PostgreSQL** (`drizzle-orm/pg-core`) schema อยู่ที่ `src/server/db/schema`, client ที่ `src/server/db/index.ts`

## ภาพรวมความสัมพันธ์ที่ต้องมี

```
users (1) ───< (หลาย) posts        // ผู้เขียนหนึ่งคน มีหลายบทความ
categories (1) ───< (หลาย) posts   // หนึ่งหมวด มีหลายบทความ
posts (หลาย) >───< (หลาย) tags     // ผ่าน join table posts_to_tags
```

Drizzle แยก 2 เรื่อง: **table definition** (โครงตารางจริง/SQL) กับ **`relations()`** (บอก query builder ว่าตารางโยงกันยังไง เพื่อใช้ `with`) ต้องทำทั้งสองอย่าง

## table: users (one ฝั่งของ User→Post)

```ts
// server/db/schema/users.ts
import { pgTable, uuid, varchar, text, pgEnum, timestamp } from "drizzle-orm/pg-core";

export const userRole = pgEnum("user_role", ["ADMIN", "AUTHOR"]);

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  email: varchar("email", { length: 255 }).notNull().unique(),   // unique กันอีเมลซ้ำ
  name: varchar("name", { length: 120 }).notNull(),
  passwordHash: text("password_hash").notNull(),                 // เก็บ hash เท่านั้น (ดู skill security-auth)
  role: userRole("role").notNull().default("AUTHOR"),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});
```

## table: categories + posts (one-to-many)

```ts
// server/db/schema/categories.ts
import { pgTable, uuid, varchar } from "drizzle-orm/pg-core";
export const categories = pgTable("categories", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: varchar("name", { length: 120 }).notNull(),
  slug: varchar("slug", { length: 140 }).notNull().unique(),
});
```

```ts
// server/db/schema/posts.ts
import { pgTable, uuid, varchar, text, pgEnum, timestamp, index } from "drizzle-orm/pg-core";
import { users } from "./users";
import { categories } from "./categories";

export const postStatus = pgEnum("post_status", ["DRAFT", "PUBLISHED"]);

export const posts = pgTable("posts", {
  id: uuid("id").defaultRandom().primaryKey(),
  title: varchar("title", { length: 200 }).notNull(),
  slug: varchar("slug", { length: 220 }).notNull().unique(),
  content: text("content").notNull(),
  status: postStatus("status").notNull().default("DRAFT"),
  // FK -> users (เจ้าของ): ลบ user แล้วบทความหายตาม
  authorId: uuid("author_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  // FK -> categories: ลบหมวดแล้วตั้ง category เป็น null (บทความยังอยู่)
  categoryId: uuid("category_id").references(() => categories.id, { onDelete: "set null" }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  statusIdx: index("posts_status_idx").on(t.status),   // index ช่วย query หน้า public ที่กรอง PUBLISHED
}));
```

## many-to-many: posts ↔ tags ผ่าน join table

```ts
// server/db/schema/tags.ts
import { pgTable, uuid, varchar } from "drizzle-orm/pg-core";
export const tags = pgTable("tags", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: varchar("name", { length: 80 }).notNull().unique(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
});
```

```ts
// server/db/schema/posts-to-tags.ts  — ตารางกลาง
import { pgTable, uuid, primaryKey } from "drizzle-orm/pg-core";
import { posts } from "./posts";
import { tags } from "./tags";

export const postsToTags = pgTable("posts_to_tags", {
  postId: uuid("post_id").notNull().references(() => posts.id, { onDelete: "cascade" }),
  tagId:  uuid("tag_id").notNull().references(() => tags.id,  { onDelete: "cascade" }),
}, (t) => ({
  pk: primaryKey({ columns: [t.postId, t.tagId] }),   // composite PK กันแท็กซ้ำในบทความเดียว
}));
```

## relations() — ให้ query ใช้ `with` ได้

```ts
// server/db/schema/relations.ts
import { relations } from "drizzle-orm";
import { users } from "./users";
import { posts } from "./posts";
import { categories } from "./categories";
import { tags } from "./tags";
import { postsToTags } from "./posts-to-tags";

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const categoriesRelations = relations(categories, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one, many }) => ({
  author:   one(users,      { fields: [posts.authorId],   references: [users.id] }),
  category: one(categories, { fields: [posts.categoryId], references: [categories.id] }),
  tags:     many(postsToTags),
}));

export const tagsRelations = relations(tags, ({ many }) => ({
  posts: many(postsToTags),
}));

export const postsToTagsRelations = relations(postsToTags, ({ one }) => ({
  post: one(posts, { fields: [postsToTags.postId], references: [posts.id] }),
  tag:  one(tags,  { fields: [postsToTags.tagId],  references: [tags.id]  }),
}));
```

## db client + รวม schema

```ts
// server/db/index.ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import { env } from "@/core/config/env";
import * as schema from "./schema";          // re-export ทุกตาราง + relations จาก schema/index.ts

const pool = new Pool({ connectionString: env.DATABASE_URL });
export const db = drizzle(pool, { schema });  // ส่ง schema เข้าไปเพื่อให้ db.query.* + with ใช้ได้
```

```ts
// server/db/schema/index.ts  — รวม export ทั้งหมด
export * from "./users";
export * from "./categories";
export * from "./posts";
export * from "./tags";
export * from "./posts-to-tags";
export * from "./relations";
```

## query ที่ใช้บ่อย

```ts
// ดึงบทความ + ผู้เขียน + หมวด + แท็ก (nested) ด้วย relational query
const post = await db.query.posts.findFirst({
  where: (p, { eq }) => eq(p.slug, slug),
  with: {
    author:   { columns: { id: true, name: true } },   // เลือกเฉพาะ column ปลอดภัย (ไม่เอา passwordHash)
    category: true,
    tags:     { with: { tag: true } },                 // join table -> ดึง tag จริงผ่าน nested with
  },
});

// pagination หน้า public
const items = await db.query.posts.findMany({
  where: (p, { eq }) => eq(p.status, "PUBLISHED"),
  orderBy: (p, { desc }) => desc(p.createdAt),
  limit, offset: (page - 1) * limit,
});
```

> ตอน select ตาราง users **อย่าดึง passwordHash** ออกไปชั้นนอก ระบุ `columns` ที่ปลอดภัยเสมอ

## migration workflow (drizzle-kit)

1. ตั้ง `drizzle.config.ts` ชี้ `schema: "./src/server/db/schema"`, `out: "./src/server/db/migrations"`, dialect `postgresql`, ใช้ `env.DATABASE_URL`
2. `npx drizzle-kit generate` — สร้าง SQL migration จาก schema
3. `npx drizzle-kit migrate` — apply เข้าฐานข้อมูล
4. `npx drizzle-kit studio` — เปิดดู/แก้ข้อมูลผ่าน UI
แก้ schema เมื่อไหร่ ให้ generate + migrate ใหม่เสมอ อย่าแก้ DB ด้วยมือ

## checklist รีวิว schema/query

- [ ] ทุก FK ระบุ `onDelete` ชัดเจน (cascade / set null) ตามความหมายธุรกิจ
- [ ] join table มี composite primary key กันคู่ซ้ำ
- [ ] field ที่ต้องไม่ซ้ำ (email, slug) ใส่ `.unique()`
- [ ] เขียน `relations()` ครบทุกตารางที่ต้องใช้ `with`
- [ ] query ที่แตะ users เลือก `columns` ไม่หลุด passwordHash
- [ ] มี index บน column ที่ใช้กรอง/เรียงบ่อย (เช่น posts.status)
- [ ] แก้ schema แล้ว generate + migrate ใหม่ ไม่แก้ DB ด้วยมือ
