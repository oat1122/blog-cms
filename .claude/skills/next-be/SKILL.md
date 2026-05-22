---
name: next-be
description: ใช้เมื่อเขียนหรือแก้โค้ดฝั่ง backend/server ของ blog-cms ที่ไม่ใช่ตัว route handler เอง — ได้แก่ use case (`features/*/application/use-cases`), repository implementation (`features/*/infrastructure/repositories`), mapper, DTO, Server Action, การประกอบร่าง dependency (composition root), การดึงข้อมูลใน Server Component ผ่าน use case ตรง ๆ (ไม่ผ่าน HTTP), การ validate env, และการ map error ภายใน business logic. Triggers — "เขียน use case", "business logic", "repository implementation", "server action", "ดึงข้อมูลใน server component", "compose dependency", "mapper entity", "DTO", "ออกแบบ flow ฝั่ง server". ข้ามไปใช้ skill `api-foundation` ถ้าเป็น route.ts/HTTP โดยตรง, ใช้ `database-drizzle` ถ้าเป็นเรื่อง schema/relations/SQL, ใช้ `next-fe` ถ้าเป็น UI/component.
---

# Next.js Backend (use case · repository · server-side flow)

ครอบคลุมงาน server ทั้งหมดที่ "อยู่ด้านในกว่า" route handler โฟกัสที่ business logic ที่สะอาดและทดสอบได้

## use case = หน่วยของ business logic

หนึ่ง use case = หนึ่งเจตนาของผู้ใช้ ตั้งชื่อเป็นกริยา+นาม (`CreatePost`, `PublishPost`, `DeletePost`, `ListPublishedPosts`)

หลักการเขียน:
- รับ dependency (repository, service) ผ่าน **constructor** — พึ่งพา **interface** ของ domain เท่านั้น
- เมธอดหลักชื่อ `execute(input)` รับ **plain object (DTO)** ไม่รับ `Request`
- คืน **DTO** ไม่คืน entity ดิบ (กัน field ภายในรั่ว และทำให้ contract กับ client ชัด)
- โยน `AppError`/subclass เมื่อผิดกฎ (ดู skill `api-foundation`) อย่าคืน null เงียบ ๆ

```ts
// features/posts/application/use-cases/publish-post.ts
import type { PostRepository } from "@/features/posts/domain/repositories/post-repository";
import { NotFoundError, ForbiddenError } from "@/core/errors/app-error";

export class PublishPost {
  constructor(private readonly posts: PostRepository) {}

  async execute(input: { postId: string; actor: { id: string; role: "ADMIN" | "AUTHOR" } }) {
    const post = await this.posts.findById(input.postId);
    if (!post) throw new NotFoundError("ไม่พบบทความ");

    // กฎสิทธิ์ "เจ้าของ" เป็น business rule -> อยู่ตรงนี้ ไม่ใช่ใน route
    const isOwner = post.authorId === input.actor.id;
    if (!isOwner && input.actor.role !== "ADMIN") throw new ForbiddenError("แก้บทความของคนอื่นไม่ได้");

    post.publish();                 // กฎการ publish อยู่ใน entity
    await this.posts.save(post);
    return { id: post.id, status: post.status };
  }
}
```

## repository implementation (infrastructure)

implement interface จาก domain ด้วย Drizzle แล้วใช้ **mapper** แปลง row -> entity เสมอ (อย่าให้ Drizzle row รั่วออกไปชั้นใน)

```ts
// features/posts/infrastructure/repositories/drizzle-post-repository.ts
import { eq } from "drizzle-orm";
import { db } from "@/server/db";
import { posts } from "@/server/db/schema";
import type { PostRepository } from "@/features/posts/domain/repositories/post-repository";
import { Post } from "@/features/posts/domain/entities/post";
import { toEntity, toRow } from "../mappers/post-mapper";

export class DrizzlePostRepository implements PostRepository {
  async findById(id: string): Promise<Post | null> {
    const row = await db.query.posts.findFirst({ where: eq(posts.id, id) });
    return row ? toEntity(row) : null;
  }
  async findPublished({ page, limit }: { page: number; limit: number }) {
    const offset = (page - 1) * limit;
    const rows = await db.query.posts.findMany({
      where: eq(posts.status, "PUBLISHED"),
      limit, offset, orderBy: (p, { desc }) => desc(p.createdAt),
    });
    const total = await db.$count(posts, eq(posts.status, "PUBLISHED"));
    return { items: rows.map(toEntity), total };
  }
  async save(post: Post): Promise<void> {
    const row = toRow(post);
    await db.insert(posts).values(row).onConflictDoUpdate({ target: posts.id, set: row });
  }
  async delete(id: string): Promise<void> {
    await db.delete(posts).where(eq(posts.id, id));
  }
}
```

## mapper

```ts
// features/posts/infrastructure/mappers/post-mapper.ts
import { Post } from "@/features/posts/domain/entities/post";
import type { posts } from "@/server/db/schema";
type Row = typeof posts.$inferSelect;

export const toEntity = (r: Row): Post => Post.rehydrate({   // rehydrate = สร้าง entity จากข้อมูลที่มีอยู่แล้ว
  id: r.id, title: r.title, slug: r.slug, content: r.content,
  status: r.status, authorId: r.authorId, categoryId: r.categoryId,
});
export const toRow = (p: Post) => ({
  id: p.id, title: p.title, slug: p.slug, content: p.content,
  status: p.status, authorId: p.authorId, categoryId: p.categoryId,
});
```

> เพิ่ม static `Post.rehydrate(props)` ใน entity สำหรับสร้างจากข้อมูลเดิม (ต่างจาก `create` ที่ใช้ตอนสร้างใหม่และตั้ง default/ตรวจกฎ)

## composition root — จุดประกอบร่าง

ที่เดียวที่ผูก "interface" เข้ากับ "implementation จริง" อยู่ชั้นนอก ไม่ใช่ใน use case

```ts
// features/posts/composition.ts
import { DrizzlePostRepository } from "./infrastructure/repositories/drizzle-post-repository";
import { CreatePost } from "./application/use-cases/create-post";
import { PublishPost } from "./application/use-cases/publish-post";
import { ListPublishedPosts } from "./application/use-cases/list-published-posts";

export function makePostUseCases() {
  const repo = new DrizzlePostRepository();
  return {
    createPost:    new CreatePost(repo),
    publishPost:   new PublishPost(repo),
    listPublished: new ListPublishedPosts(repo),
  };
}
```

route handler และ Server Component เรียก `makePostUseCases()` เพื่อได้ use case พร้อมใช้

## Server Component ดึงข้อมูล — เรียก use case ตรง ไม่ผ่าน HTTP

ใน RSC ไม่ต้อง fetch `/api/...` ตัวเอง เรียก use case ได้เลย (เร็วกว่า, ไม่มี round-trip)

```tsx
// app/(public)/blog/page.tsx  (Server Component)
import { makePostUseCases } from "@/features/posts/composition";

export default async function BlogPage({ searchParams }: { searchParams: { page?: string } }) {
  const page = Number(searchParams.page ?? 1);
  const { listPublished } = makePostUseCases();
  const { items, total } = await listPublished.execute({ page, limit: 10 });
  return <PostList items={items} total={total} page={page} />;
}
```

> ใช้ HTTP route เมื่อ client (Client Component) ต้องเรียกเอง (mutation, infinite scroll) — ดู skill `api-foundation`/`next-fe`

## Server Action (ทางเลือกสำหรับ mutation จากฟอร์ม)

```ts
"use server";
import { revalidatePath } from "next/cache";
import { requireAuth } from "@/server/auth/guards";
import { createPostSchema } from "@/features/posts/presentation/schemas/post-schema";
import { makePostUseCases } from "@/features/posts/composition";

export async function createPostAction(formData: FormData) {
  const session = await requireAuth();
  const input = createPostSchema.parse(Object.fromEntries(formData));
  await makePostUseCases().createPost.execute({ ...input, authorId: session.user.id });
  revalidatePath("/dashboard/posts");
}
```

## env validation (`src/core/config`)

โหลด .env แล้ว validate ด้วย Zod ครั้งเดียว ใช้ตัวที่ validate แล้วทั่วทั้ง backend อย่าอ่าน `process.env` ดิบกระจาย

```ts
// core/config/env.ts
import "dotenv/config";
import { z } from "zod";
const schema = z.object({
  DATABASE_URL: z.string().url(),
  AUTH_SECRET: z.string().min(16),
  AUTH_URL: z.string().url(),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
});
export const env = schema.parse(process.env);  // ล้มเร็วตอน boot ถ้า env ผิด
```

## checklist รีวิวฝั่ง backend

- [ ] use case รับ repository ผ่าน interface (DI) ไม่ import db ตรง
- [ ] business rule/สิทธิ์เจ้าของ อยู่ใน use case หรือ entity ไม่อยู่ใน route/component
- [ ] repo impl ใช้ mapper แปลง row<->entity ไม่ปล่อย Drizzle row หลุดเข้าชั้นใน
- [ ] use case คืน DTO ไม่คืน entity ดิบ และไม่คืน field ลับ (เช่น passwordHash)
- [ ] การประกอบร่างอยู่ใน composition root ที่ชั้นนอก
- [ ] อ่าน env ผ่าน `core/config/env` ที่ validate แล้วเท่านั้น
