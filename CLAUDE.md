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

- Write\Edit フックにformatを設定している。ファイルが保存されるたびformatが自動発火。手動指示不要。

## ドキュメント

| パス                           | 内容       |
| ------------------------------ | ---------- |
| `docs/plans/design.md`         | 設計書     |
| `docs/plans/implementation.md` | 実装プラン |
| `docs/memo.md`                 | メモ       |

## Git Worktree

worktreeはリポジトリ内ではなく、親ディレクトリに兄弟として作成すること。
命名規則: `{repo}-{branch-name}`

例:
myapp-myprojects/ ← 整理用親ディレクトリ
├── myapp-main ← main メインリポジトリ
├── myapp-feature-auth ← feature/auth リポジトリ / worktree
└── myapp-feature-ui ← feature/ui リポジトリ / worktree

## 行動原則

- 変更の意図は簡潔に伝える(冗長な説明は不要)
- ベストプラクティスに沿った実装を提案する
- トレードオフがある場合は選択肢を提示する
- 動作を証明できるまでタスクを完了とマークしない

## ワークスタイル

- 3ステップ以上のタスクは必ずPlanモードで開始する
- 変更は必要な箇所(Diff)のみ。影響範囲を最小化する
- 途中でうまくいかなくなったら、無理に進めず立ち止まって再計画する
- コードを読まずに書かない

## Superpowers

計画・実装・見直しには必ず Superpowers スキルを使うこと。

| フェーズ | スキル                                                                              |
| -------- | ----------------------------------------------------------------------------------- |
| 計画     | `superpowers:writing-plans`                                                         |
| 実装     | `superpowers:executing-plans` または `superpowers:subagent-driven-development`      |
| 見直し   | `superpowers:requesting-code-review` / `superpowers:verification-before-completion` |

## Supabase

Supabase への操作（テーブル作成・マイグレーション・クエリ実行・プロジェクト情報取得など）は、**Supabase MCP ツール**（`mcp__plugin_supabase_supabase__*`）を使うこと。

## React Compiler

React Compiler が有効化済み（`next.config.ts`）。`memo`・`useMemo`・`useCallback` による手動最適化は不要。コンパイラが自動的に行う。
