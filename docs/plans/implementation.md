# サブスクリプション管理アプリ 実装プラン

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 個人向けサブスクリプション管理アプリを構築する。月額/年額の支出可視化、次回更新日アラート、カテゴリ別内訳を提供する。

**Architecture:** Next.js 16 App Router のサーバーコンポーネントで Supabase からデータ取得し、Server Actions で CRUD 処理を行う。認証は Supabase Auth の Google OAuth を使い、proxy.ts（middleware）で保護する。

**Tech Stack:** Next.js 16, React 19, TypeScript 5, Tailwind CSS v4, Supabase (PostgreSQL + Auth), @supabase/ssr

**設計書:** `docs/plans/design.md`

---

## ファイル構成

| 操作   | パス                                    | 責務                                   |
| ------ | --------------------------------------- | -------------------------------------- |
| Create | `src/types/subs.ts`                     | Subscription 型定義・カテゴリ定数      |
| Create | `src/lib/supabase/client.ts`            | ブラウザ用 Supabase クライアント       |
| Create | `src/lib/supabase/server.ts`            | サーバー用 Supabase クライアント       |
| Create | `src/lib/supabase/middleware.ts`        | Supabase Auth ミドルウェアヘルパー     |
| Create | `src/proxy.ts`                          | 認証ミドルウェア（Next.js middleware） |
| Create | `src/app/login/page.tsx`                | ログインページ                         |
| Create | `src/app/auth/callback/route.ts`        | OAuth コールバック Route Handler       |
| Create | `src/components/header.tsx`             | 共通ヘッダー（アプリ名 + ログアウト）  |
| Modify | `src/app/layout.tsx`                    | ヘッダー組み込み・metadata 更新        |
| Create | `src/actions/subscriptions.ts`          | Server Actions（CRUD）                 |
| Create | `src/lib/format.ts`                     | 通貨・日付フォーマットユーティリティ   |
| Create | `src/components/subs-modal.tsx`         | 追加/編集モーダル                      |
| Create | `src/components/subs-list.tsx`          | サブスク一覧（カテゴリグルーピング）   |
| Create | `src/app/subscriptions/page.tsx`        | サブスク一覧ページ                     |
| Create | `src/components/summary-cards.tsx`      | 通貨別月額サマリー                     |
| Create | `src/components/upcoming-renewals.tsx`  | 更新アラート（30日以内）               |
| Create | `src/components/category-breakdown.tsx` | カテゴリ別内訳                         |
| Modify | `src/app/page.tsx`                      | ダッシュボードページ                   |

- subscriptionsは長いため一部ファイル名に略称である`subs`を使用しています。

---

## Task 1: 依存パッケージのインストールと環境変数の準備

**Files:**

- Modify: `package.json`
- Create: `.env.local.example`

- [ ] **Step 1: Supabase パッケージをインストール**

```bash
pnpm add @supabase/supabase-js @supabase/ssr
```

- [ ] **Step 2: 環境変数のテンプレートを作成**

`.env.local.example`:

```
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

- [ ] **Step 3: .gitignore に .env.local が含まれていることを確認**

```bash
grep -q '.env.local' .gitignore && echo "OK" || echo ".env.local" >> .gitignore
```

- [ ] **Step 4: コミット**

```bash
git add package.json pnpm-lock.yaml .env.local.example .gitignore
git commit -m "chore: add supabase dependencies and env template"
```

---

## Task 2: 型定義とカテゴリ定数

**Files:**

- Create: `src/types/subs.ts`

- [ ] **Step 1: Subscription 型とカテゴリ定数を定義**

`src/types/subs.ts`:

```typescript
export type SubscriptionCycle = "monthly" | "yearly";

export type SubscriptionCategory = "video" | "music" | "game" | "storage" | "tool" | "other";

export interface Subscription {
    id: string;
    user_id: string;
    name: string;
    amount: number;
    currency: string;
    cycle: SubscriptionCycle;
    category: SubscriptionCategory;
    category_custom: string | null;
    next_billing_date: string;
    created_at: string;
    updated_at: string;
}

export type SubscriptionInsert = Omit<Subscription, "id" | "user_id" | "created_at" | "updated_at">;

export type SubscriptionUpdate = Partial<SubscriptionInsert>;

export const CATEGORY_LABELS: Record<SubscriptionCategory, string> = {
    video: "動画",
    music: "音楽",
    game: "ゲーム",
    storage: "ストレージ",
    tool: "ツール",
    other: "その他",
};

export const CATEGORIES: SubscriptionCategory[] = [
    "video",
    "music",
    "game",
    "storage",
    "tool",
    "other",
];
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/types/subs.ts
git commit -m "feat: add subscription type definitions and category constants"
```

---

## Task 3: Supabase クライアントセットアップ

**Files:**

- Create: `src/lib/supabase/client.ts`
- Create: `src/lib/supabase/server.ts`
- Create: `src/lib/supabase/middleware.ts`

- [ ] **Step 1: ブラウザ用クライアントを作成**

`src/lib/supabase/client.ts`:

```typescript
import { createBrowserClient } from "@supabase/ssr";

export function createClient() {
    return createBrowserClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL!,
        process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
    );
}
```

- [ ] **Step 2: サーバー用クライアントを作成**

`src/lib/supabase/server.ts`:

```typescript
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function createClient() {
    const cookieStore = await cookies();

    return createServerClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL!,
        process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
        {
            cookies: {
                getAll() {
                    return cookieStore.getAll();
                },
                setAll(cookiesToSet) {
                    for (const { name, value, options } of cookiesToSet) {
                        cookieStore.set(name, value, options);
                    }
                },
            },
        }
    );
}
```

- [ ] **Step 3: ミドルウェアヘルパーを作成**

`src/lib/supabase/middleware.ts`:

```typescript
import { createServerClient } from "@supabase/ssr";
import { NextResponse, type NextRequest } from "next/server";

export async function updateSession(request: NextRequest) {
    let supabaseResponse = NextResponse.next({ request });

    const supabase = createServerClient(
        process.env.NEXT_PUBLIC_SUPABASE_URL!,
        process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
        {
            cookies: {
                getAll() {
                    return request.cookies.getAll();
                },
                setAll(cookiesToSet) {
                    for (const { name, value } of cookiesToSet) {
                        request.cookies.set(name, value);
                    }
                    supabaseResponse = NextResponse.next({ request });
                    for (const { name, value, options } of cookiesToSet) {
                        supabaseResponse.cookies.set(name, value, options);
                    }
                },
            },
        }
    );

    const {
        data: { user },
    } = await supabase.auth.getUser();

    if (
        !user &&
        !request.nextUrl.pathname.startsWith("/login") &&
        !request.nextUrl.pathname.startsWith("/auth")
    ) {
        const url = request.nextUrl.clone();
        url.pathname = "/login";
        return NextResponse.redirect(url);
    }

    return supabaseResponse;
}
```

- [ ] **Step 4: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 5: コミット**

```bash
git add src/lib/supabase/
git commit -m "feat: add supabase client setup (browser, server, middleware)"
```

---

## Task 4: 認証ミドルウェア（proxy.ts）

**Files:**

- Create: `src/proxy.ts`

設計書では `proxy.ts` と命名されているが、Next.js のミドルウェア規約に従い `middleware.ts` として配置する。設計書の認証ミドルウェア仕様をそのまま実装する。

- [ ] **Step 1: middleware.ts を作成**

`src/middleware.ts`:

```typescript
import { type NextRequest } from "next/server";
import { updateSession } from "@/lib/supabase/middleware";

export async function middleware(request: NextRequest) {
    return await updateSession(request);
}

export const config = {
    matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)"],
};
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/middleware.ts
git commit -m "feat: add auth middleware to protect routes"
```

---

## Task 5: ログインページ

**Files:**

- Create: `src/app/login/page.tsx`

- [ ] **Step 1: ログインページを作成**

`src/app/login/page.tsx`:

```tsx
"use client";

import { createClient } from "@/lib/supabase/client";

export default function LoginPage() {
    const handleLogin = async () => {
        const supabase = createClient();
        await supabase.auth.signInWithOAuth({
            provider: "google",
            options: {
                redirectTo: `${window.location.origin}/auth/callback`,
            },
        });
    };

    return (
        <div className="flex min-h-screen items-center justify-center">
            <div className="text-center">
                <h1 className="mb-8 text-2xl font-bold">サブスク管理</h1>
                <button
                    onClick={handleLogin}
                    className="rounded-lg bg-blue-600 px-6 py-3 text-white hover:bg-blue-700"
                >
                    Google でログイン
                </button>
            </div>
        </div>
    );
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/app/login/page.tsx
git commit -m "feat: add login page with Google OAuth button"
```

---

## Task 6: OAuth コールバック Route Handler

**Files:**

- Create: `src/app/auth/callback/route.ts`

- [ ] **Step 1: コールバックハンドラーを作成**

`src/app/auth/callback/route.ts`:

```typescript
import { NextResponse } from "next/server";
import { createClient } from "@/lib/supabase/server";

export async function GET(request: Request) {
    const { searchParams, origin } = new URL(request.url);
    const code = searchParams.get("code");

    if (code) {
        const supabase = await createClient();
        const { error } = await supabase.auth.exchangeCodeForSession(code);
        if (!error) {
            return NextResponse.redirect(origin);
        }
    }

    return NextResponse.redirect(`${origin}/login`);
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/app/auth/callback/route.ts
git commit -m "feat: add OAuth callback route handler"
```

---

## Task 7: ヘッダーコンポーネントとログアウト

**Files:**

- Create: `src/components/header.tsx`
- Create: `src/actions/auth.ts`

- [ ] **Step 1: ログアウト Server Action を作成**

`src/actions/auth.ts`:

```typescript
"use server";

import { redirect } from "next/navigation";
import { createClient } from "@/lib/supabase/server";

export async function signOut() {
    const supabase = await createClient();
    await supabase.auth.signOut();
    redirect("/login");
}
```

- [ ] **Step 2: ヘッダーコンポーネントを作成**

`src/components/header.tsx`:

```tsx
import { signOut } from "@/actions/auth";

export function Header() {
    return (
        <header className="border-b border-gray-200 bg-white px-6 py-4">
            <div className="mx-auto flex max-w-4xl items-center justify-between">
                <h1 className="text-lg font-bold">サブスク管理</h1>
                <form action={signOut}>
                    <button type="submit" className="text-sm text-gray-500 hover:text-gray-700">
                        ログアウト
                    </button>
                </form>
            </div>
        </header>
    );
}
```

- [ ] **Step 3: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 4: コミット**

```bash
git add src/components/header.tsx src/actions/auth.ts
git commit -m "feat: add header component with sign-out action"
```

---

## Task 8: レイアウト更新

**Files:**

- Modify: `src/app/layout.tsx`

- [ ] **Step 1: layout.tsx を更新**

`src/app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import "./globals.css";

export const metadata: Metadata = {
    title: "サブスク管理",
    description: "サブスクリプションの支払いを管理するWebアプリケーション",
};

export default function RootLayout({
    children,
}: Readonly<{
    children: React.ReactNode;
}>) {
    return (
        <html lang="ja">
            <body className="bg-gray-50 text-gray-900">{children}</body>
        </html>
    );
}
```

ヘッダーはダッシュボードとサブスク一覧ページで表示し、ログインページでは非表示にする。各ページで個別にヘッダーを配置する。

- [ ] **Step 2: ビルドチェック**

```bash
pnpm build
```

Expected: ビルド成功

- [ ] **Step 3: コミット**

```bash
git add src/app/layout.tsx
git commit -m "feat: update root layout with app metadata and Japanese locale"
```

---

## Task 9: フォーマットユーティリティ

**Files:**

- Create: `src/lib/format.ts`

- [ ] **Step 1: 通貨・日付フォーマット関数を作成**

`src/lib/format.ts`:

```typescript
import type { Subscription } from "@/types/subs";

export function formatCurrency(amount: number, currency: string): string {
    return new Intl.NumberFormat("ja-JP", {
        style: "currency",
        currency,
        minimumFractionDigits: currency === "JPY" ? 0 : 2,
    }).format(amount);
}

export function formatDate(dateStr: string): string {
    const date = new Date(dateStr);
    return new Intl.DateTimeFormat("ja-JP", {
        month: "long",
        day: "numeric",
    }).format(date);
}

export function toMonthlyAmount(subscription: Subscription): number {
    if (subscription.cycle === "monthly") {
        return subscription.amount;
    }
    return Math.round((subscription.amount / 12) * 100) / 100;
}

export function daysUntil(dateStr: string): number {
    const target = new Date(dateStr);
    const today = new Date();
    today.setHours(0, 0, 0, 0);
    target.setHours(0, 0, 0, 0);
    return Math.ceil((target.getTime() - today.getTime()) / (1000 * 60 * 60 * 24));
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/lib/format.ts
git commit -m "feat: add currency and date formatting utilities"
```

---

## Task 10: Server Actions（CRUD）

**Files:**

- Create: `src/actions/subscriptions.ts`

- [ ] **Step 1: CRUD Server Actions を作成**

`src/actions/subscriptions.ts`:

```typescript
"use server";

import { revalidatePath } from "next/cache";
import { createClient } from "@/lib/supabase/server";
import type { SubscriptionCategory, SubscriptionCycle } from "@/types/subs";

export async function getSubscriptions() {
    const supabase = await createClient();
    const { data, error } = await supabase
        .from("subscriptions")
        .select("*")
        .order("next_billing_date", { ascending: true });

    if (error) {
        throw new Error(error.message);
    }
    return data;
}

export async function createSubscription(formData: FormData) {
    const supabase = await createClient();
    const {
        data: { user },
    } = await supabase.auth.getUser();

    if (!user) {
        throw new Error("認証が必要です");
    }

    const name = formData.get("name") as string;
    const amount = Number(formData.get("amount"));
    const currency = formData.get("currency") as string;
    const cycle = formData.get("cycle") as SubscriptionCycle;
    const category = formData.get("category") as SubscriptionCategory;
    const categoryCustom = formData.get("category_custom") as string | null;
    const nextBillingDate = formData.get("next_billing_date") as string;

    if (!name || !amount || !currency || !cycle || !category || !nextBillingDate) {
        throw new Error("必須項目が入力されていません");
    }

    const { error } = await supabase.from("subscriptions").insert({
        user_id: user.id,
        name,
        amount,
        currency,
        cycle,
        category,
        category_custom: category === "other" ? categoryCustom : null,
        next_billing_date: nextBillingDate,
    });

    if (error) {
        throw new Error(error.message);
    }

    revalidatePath("/");
    revalidatePath("/subscriptions");
}

export async function updateSubscription(id: string, formData: FormData) {
    const supabase = await createClient();

    const name = formData.get("name") as string;
    const amount = Number(formData.get("amount"));
    const currency = formData.get("currency") as string;
    const cycle = formData.get("cycle") as SubscriptionCycle;
    const category = formData.get("category") as SubscriptionCategory;
    const categoryCustom = formData.get("category_custom") as string | null;
    const nextBillingDate = formData.get("next_billing_date") as string;

    const { error } = await supabase
        .from("subscriptions")
        .update({
            name,
            amount,
            currency,
            cycle,
            category,
            category_custom: category === "other" ? categoryCustom : null,
            next_billing_date: nextBillingDate,
        })
        .eq("id", id);

    if (error) {
        throw new Error(error.message);
    }

    revalidatePath("/");
    revalidatePath("/subscriptions");
}

export async function deleteSubscription(id: string) {
    const supabase = await createClient();

    const { error } = await supabase.from("subscriptions").delete().eq("id", id);

    if (error) {
        throw new Error(error.message);
    }

    revalidatePath("/");
    revalidatePath("/subscriptions");
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/actions/subscriptions.ts
git commit -m "feat: add subscription CRUD server actions"
```

---

## Task 11: サブスク追加/編集モーダル

**Files:**

- Create: `src/components/subs-modal.tsx`

- [ ] **Step 1: モーダルコンポーネントを作成**

`src/components/subs-modal.tsx`:

```tsx
"use client";

import { useRef } from "react";
import { createSubscription, updateSubscription } from "@/actions/subscriptions";
import { CATEGORIES, CATEGORY_LABELS, type Subscription } from "@/types/subs";

interface SubscriptionModalProps {
    subscription?: Subscription;
    onClose: () => void;
}

export function SubscriptionModal({ subscription, onClose }: SubscriptionModalProps) {
    const formRef = useRef<HTMLFormElement>(null);
    const isEdit = !!subscription;

    const handleSubmit = async (formData: FormData) => {
        if (isEdit) {
            await updateSubscription(subscription.id, formData);
        } else {
            await createSubscription(formData);
        }
        onClose();
    };

    return (
        <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/50">
            <div className="w-full max-w-md rounded-lg bg-white p-6">
                <h2 className="mb-4 text-lg font-bold">
                    {isEdit ? "サブスクを編集" : "サブスクを追加"}
                </h2>
                <form ref={formRef} action={handleSubmit} className="space-y-4">
                    <div>
                        <label className="mb-1 block text-sm font-medium">サービス名</label>
                        <input
                            name="name"
                            type="text"
                            required
                            defaultValue={subscription?.name}
                            className="w-full rounded border px-3 py-2"
                        />
                    </div>
                    <div className="grid grid-cols-2 gap-4">
                        <div>
                            <label className="mb-1 block text-sm font-medium">金額</label>
                            <input
                                name="amount"
                                type="number"
                                required
                                min="0"
                                step="0.01"
                                defaultValue={subscription?.amount}
                                className="w-full rounded border px-3 py-2"
                            />
                        </div>
                        <div>
                            <label className="mb-1 block text-sm font-medium">通貨</label>
                            <select
                                name="currency"
                                required
                                defaultValue={subscription?.currency ?? "JPY"}
                                className="w-full rounded border px-3 py-2"
                            >
                                <option value="JPY">JPY</option>
                                <option value="USD">USD</option>
                            </select>
                        </div>
                    </div>
                    <div>
                        <label className="mb-1 block text-sm font-medium">請求サイクル</label>
                        <select
                            name="cycle"
                            required
                            defaultValue={subscription?.cycle ?? "monthly"}
                            className="w-full rounded border px-3 py-2"
                        >
                            <option value="monthly">月額</option>
                            <option value="yearly">年額</option>
                        </select>
                    </div>
                    <div>
                        <label className="mb-1 block text-sm font-medium">カテゴリ</label>
                        <select
                            name="category"
                            required
                            defaultValue={subscription?.category ?? "video"}
                            className="w-full rounded border px-3 py-2"
                        >
                            {CATEGORIES.map((cat) => (
                                <option key={cat} value={cat}>
                                    {CATEGORY_LABELS[cat]}
                                </option>
                            ))}
                        </select>
                    </div>
                    <div>
                        <label className="mb-1 block text-sm font-medium">
                            カスタムカテゴリ名（「その他」の場合）
                        </label>
                        <input
                            name="category_custom"
                            type="text"
                            defaultValue={subscription?.category_custom ?? ""}
                            className="w-full rounded border px-3 py-2"
                        />
                    </div>
                    <div>
                        <label className="mb-1 block text-sm font-medium">次回請求日</label>
                        <input
                            name="next_billing_date"
                            type="date"
                            required
                            defaultValue={subscription?.next_billing_date}
                            className="w-full rounded border px-3 py-2"
                        />
                    </div>
                    <div className="flex justify-end gap-2">
                        <button
                            type="button"
                            onClick={onClose}
                            className="rounded px-4 py-2 text-gray-600 hover:bg-gray-100"
                        >
                            キャンセル
                        </button>
                        <button
                            type="submit"
                            className="rounded bg-blue-600 px-4 py-2 text-white hover:bg-blue-700"
                        >
                            {isEdit ? "更新" : "追加"}
                        </button>
                    </div>
                </form>
            </div>
        </div>
    );
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/components/subs-modal.tsx
git commit -m "feat: add subscription add/edit modal component"
```

---

## Task 12: サブスク一覧コンポーネント

**Files:**

- Create: `src/components/subs-list.tsx`

- [ ] **Step 1: 一覧コンポーネントを作成**

`src/components/subs-list.tsx`:

```tsx
"use client";

import { useState } from "react";
import { deleteSubscription } from "@/actions/subscriptions";
import {
    CATEGORIES,
    CATEGORY_LABELS,
    type Subscription,
    type SubscriptionCategory,
} from "@/types/subs";
import { formatCurrency, formatDate } from "@/lib/format";
import { SubscriptionModal } from "@/components/subs-modal";

interface SubscriptionListProps {
    subscriptions: Subscription[];
}

export function SubscriptionList({ subscriptions }: SubscriptionListProps) {
    const [editTarget, setEditTarget] = useState<Subscription | null>(null);
    const [showAdd, setShowAdd] = useState(false);
    const [filterCategory, setFilterCategory] = useState<SubscriptionCategory | "all">("all");

    const filtered =
        filterCategory === "all"
            ? subscriptions
            : subscriptions.filter((s) => s.category === filterCategory);

    const grouped = CATEGORIES.reduce(
        (acc, cat) => {
            const items = filtered.filter((s) => s.category === cat);
            if (items.length > 0) {
                acc.push({ category: cat, items });
            }
            return acc;
        },
        [] as { category: SubscriptionCategory; items: Subscription[] }[]
    );

    const handleDelete = async (id: string) => {
        if (confirm("このサブスクを削除しますか？")) {
            await deleteSubscription(id);
        }
    };

    return (
        <div>
            <div className="mb-6 flex items-center justify-between">
                <div className="flex items-center gap-2">
                    <label className="text-sm font-medium">カテゴリ:</label>
                    <select
                        value={filterCategory}
                        onChange={(e) =>
                            setFilterCategory(e.target.value as SubscriptionCategory | "all")
                        }
                        className="rounded border px-2 py-1 text-sm"
                    >
                        <option value="all">すべて</option>
                        {CATEGORIES.map((cat) => (
                            <option key={cat} value={cat}>
                                {CATEGORY_LABELS[cat]}
                            </option>
                        ))}
                    </select>
                </div>
                <button
                    onClick={() => setShowAdd(true)}
                    className="rounded-lg bg-blue-600 px-4 py-2 text-sm text-white hover:bg-blue-700"
                >
                    追加
                </button>
            </div>

            {grouped.length === 0 ? (
                <p className="py-12 text-center text-gray-500">サブスクが登録されていません</p>
            ) : (
                <div className="space-y-6">
                    {grouped.map(({ category, items }) => (
                        <section key={category}>
                            <h3 className="mb-2 text-sm font-semibold text-gray-500">
                                {CATEGORY_LABELS[category]}
                            </h3>
                            <ul className="divide-y rounded-lg border bg-white">
                                {items.map((sub) => (
                                    <li
                                        key={sub.id}
                                        className="flex items-center justify-between px-4 py-3"
                                    >
                                        <div>
                                            <p className="font-medium">{sub.name}</p>
                                            <p className="text-sm text-gray-500">
                                                {formatCurrency(sub.amount, sub.currency)}/
                                                {sub.cycle === "monthly" ? "月" : "年"}
                                                {" · "}
                                                次回 {formatDate(sub.next_billing_date)}
                                            </p>
                                        </div>
                                        <div className="flex gap-2">
                                            <button
                                                onClick={() => setEditTarget(sub)}
                                                className="text-sm text-blue-600 hover:underline"
                                            >
                                                編集
                                            </button>
                                            <button
                                                onClick={() => handleDelete(sub.id)}
                                                className="text-sm text-red-600 hover:underline"
                                            >
                                                削除
                                            </button>
                                        </div>
                                    </li>
                                ))}
                            </ul>
                        </section>
                    ))}
                </div>
            )}

            {showAdd && <SubscriptionModal onClose={() => setShowAdd(false)} />}
            {editTarget && (
                <SubscriptionModal subscription={editTarget} onClose={() => setEditTarget(null)} />
            )}
        </div>
    );
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/components/subs-list.tsx
git commit -m "feat: add subscription list component with filter and CRUD"
```

---

## Task 13: サブスク一覧ページ

**Files:**

- Create: `src/app/subscriptions/page.tsx`

- [ ] **Step 1: サブスク一覧ページを作成**

`src/app/subscriptions/page.tsx`:

```tsx
import { Header } from "@/components/header";
import { SubscriptionList } from "@/components/subs-list";
import { getSubscriptions } from "@/actions/subscriptions";

export default async function SubscriptionsPage() {
    const subscriptions = await getSubscriptions();

    return (
        <>
            <Header />
            <main className="mx-auto max-w-4xl px-6 py-8">
                <h2 className="mb-6 text-xl font-bold">すべてのサブスク</h2>
                <SubscriptionList subscriptions={subscriptions} />
            </main>
        </>
    );
}
```

- [ ] **Step 2: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 3: コミット**

```bash
git add src/app/subscriptions/page.tsx
git commit -m "feat: add subscriptions list page"
```

---

## Task 14: ダッシュボードコンポーネント群

**Files:**

- Create: `src/components/summary-cards.tsx`
- Create: `src/components/upcoming-renewals.tsx`
- Create: `src/components/category-breakdown.tsx`

- [ ] **Step 1: 通貨別サマリーカードを作成**

`src/components/summary-cards.tsx`:

```tsx
import type { Subscription } from "@/types/subs";
import { formatCurrency, toMonthlyAmount } from "@/lib/format";

interface SummaryCardsProps {
    subscriptions: Subscription[];
}

export function SummaryCards({ subscriptions }: SummaryCardsProps) {
    const totals = new Map<string, number>();

    for (const sub of subscriptions) {
        const monthly = toMonthlyAmount(sub);
        totals.set(sub.currency, (totals.get(sub.currency) ?? 0) + monthly);
    }

    if (totals.size === 0) {
        return <p className="py-4 text-center text-gray-500">サブスクが登録されていません</p>;
    }

    return (
        <div className="grid grid-cols-2 gap-4">
            {[...totals.entries()].map(([currency, total]) => (
                <div key={currency} className="rounded-lg border bg-white p-4">
                    <p className="text-sm text-gray-500">{currency} 合計</p>
                    <p className="text-2xl font-bold">
                        {formatCurrency(total, currency)}
                        <span className="text-sm font-normal text-gray-500">/月</span>
                    </p>
                </div>
            ))}
        </div>
    );
}
```

- [ ] **Step 2: 更新アラートコンポーネントを作成**

`src/components/upcoming-renewals.tsx`:

```tsx
import type { Subscription } from "@/types/subs";
import { formatCurrency, formatDate, daysUntil } from "@/lib/format";

interface UpcomingRenewalsProps {
    subscriptions: Subscription[];
}

export function UpcomingRenewals({ subscriptions }: UpcomingRenewalsProps) {
    const upcoming = subscriptions
        .filter((s) => {
            const days = daysUntil(s.next_billing_date);
            return days >= 0 && days <= 30;
        })
        .sort(
            (a, b) =>
                new Date(a.next_billing_date).getTime() - new Date(b.next_billing_date).getTime()
        );

    if (upcoming.length === 0) {
        return null;
    }

    return (
        <section className="rounded-lg border bg-white p-4">
            <h3 className="mb-3 text-sm font-semibold text-amber-600">
                ⚠ 次回更新が近いサブスク（30日以内）
            </h3>
            <ul className="space-y-2">
                {upcoming.map((sub) => (
                    <li key={sub.id} className="flex items-center justify-between">
                        <span className="font-medium">{sub.name}</span>
                        <span className="text-sm text-gray-500">
                            {formatCurrency(sub.amount, sub.currency)}
                            {" → "}
                            {formatDate(sub.next_billing_date)} 更新
                        </span>
                    </li>
                ))}
            </ul>
        </section>
    );
}
```

- [ ] **Step 3: カテゴリ別内訳コンポーネントを作成**

`src/components/category-breakdown.tsx`:

```tsx
import type { Subscription } from "@/types/subs";
import { CATEGORIES, CATEGORY_LABELS } from "@/types/subs";
import { formatCurrency, toMonthlyAmount } from "@/lib/format";

interface CategoryBreakdownProps {
    subscriptions: Subscription[];
}

interface CategoryGroup {
    label: string;
    currency: string;
    total: number;
    count: number;
}

export function CategoryBreakdown({ subscriptions }: CategoryBreakdownProps) {
    const groups: CategoryGroup[] = [];

    for (const cat of CATEGORIES) {
        const items = subscriptions.filter((s) => s.category === cat);
        if (items.length === 0) continue;

        const byCurrency = new Map<string, number>();
        for (const item of items) {
            const monthly = toMonthlyAmount(item);
            byCurrency.set(item.currency, (byCurrency.get(item.currency) ?? 0) + monthly);
        }

        const label =
            cat === "other"
                ? items
                      .map((i) => i.category_custom ?? "その他")
                      .filter((v, i, a) => a.indexOf(v) === i)
                      .join(", ")
                : CATEGORY_LABELS[cat];

        for (const [currency, total] of byCurrency) {
            groups.push({
                label,
                currency,
                total,
                count: items.filter((i) => i.currency === currency).length,
            });
        }
    }

    if (groups.length === 0) {
        return null;
    }

    return (
        <section className="rounded-lg border bg-white p-4">
            <h3 className="mb-3 text-sm font-semibold text-gray-500">カテゴリ別内訳</h3>
            <ul className="space-y-2">
                {groups.map((group, i) => (
                    <li key={i} className="flex items-center justify-between">
                        <span>{group.label}</span>
                        <span className="text-sm text-gray-500">
                            {formatCurrency(group.total, group.currency)}/月
                            {"  "}({group.count}件)
                        </span>
                    </li>
                ))}
            </ul>
        </section>
    );
}
```

- [ ] **Step 4: 型チェックを実行**

```bash
pnpm exec tsc --noEmit
```

Expected: エラーなし

- [ ] **Step 5: コミット**

```bash
git add src/components/summary-cards.tsx src/components/upcoming-renewals.tsx src/components/category-breakdown.tsx
git commit -m "feat: add dashboard components (summary, renewals, breakdown)"
```

---

## Task 15: ダッシュボードページ

**Files:**

- Modify: `src/app/page.tsx`

- [ ] **Step 1: ダッシュボードページを実装**

`src/app/page.tsx`:

```tsx
import Link from "next/link";
import { Header } from "@/components/header";
import { SummaryCards } from "@/components/summary-cards";
import { UpcomingRenewals } from "@/components/upcoming-renewals";
import { CategoryBreakdown } from "@/components/category-breakdown";
import { getSubscriptions } from "@/actions/subscriptions";

export default async function Home() {
    const subscriptions = await getSubscriptions();

    return (
        <>
            <Header />
            <main className="mx-auto max-w-4xl space-y-6 px-6 py-8">
                <h2 className="text-xl font-bold">月額サマリー</h2>
                <SummaryCards subscriptions={subscriptions} />
                <UpcomingRenewals subscriptions={subscriptions} />
                <CategoryBreakdown subscriptions={subscriptions} />
                <div className="text-center">
                    <Link href="/subscriptions" className="text-blue-600 hover:underline">
                        すべてのサブスクを見る →
                    </Link>
                </div>
            </main>
        </>
    );
}
```

- [ ] **Step 2: ビルドチェック**

```bash
pnpm build
```

Expected: ビルド成功

- [ ] **Step 3: コミット**

```bash
git add src/app/page.tsx
git commit -m "feat: implement dashboard page with summary and renewals"
```

---

## Task 16: Supabase データベースセットアップ（SQL）

**Files:**

- Create: `supabase/schema.sql`

このタスクは Supabase ダッシュボードの SQL Editor で実行する。参照用にファイルとしても保存する。

- [ ] **Step 1: スキーマ SQL を作成**

`supabase/schema.sql`:

```sql
-- subscriptions テーブル
CREATE TABLE subscriptions (
    id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    name text NOT NULL,
    amount numeric NOT NULL,
    currency text NOT NULL,
    cycle text NOT NULL CHECK (cycle IN ('monthly', 'yearly')),
    category text NOT NULL CHECK (category IN ('video', 'music', 'game', 'storage', 'tool', 'other')),
    category_custom text,
    next_billing_date date NOT NULL,
    created_at timestamptz DEFAULT now(),
    updated_at timestamptz DEFAULT now()
);

-- RLS 有効化
ALTER TABLE subscriptions ENABLE ROW LEVEL SECURITY;

-- RLS ポリシー: ユーザーは自分のデータのみアクセス可
CREATE POLICY "Users can view own subscriptions"
    ON subscriptions FOR SELECT
    USING (auth.uid() = user_id);

CREATE POLICY "Users can insert own subscriptions"
    ON subscriptions FOR INSERT
    WITH CHECK (auth.uid() = user_id);

CREATE POLICY "Users can update own subscriptions"
    ON subscriptions FOR UPDATE
    USING (auth.uid() = user_id);

CREATE POLICY "Users can delete own subscriptions"
    ON subscriptions FOR DELETE
    USING (auth.uid() = user_id);

-- updated_at 自動更新トリガー
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = now();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER subscriptions_updated_at
    BEFORE UPDATE ON subscriptions
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at();
```

- [ ] **Step 2: Supabase ダッシュボードで SQL を実行**

1. Supabase ダッシュボード → SQL Editor
2. 上記 SQL をペーストして実行
3. Authentication → Providers → Google を有効化し、Client ID / Secret を設定
4. Authentication → URL Configuration → Redirect URLs に `http://localhost:3000/auth/callback` を追加

- [ ] **Step 3: コミット**

```bash
mkdir -p supabase
git add supabase/schema.sql
git commit -m "docs: add database schema SQL for reference"
```

---

## Task 17: 最終確認

- [ ] **Step 1: .env.local を作成して環境変数を設定**

```bash
cp .env.local.example .env.local
# 実際の Supabase URL と Anon Key を設定する
```

- [ ] **Step 2: フォーマットチェック**

```bash
pnpm format
```

- [ ] **Step 3: リントチェック**

```bash
pnpm lint
```

- [ ] **Step 4: ビルド確認**

```bash
pnpm build
```

Expected: ビルド成功

- [ ] **Step 5: 開発サーバーで動作確認**

```bash
pnpm dev
```

確認項目:

1. `http://localhost:3000` → 未認証なら `/login` にリダイレクト
2. Google ログインが成功し `/` に戻る
3. ダッシュボードにサマリー・更新アラート・カテゴリ内訳が表示
4. `/subscriptions` でサブスクの追加・編集・削除ができる
5. ログアウトで `/login` に戻る

- [ ] **Step 6: フォーマット適用してコミット**

```bash
pnpm format
git add -A
git commit -m "chore: apply formatting and finalize initial implementation"
```
