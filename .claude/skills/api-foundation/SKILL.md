---
name: api-foundation
description: Use when creating, modifying, or reviewing any Next.js App Router API route handler (`src/app/api/**/route.ts`) or React Query hook (`src/features/*/presentation/hooks/use*.ts`) in this codebase. Triggers — "สร้าง route ใหม่", "เพิ่ม endpoint", "API handler", "route handler", "เพิ่ม hook", "useQuery", "useMutation", "auth guard", "role check ใน route", "pagination", "validate body", "Zod parse", "response shape", "centralized error", "logger ใน route", or when seeing inline `requireRole` + `try/catch` + `toErrorResponse` patterns that should be replaced by `withApiHandler`. Skip for NextAuth handler (`/api/auth/[...nextauth]`), Server Component data fetching (use case ตรง — ไม่ผ่าน HTTP), pure backend use case/repository, หรือ UI-only changes.
---

# API Foundation (route handlers + React Query hooks)

ชั้น presentation ฝั่ง HTTP ของ blog-cms ทุก route และทุก hook ต้องสม่ำเสมอ: response รูปเดียวกัน, error แปลที่เดียว, auth/validation ทำแบบเดียวกัน

## หลักการ: route handler ต้อง "บาง"

หนึ่ง route ทำแค่ 4 อย่าง แล้วจบ:
1. **validate** input ด้วย Zod (body / query / params)
2. **guard** auth + role
3. **เรียก use case** (business logic อยู่ในนั้น ไม่ใช่ใน route)
4. **คืน response** รูปแบบกลาง

ถ้า route เริ่มมี if/else ของกฎธุรกิจ = ผิด ให้ย้ายเข้า use case (ดู skill `next-be`)

## รูปแบบ response กลาง (`src/core/api`)

ตอบสำเร็จและล้มเหลวด้วยรูปเดียวเสมอ ฝั่ง client จะ parse ได้ง่าย

```ts
// core/api/response.ts
import { NextResponse } from "next/server";

export type ApiOk<T>  = { ok: true; data: T; meta?: Record<string, unknown> };
export type ApiErr    = { ok: false; error: { code: string; message: string; details?: unknown } };

export const ok  = <T>(data: T, meta?: Record<string, unknown>, status = 200) =>
  NextResponse.json<ApiOk<T>>({ ok: true, data, ...(meta ? { meta } : {}) }, { status });

export const fail = (code: string, message: string, status = 400, details?: unknown) =>
  NextResponse.json<ApiErr>({ ok: false, error: { code, message, details } }, { status });
```

## error กลาง + การ map (`src/core/errors`)

โยน `AppError` จาก use case/guard แล้วให้ withApiHandler แปลเป็น HTTP ที่เดียว

```ts
// core/errors/app-error.ts
export class AppError extends Error {
  constructor(public code: string, message: string, public status = 400, public details?: unknown) {
    super(message);
  }
}
export class NotFoundError      extends AppError { constructor(m="ไม่พบข้อมูล")     { super("NOT_FOUND", m, 404); } }
export class UnauthorizedError  extends AppError { constructor(m="ต้องเข้าสู่ระบบ")  { super("UNAUTHORIZED", m, 401); } }
export class ForbiddenError     extends AppError { constructor(m="ไม่มีสิทธิ์")      { super("FORBIDDEN", m, 403); } }
export class ValidationError    extends AppError { constructor(d:unknown)            { super("VALIDATION", "ข้อมูลไม่ถูกต้อง", 422, d); } }
```

## withApiHandler — ตัวห่อกลาง (หัวใจของ skill นี้)

แทนที่แพทเทิร์น inline `requireRole` + `try/catch` + `toErrorResponse` ที่กระจัดกระจาย ด้วยตัวเดียว

```ts
// core/api/with-api-handler.ts
import { ZodError } from "zod";
import { AppError } from "@/core/errors/app-error";
import { fail } from "./response";
import { logger } from "@/core/logger";

type Handler = (req: Request, ctx: { params: Record<string, string> }) => Promise<Response>;

export function withApiHandler(handler: Handler): Handler {
  return async (req, ctx) => {
    try {
      return await handler(req, ctx);
    } catch (err) {
      if (err instanceof ZodError)  return fail("VALIDATION", "ข้อมูลไม่ถูกต้อง", 422, err.flatten());
      if (err instanceof AppError)  return fail(err.code, err.message, err.status, err.details);
      logger.error("unhandled route error", { err });
      return fail("INTERNAL", "เกิดข้อผิดพลาดภายในระบบ", 500);
    }
  };
}
```

## โครงมาตรฐานของ route.ts

```ts
// app/api/posts/route.ts
import { withApiHandler } from "@/core/api/with-api-handler";
import { ok } from "@/core/api/response";
import { requireAuth } from "@/server/auth/guards";          // ดู skill security-auth
import { createPostSchema, listQuerySchema } from "@/features/posts/presentation/schemas/post-schema";
import { makePostUseCases } from "@/features/posts/composition"; // ประกอบ repo+usecase ที่ชั้นนอก

export const GET = withApiHandler(async (req) => {
  const url = new URL(req.url);
  const { page, limit } = listQuerySchema.parse(Object.fromEntries(url.searchParams));
  const { listPublished } = makePostUseCases();
  const result = await listPublished.execute({ page, limit });
  return ok(result.items, { page, limit, total: result.total });
});

export const POST = withApiHandler(async (req) => {
  const session = await requireAuth();                       // โยน UnauthorizedError ถ้าไม่ login
  const body = createPostSchema.parse(await req.json());     // ZodError -> 422 อัตโนมัติ
  const { createPost } = makePostUseCases();
  const created = await createPost.execute({ ...body, authorId: session.user.id });
  return ok(created, undefined, 201);
});
```

## pagination — กติกามาตรฐาน

- query: `?page=1&limit=10` (page เริ่มที่ 1)
- validate + ใส่ default + จำกัดเพดานด้วย Zod:

```ts
// features/posts/presentation/schemas/post-schema.ts
import { z } from "zod";
export const listQuerySchema = z.object({
  page:  z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(50).default(10),
});
```

- ตอบกลับใส่ `meta: { page, limit, total }` เสมอ เพื่อให้ client คำนวณจำนวนหน้าได้

## React Query hooks (`src/features/*/presentation/hooks/use*.ts`)

ฝั่ง client คุย API ผ่าน hook เท่านั้น ไม่ fetch ดิบกระจายใน component

```ts
// features/posts/presentation/hooks/use-posts.ts
"use client";
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import type { ApiOk } from "@/core/api/response";

const postKeys = {
  all: ["posts"] as const,
  list: (p: { page: number; limit: number }) => ["posts", "list", p] as const,
};

async function getJson<T>(url: string): Promise<ApiOk<T>> {
  const res = await fetch(url);
  const json = await res.json();
  if (!json.ok) throw new Error(json.error.message);   // โยนต่อให้ React Query จัดการ error state
  return json;
}

export function usePosts(params: { page: number; limit: number }) {
  return useQuery({
    queryKey: postKeys.list(params),
    queryFn: () => getJson<PostListItem[]>(`/api/posts?page=${params.page}&limit=${params.limit}`),
  });
}

export function useCreatePost() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: async (input: CreatePostInput) => {
      const res = await fetch("/api/posts", { method: "POST", body: JSON.stringify(input) });
      const json = await res.json();
      if (!json.ok) throw new Error(json.error.message);
      return json.data;
    },
    onSuccess: () => qc.invalidateQueries({ queryKey: postKeys.all }),
  });
}
```

แนวทาง hook: ตั้ง **query key เป็น factory** ต่อ feature, โยน error เมื่อ `ok:false`, และ `invalidateQueries` หลัง mutation สำเร็จ

## checklist รีวิว route/hook

- [ ] route ห่อด้วย `withApiHandler` (ไม่มี try/catch เอง)
- [ ] input ทุกชนิด (body/query/params) ผ่าน Zod ก่อนใช้
- [ ] route ที่ต้องล็อกอินเรียก guard ก่อนทำงาน และ role check ตรงตามสิทธิ์
- [ ] ไม่มี business logic ใน route — เรียก use case อย่างเดียว
- [ ] response เป็นรูป `ok()/fail()` และ list มี `meta` ของ pagination
- [ ] hook ใช้ query key factory, จัดการ error, invalidate หลัง mutate
- [ ] ไม่แตะ route `/api/auth/[...nextauth]` ด้วยแพทเทิร์นนี้ (เป็นของ NextAuth)
