# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## アプリ概要

サブスクリプションの支払いを管理するWebアプリケーション。

## 技術スタック

- **Framework:** Next.js (App Router), React 19, TypeScript 5
- **Styling:** Tailwind CSS v4
- **PM** pnpm
- **Formatter:** oxfmt（`.oxfmtrc.json` で設定）

## コマンド

| コマンド            | 用途                                 |
| ------------------- | ------------------------------------ |
| `pnpm dev`          | 開発サーバー起動                     |
| `pnpm build`        | 本番ビルド                           |
| `pnpm lint`         | ESLint 実行                          |
| `pnpm format`       | oxfmt でフォーマット                 |
| `pnpm format:check` | フォーマットチェックのみ（変更なし） |

## React Compiler

React Compiler が有効化済み（`next.config.ts`）。`memo`・`useMemo`・`useCallback` による手動最適化は不要。コンパイラが自動的に行う。

## コードスタイル

- セミコロン必須、ダブルクォート、末尾カンマ（ES5互換箇所のみ）
- インデント: 4スペース、最大行長: 100文字
- パスエイリアス: `@/*` → `src/*`（`tsconfig.json` で設定）

## Git 規約

- **ブランチ命名:** `feature/xxx`, `fix/xxx`, `chore/xxx`
- **コミットメッセージ:** Conventional Commits（`feat:`, `fix:`, `chore:`, `refactor:`, `docs:` など）
