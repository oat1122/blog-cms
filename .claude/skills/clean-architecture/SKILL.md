---
name: clean-architecture
description: ใช้เมื่อกำลังตัดสินใจว่าโค้ดชิ้นหนึ่งควรอยู่ "ชั้นไหน" ของโปรเจคนี้ (domain / application / infrastructure / presentation), เมื่อจะสร้าง feature ใหม่หรือ entity/use case/repository ใหม่, เมื่อรีวิวว่ามีการละเมิด Dependency Rule หรือไม่ (เช่น domain import จาก Drizzle/Next, use case import db ตรง ๆ, business logic หลุดไปอยู่ใน route หรือ component), หรือเมื่อสับสนเรื่องทิศทางการ import และการทำ Dependency Injection. Triggers — "ควรวางไฟล์นี้ไว้ที่ไหน", "สร้าง feature ใหม่", "เพิ่ม use case", "เพิ่ม entity", "repository interface", "โครงสร้างถูกไหม", "ผิดหลัก clean architecture ไหม", "DI", "dependency rule". ข้ามได้ถ้าเป็นงาน config ล้วน, งานปรับ CSS/style, หรือคำถามทั่วไปที่ไม่เกี่ยวกับการจัดชั้นโค้ด.
---

# Clean Architecture (blog-cms)

โค้ดเบสนี้ใช้ Clean Architecture แบบ **feature-based / vertical slice** หนึ่ง feature มีครบ 4 ชั้นในตัวเอง

## กฎเหล็กข้อเดียวที่ต้องจำ

> **Dependency Rule:** การ import ชี้ "เข้าด้านใน" เสมอ — ชั้นในห้ามรู้จักชั้นนอก

```
presentation ─┐
              ├─→ application ─→ domain ←─ infrastructure
   (ชั้นนอก)   ┘                 (ในสุด)      (ชั้นนอก)
```

domain ไม่รู้จักใครเลย · application รู้จักแค่ domain · infrastructure + presentation รู้จัก application/domain ได้

## ชั้นไหน เก็บอะไร (ตารางตัดสินใจเร็ว)

| สิ่งที่จะเขียน | ชั้น | โฟลเดอร์ |
|---|---|---|
| รูปร่างข้อมูลธุรกิจ + กฎที่ติดกับข้อมูลนั้น (Post, User) | domain | `features/<f>/domain/entities` |
| ค่าที่ถูกต้องตามกฎ (Slug, Email, Role) | domain | `features/<f>/domain/value-objects` |
| "สัญญา" ของการเก็บ/ดึงข้อมูล (interface) | domain | `features/<f>/domain/repositories` |
| ขั้นตอนทำงานหนึ่งเจตนา (CreatePost) | application | `features/<f>/application/use-cases` |
| รูปร่าง input/output ของ use case | application | `features/<f>/application/dtos` |
| implement repository ด้วย Drizzle | infrastructure | `features/<f>/infrastructure/repositories` |
| แปลง DB row ↔ entity | infrastructure | `features/<f>/infrastructure/mappers` |
| API route, component, hook, Zod schema | presentation | `features/<f>/presentation/*` หรือ `app/` |
| ของกลางที่ทุก feature ใช้ (errors, env, withApiHandler) | core | `src/core/*` |

## คำถามคัดกรองก่อนวางไฟล์

1. โค้ดนี้ **import จาก framework/lib ภายนอกไหม** (next, drizzle, zod, bcrypt)? ถ้าใช่ -> ห้ามอยู่ใน `domain`
2. โค้ดนี้ **รู้เรื่อง HTTP / request / response ไหม**? ถ้าใช่ -> อยู่ `presentation` ไม่ใช่ `application`
3. โค้ดนี้ **รู้ว่าเก็บข้อมูลด้วยอะไร (SQL, table)**? ถ้าใช่ -> อยู่ `infrastructure`
4. โค้ดนี้คือ **กฎธุรกิจที่จริงเสมอไม่ว่าจะใช้เทคโนโลยีอะไร**? -> `domain`

## รูปแบบมาตรฐานของ entity (domain — pure)

```ts
// features/posts/domain/entities/post.ts  — ห้าม import อะไรจากชั้นนอก
export type PostStatus = "DRAFT" | "PUBLISHED";

export class Post {
  private constructor(
    public readonly id: string,
    public title: string,
    public slug: string,
    public content: string,
    public status: PostStatus,
    public readonly authorId: string,
    public categoryId: string | null,
  ) {}

  static create(props: { title: string; slug: string; content: string; authorId: string; categoryId?: string | null }): Post {
    if (props.title.trim().length < 3) throw new Error("title สั้นเกินไป");
    return new Post(crypto.randomUUID(), props.title, props.slug, props.content, "DRAFT", props.authorId, props.categoryId ?? null);
  }

  publish() {
    if (!this.content.trim()) throw new Error("ห้าม publish บทความที่ไม่มีเนื้อหา");
    this.status = "PUBLISHED";
  }
}
```

## repository interface (domain) vs implementation (infrastructure)

```ts
// features/posts/domain/repositories/post-repository.ts  (interface — domain)
import type { Post } from "../entities/post";
export interface PostRepository {
  findById(id: string): Promise<Post | null>;
  findPublished(opts: { page: number; limit: number }): Promise<{ items: Post[]; total: number }>;
  save(post: Post): Promise<void>;
  delete(id: string): Promise<void>;
}
```

```ts
// features/posts/infrastructure/repositories/drizzle-post-repository.ts  (impl — infrastructure)
import type { PostRepository } from "@/features/posts/domain/repositories/post-repository";
// ใช้ drizzle/db/mapper ได้ตรงนี้ เพราะเป็นชั้นนอก
```

## use case + Dependency Injection (application)

use case **รับ repository ผ่าน constructor** (พึ่งพา interface ไม่ใช่ของจริง) — ทำให้ทดสอบได้โดย mock

```ts
// features/posts/application/use-cases/create-post.ts
import type { PostRepository } from "@/features/posts/domain/repositories/post-repository";
import { Post } from "@/features/posts/domain/entities/post";

export class CreatePost {
  constructor(private readonly posts: PostRepository) {}   // <- DI ผ่าน interface

  async execute(input: { title: string; slug: string; content: string; authorId: string; categoryId?: string }) {
    const post = Post.create(input);
    await this.posts.save(post);
    return { id: post.id, slug: post.slug, status: post.status };  // คืน DTO ไม่คืน entity ดิบ
  }
}
```

จุดที่ "ประกอบร่าง" (สร้าง Drizzle repo จริงแล้วยัดเข้า use case) ทำที่ **ชั้นนอก** เท่านั้น เช่นใน route handler หรือ composition root เล็ก ๆ ใน `core`/`server` — ดู skill `api-foundation` และ `next-be`

## สัญญาณว่าผิดหลัก (ให้ทักท้วงทันที)

- `import ... from "drizzle-orm"` หรือ `@/server/db` โผล่ในโฟลเดอร์ `domain/` หรือ `application/`
- `route.ts` มี `if (post.author !== ...)` หรือเงื่อนไขธุรกิจ -> ควรย้ายเข้า use case/entity
- React component คำนวณกฎธุรกิจเอง -> ย้ายเข้า use case
- use case รับ `Request`/`NextRequest` เป็น argument -> ผิด ควรรับ plain object (DTO)
- entity มี method ที่ยิง SQL -> ผิด entity ต้อง pure

## ลำดับสร้าง feature ใหม่ (ก๊อปจาก posts)

domain (entity + repo interface) -> application (use case + dto) -> infrastructure (drizzle repo + mapper) -> presentation (route + hook + schema)
ดูรายละเอียดเชิงปฏิบัติที่ skill `next-be`, `next-fe`, `database-drizzle`, `security-auth`
