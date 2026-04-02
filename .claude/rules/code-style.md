---
paths: 
  - src/**/*.tsx
---

## コードスタイル

- Server Componentsをデフォルトにする。"use client"は最小限に
- データ取得はServer Componentで行い、Clientに渡す
- Supabaseクライアントはサーバー側（createServerClient）を使う
