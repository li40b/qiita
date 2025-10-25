---
title: 健康管理アプリの開発環境を構築してみた（#3 開発環境構築編）
tags:
  - 環境構築
  - 個人開発
  - Next.js
private: false
updated_at: '2025-10-26T02:19:52+09:00'
id: 5f5d8a2c4462d57a56a4
organization_url_name: null
slide: false
ignorePublish: false
---

## 1.はじめに

前回の記事で技術選定についてまとめました。  
今回は、実際に開発環境を構築していきます。

以下の流れで進めます：
1. Next.jsプロジェクトの作成
2. Tailwind CSS の確認
3. shadcn/ui の導入（UI）

## 2. Next.jsプロジェクトの作成

### 前提条件
- Node.js（最新LTS）
- npm

### 手順

```bash
# Next.jsプロジェクトの作成
npx create-next-app@latest
```

対話形式で以下を選択しました：

```bash
✔ What is your project named? … habit-care
✔ Would you like to use the recommended Next.js defaults? › No, customize settings
✔ Would you like to use TypeScript? … Yes
✔ Which linter would you like to use? › Biome
✔ Would you like to use React Compiler? … Yes
✔ Would you like to use Tailwind CSS? … Yes
✔ Would you like your code inside a `src/` directory? … Yes
✔ Would you like to use App Router? (recommended) … Yes
✔ Would you like to use Turbopack? (recommended) … Yes
✔ Would you like to customize the import alias (`@/*` by default)? … No
```

```bash
# プロジェクトディレクトリに移動
cd habit-care

# 開発サーバー起動
npm run dev
```

http://localhost:3000 にアクセスして、Next.jsのデフォルトページが表示されたのでOKです。

## 3. Tailwind CSS の確認

create-next-app で Tailwind を有効化している前提です。以下を確認します。

- tailwind.config.(js|ts) の `content` に `src/app`, `src/components` が含まれている
- `src/app/globals.css` に以下がある  
  `@tailwind base;`  
  `@tailwind components;`  
  `@tailwind utilities;`

問題があれば `npm i -D tailwindcss postcss autoprefixer && npx tailwindcss init -p` で再設定します。

## 4. shadcn/ui の導入

Tailwind を前提としたコンポーネントキットを導入します。

```bash
# 初期化（対話に従って進める）
npx shadcn@latest init
```

よく使うコンポーネントを追加（必要に応じて拡張）:
```bash
npx shadcn@latest add button
npx shadcn@latest add input
npx shadcn@latest add card
npx shadcn@latest add dialog
npx shadcn@latest add dropdown-menu
```

設定チェック（initで自動設定されますが念のため）:
- components.json が生成されている（`tailwind.css` や `tsx`、`aliases` が意図通り）
- tailwind.config.(js|ts) の `plugins` に `tailwindcss-animate` が追加されている
- `src/components/ui/` 以下に追加したコンポーネントが生成されている

動作確認（トップページで Button を表示）:
```tsx
// filepath: src/app/page.tsx
import { Button } from "@/components/ui/button";

export default function Page() {
  return (
    <main className="p-6">
      <h1 className="text-xl font-bold mb-4">Hello, shadcn/ui</h1>
      <Button>Button</Button>
    </main>
  );
}
```

### トラブルシュート
- shadcn init で Tailwind のエラーが出る → Tailwind が有効かを確認（上記「Tailwind の確認」参照）
- `@/components/ui/...` が解決できない → いずれかのコンポーネントを `add` してから参照する
- 型エラーや補完が効かない → エディタを再起動、`npm run dev` を再起動

## まとめ

- Next.js（TypeScript + App Router + Tailwind + Biome）を作成し、起動を確認
- UI ライブラリとして shadcn/ui を導入し、基本コンポーネントを追加

---

次回は「Supabase のローカル環境セットアップ（CLI）と Prisma 初期設定、その他ライブラリ導入」についてまとめます。
