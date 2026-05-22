---
name: next-fe
description: ใช้เมื่อเขียนหรือแก้ UI/ฝั่ง client ของ blog-cms — React Server/Client Component (`app/**`, `features/*/presentation/components`), ฟอร์มด้วย shadcn/ui + react-hook-form + Zod resolver, การแสดงผลรายการ/ตาราง/dialog, loading/empty/error state, การต่อข้อมูลผ่าน React Query hook, การจัดการ pagination บนหน้าจอ, และการตัดสินใจ Server vs Client Component. Triggers — "ทำหน้า", "สร้าง component", "ฟอร์ม", "shadcn", "react-hook-form", "validate ฟอร์ม", "ตาราง/list บทความ", "dialog/modal", "loading state", "ต่อ hook กับ UI", "client component", "ปุ่ม submit". ข้ามไปใช้ `api-foundation` ถ้าเป็นตัว route.ts หรือการเขียน hook fetch logic เอง, ใช้ `next-be` ถ้าเป็น use case/server logic, ใช้ `security-auth` ถ้าเป็นการเช็ค session/role เชิงความปลอดภัย.
---

# Next.js Frontend (shadcn/ui · forms · data binding)

ชั้น presentation ฝั่งหน้าจอ เน้น UI ที่ "โง่" (ไม่มี business logic) ต่อกับ backend ผ่าน hook/use case อย่างเป็นระบบ

## Server vs Client Component — ตัดสินใจก่อนเขียน

| ใช้ Server Component (ค่าเริ่มต้น) | ใช้ Client Component (`"use client"`) |
|---|---|
| ดึงข้อมูลมาแสดง (เรียก use case ตรง) | มี state / event (onClick, onChange) |
| หน้า public, หน้า list แบบ static-ish | ฟอร์ม, dialog, ปุ่มที่ mutate |
| ไม่ต้องใช้ hook ของ React | ใช้ React Query / react-hook-form |

> หลัก: เป็น Server Component ให้ได้มากที่สุด ใส่ `"use client"` เฉพาะส่วนที่ต้อง interactive จริง ๆ (เช่นแยกปุ่ม/ฟอร์มออกเป็น component เล็ก)

## ฟอร์ม = shadcn/ui + react-hook-form + Zod (แพทเทิร์นมาตรฐาน)

ใช้ **Zod schema ตัวเดียวกัน** ที่ backend ใช้ validate (อยู่ใน `features/<f>/presentation/schemas`) เพื่อ single source of truth

```tsx
// features/posts/presentation/components/post-form.tsx
"use client";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { createPostSchema } from "../schemas/post-schema";
import { useCreatePost } from "../hooks/use-posts";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Form, FormField, FormItem, FormLabel, FormControl, FormMessage } from "@/components/ui/form";

type FormValues = z.infer<typeof createPostSchema>;

export function PostForm() {
  const form = useForm<FormValues>({
    resolver: zodResolver(createPostSchema),
    defaultValues: { title: "", slug: "", content: "", categoryId: undefined },
  });
  const createPost = useCreatePost();

  const onSubmit = form.handleSubmit((values) =>
    createPost.mutate(values, { onSuccess: () => form.reset() })
  );

  return (
    <Form {...form}>
      <form onSubmit={onSubmit} className="space-y-4">
        <FormField control={form.control} name="title" render={({ field }) => (
          <FormItem>
            <FormLabel>หัวข้อ</FormLabel>
            <FormControl><Input placeholder="ชื่อบทความ" {...field} /></FormControl>
            <FormMessage />
          </FormItem>
        )} />
        {/* slug, content, category ... รูปแบบเดียวกัน */}
        <Button type="submit" disabled={createPost.isPending}>
          {createPost.isPending ? "กำลังบันทึก..." : "บันทึก"}
        </Button>
      </form>
    </Form>
  );
}
```

หลักของฟอร์ม:
- validation มาจาก Zod ผ่าน `zodResolver` ไม่เขียน if เช็คเอง
- ปุ่ม submit disable ตอน `isPending` และโชว์สถานะ
- แสดง error จาก field ด้วย `<FormMessage />`

## แสดงรายการ + 3 state เสมอ (loading / error / empty)

อย่าลืม 3 สถานะนี้ในทุกหน้าที่ดึงข้อมูลฝั่ง client

```tsx
"use client";
import { usePosts } from "@/features/posts/presentation/hooks/use-posts";

export function PostTable({ page }: { page: number }) {
  const { data, isLoading, isError, error } = usePosts({ page, limit: 10 });

  if (isLoading) return <TableSkeleton />;                       // loading
  if (isError)   return <p className="text-destructive">โหลดไม่สำเร็จ: {String(error)}</p>; // error
  if (!data || data.data.length === 0) return <EmptyState />;    // empty

  return (
    <table>{/* render data.data ... ใช้ data.meta สำหรับ pagination */}</table>
  );
}
```

## React Query Provider (ครั้งเดียวที่ root)

```tsx
// providers/query-provider.tsx
"use client";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { useState } from "react";
export function QueryProvider({ children }: { children: React.ReactNode }) {
  const [client] = useState(() => new QueryClient({
    defaultOptions: { queries: { staleTime: 30_000, retry: 1 } },
  }));
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}
```

ครอบไว้ที่ `app/layout.tsx` (หรือ layout ของกลุ่มที่ใช้ client data) แล้วทุก hook จะใช้ client เดียวกัน

## pagination บนหน้าจอ

ใช้ `meta: { page, limit, total }` จาก API คำนวณจำนวนหน้า: `totalPages = Math.ceil(total / limit)`
เปลี่ยนหน้าโดยอัปเดต state/URL (`?page=`) แล้ว hook จะ refetch เพราะ query key เปลี่ยน — สำหรับหน้า public (Server Component) ส่ง `page` ผ่าน searchParams ดู skill `next-be`

## shadcn/ui — ของที่มักต้องเพิ่ม

button, input, textarea, label, form, table, dialog, dropdown-menu, select, badge, toast/sonner
เพิ่มด้วย `npx shadcn@latest add <component>` แล้วไฟล์จะลงที่ `src/components/ui/`

## checklist รีวิวฝั่ง frontend

- [ ] component ไม่มี business logic (กฎอยู่ใน use case ฝั่ง server)
- [ ] เป็น Server Component เว้นแต่ต้อง interactive จึงใส่ `"use client"`
- [ ] ฟอร์มใช้ shadcn + react-hook-form + zodResolver และใช้ Zod schema ร่วมกับ backend
- [ ] หน้าที่ดึงข้อมูล client มีครบ loading / error / empty state
- [ ] mutation ใช้ hook ที่ invalidate query แล้ว ไม่ fetch ดิบใน component
- [ ] ปุ่ม submit จัดการ disabled/pending และแสดงผลลัพธ์ (toast/redirect)
