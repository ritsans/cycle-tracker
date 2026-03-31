# サブスクリプション管理アプリ 設計書

## コンセプト

個人向けのデジタルサブスクリプション管理アプリ。月額・年額サービスの支出と次回更新日を可視化し、不要な契約の見直しや解約判断をサポートすることで節約につなげます。

## 技術構成

- **フロントエンド:** Next.js 16 (App Router) + React 19 + Tailwind CSS v4
- **バックエンド/DB:** Supabase (PostgreSQL + Auth)
- **認証:** Supabase Auth の Google OAuth
- **パッケージマネージャ:** pnpm

## ユーザー・スコープ

- 個人利用（共有機能は将来対応）
- デジタルサービスのサブスクのみ対象
- 月額・年額の2サイクル対応
- 複数通貨対応（JPY, USD など混在記録）

## データモデル

### subscriptions テーブル

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| id | uuid | PK, default: gen_random_uuid() | 一意識別子 |
| user_id | uuid | NOT NULL, FK → auth.users.id | ユーザーID |
| name | text | NOT NULL | サービス名 (例: "Netflix") |
| amount | numeric | NOT NULL | 金額 (例: 1490) |
| currency | text | NOT NULL | 通貨コード (例: "JPY", "USD") |
| cycle | text | NOT NULL | "monthly" \| "yearly" |
| category | text | NOT NULL | "video" \| "music" \| "game" \| "storage" \| "tool" \| "other" |
| category_custom | text | nullable | category="other" の場合のみ自由入力名 |
| next_billing_date | date | NOT NULL | 次回請求日 |
| created_at | timestamptz | default: now() | 作成日時 |
| updated_at | timestamptz | default: now() | 更新日時 |

### RLS ポリシー

SELECT / INSERT / UPDATE / DELETE すべて: `user_id = auth.uid()`

### 固定カテゴリ

| 値 | 表示名 |
|---|---|
| video | 動画 |
| music | 音楽 |
| game | ゲーム |
| storage | ストレージ |
| tool | ツール |
| other | その他（category_custom の値を表示） |

## 画面構成

### ルーティング

| パス | 画面 | 認証 |
|---|---|---|
| / | ダッシュボード（概要） | 必要 |
| /subscriptions | サブスク一覧（全件・CRUD） | 必要 |
| /login | ログインページ | 不要 |

### ダッシュボード（/）

```
┌─────────────────────────────────────┐
│  ヘッダー (アプリ名 / ログアウト)     │
├─────────────────────────────────────┤
│  月額サマリー                        │
│  ┌───────────┐ ┌───────────┐        │
│  │ JPY 合計   │ │ USD 合計   │       │
│  │ ¥3,480/月  │ │ $25.98/月  │       │
│  └───────────┘ └───────────┘        │
├─────────────────────────────────────┤
│  ⚠ 次回更新が近いサブスク (30日以内)  │
│  ・Netflix  ¥1,490  → 4/5 更新      │
│  ・Spotify  ¥980   → 4/12 更新      │
├─────────────────────────────────────┤
│  カテゴリ別内訳                      │
│  動画      ¥2,470/月  (2件)         │
│  音楽      ¥980/月   (1件)          │
│  ゲーム    $9.99/月   (1件)         │
│  ツール    ¥500/月   (1件)          │
│                                     │
│         [すべてのサブスクを見る →]    │
└─────────────────────────────────────┘
```

- 通貨別の月額合計を表示（年額は÷12で月額換算して合算）
- 次回更新が30日以内のサブスクを直近順で表示
- カテゴリ別の月額内訳と件数を表示

### サブスク一覧（/subscriptions）

- カテゴリごとにグルーピングされた全サブスク表示
- 追加・編集・削除はモーダルから操作
- カテゴリでフィルタリング可能

### ログイン（/login）

- 「Googleでログイン」ボタンのみのシンプルな画面
- 未ログイン時はここにリダイレクト

## 認証フロー

```
未認証ユーザー
  → proxy.ts でチェック
  → /login にリダイレクト
  → 「Googleでログイン」ボタンをクリック
  → Supabase Auth が Google OAuth を処理
  → /auth/callback (Route Handler) でセッション確立
  → / (ダッシュボード) にリダイレクト
```

ログアウト: ヘッダーのボタン → Server Action で signOut → /login にリダイレクト

## 技術方針

- **データ取得:** サーバーコンポーネントで Supabase から直接取得
- **データ変更:** Server Actions で CRUD 処理、完了後 revalidatePath で画面更新
- **認証ミドルウェア:** proxy.ts で未認証ユーザーを /login にリダイレクト
- **Supabase クライアント:** @supabase/ssr を使い Cookie ベースセッション管理
- **バリデーション:** Server Actions 内でサーバー側バリデーション + HTML ネイティブバリデーションで補助
- **日付操作:** Intl.DateTimeFormat と標準 Date で処理（ライブラリ不要）
- **追加パッケージ:** @supabase/supabase-js + @supabase/ssr のみ

## ファイル構成

```
src/
├── app/
│   ├── layout.tsx              -- 共通レイアウト
│   ├── page.tsx                -- ダッシュボード
│   ├── subscriptions/
│   │   └── page.tsx            -- サブスク一覧
│   ├── login/
│   │   └── page.tsx            -- ログインページ
│   └── auth/
│       └── callback/
│           └── route.ts        -- OAuth コールバック処理
├── lib/
│   └── supabase/
│       ├── client.ts           -- ブラウザ用クライアント
│       ├── server.ts           -- サーバー用クライアント
│       └── middleware.ts       -- Supabase Auth ヘルパー
├── components/
│   ├── header.tsx              -- 共通ヘッダー
│   ├── summary-cards.tsx       -- 通貨別サマリー
│   ├── upcoming-renewals.tsx   -- 更新アラート
│   ├── category-breakdown.tsx  -- カテゴリ別内訳
│   ├── subscription-list.tsx   -- サブスク一覧
│   └── subscription-modal.tsx  -- 追加/編集モーダル
├── actions/
│   └── subscriptions.ts        -- Server Actions (CRUD)
└── proxy.ts                    -- 認証ミドルウェア
```
