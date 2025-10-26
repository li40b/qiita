---
title: 健康管理アプリの開発環境を構築してみた（#4 認証機能実装編）
tags:
  - Next.js
  - Prisma
  - Supabase
  - NextAuth
  - 個人開発
private: false
updated_at: '2025-10-26T17:30:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## 1.はじめに

前回の記事で開発環境を構築しました。  
今回は、Next.js + Supabase + Prisma + Auth.js (NextAuth v5) を使って認証機能を実装していきます。

以下の流れで進めます：
1. Next.js のバージョン調整
2. 認証ライブラリのインストール
3. Prisma の初期化とスキーマ作成
4. Auth.js の設定
5. ログイン画面の実装
6. シードデータの作成
7. 動作確認

## 2. Next.js のバージョン調整

Auth.js v5 (next-auth@beta) は Next.js 15 までをサポートしているため、Next.js 16 を使っている場合はダウングレードが必要です。

```json
// package.json
{
  "dependencies": {
    "next": "^15.0.0"  // 16.0.0 から変更
  }
}
```

```bash
npm install
```

## 3. 認証ライブラリのインストール

必要なパッケージをインストールします。

```bash
npm i next-auth@beta @auth/prisma-adapter prisma @prisma/client bcrypt zod
npm i -D tsx @types/bcrypt
```

- `next-auth@beta`: Auth.js v5（NextAuth の次期バージョン）
- `@auth/prisma-adapter`: Prisma を Auth.js のアダプターとして使用
- `prisma`, `@prisma/client`: ORM
- `bcrypt`: パスワードのハッシュ化
- `zod`: バリデーション
- `tsx`, `@types/bcrypt`: 開発用

## 4. Prisma の初期化

### 4.1 初期化コマンド

```bash
npx prisma init --datasource-provider postgresql
```

これにより以下が生成されます：
- `prisma/schema.prisma`: データベーススキーマ
- `prisma.config.ts`: Prisma 設定ファイル
- `.env`: 環境変数（DATABASE_URL が追加される）

### 4.2 環境変数の設定

Supabase の接続情報を `.env` に設定します。

```env
# Connect to Supabase via connection pooling (runtime)
DATABASE_URL="postgresql://postgres.<ref>:<password>@aws-0-ap-northeast-1.pooler.supabase.com:6543/postgres?pgbouncer=true&sslmode=require"

# Direct connection to the database. Used for migrations
DIRECT_URL="postgresql://postgres.<ref>:<password>@aws-0-ap-northeast-1.supabase.com:5432/postgres?sslmode=require"

# Auth.js
AUTH_SECRET="your-random-secret-here"
AUTH_URL="http://localhost:3000"

# Google OAuth (任意)
AUTH_GOOGLE_ID="your-google-client-id"
AUTH_GOOGLE_SECRET="your-google-client-secret"

# NextAuth (Auth.js) URL
NEXTAUTH_URL="http://localhost:3000"
```

**重要ポイント**:
- `DATABASE_URL`: プール接続（pooler.supabase.com:6543）を使用（実行時用）
- `DIRECT_URL`: 直接接続（supabase.com:5432）を使用（マイグレーション用）
- 両方に `sslmode=require` を付ける
- `AUTH_SECRET`: `openssl rand -base64 32` などで生成した強固な値を設定

### 4.3 Prisma スキーマの作成

Auth.js 用のモデルを `prisma/schema.prisma` に追加します。

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL")
}

model User {
  id             String    @id @default(uuid()) @db.Uuid
  name           String?
  email          String?   @unique
  emailVerified  DateTime?
  image          String?
  hashedPassword String?
  accounts       Account[]
  sessions       Session[]
  createdAt      DateTime  @default(now())
  updatedAt      DateTime  @updatedAt
}

model Account {
  id                String  @id @default(uuid()) @db.Uuid
  userId            String  @db.Uuid
  type              String
  provider          String
  providerAccountId String
  refresh_token     String?
  access_token      String?
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?
  session_state     String?

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(uuid()) @db.Uuid
  sessionToken String   @unique
  userId       String   @db.Uuid
  expires      DateTime

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
}
```

### 4.4 Prisma Client の生成とマイグレーション

```bash
# Prisma Client を生成
npx prisma generate

# マイグレーションを実行
npx prisma migrate dev --name init
```

### 4.5 Prisma Client のシングルトン作成

開発時の多重インスタンス生成を防ぐため、シングルトンパターンを使います。

```typescript
// src/lib/prisma.ts
import { PrismaClient } from "@prisma/client";

const globalForPrisma = globalThis as unknown as { prisma?: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log:
      process.env.NODE_ENV === "development"
        ? ["query", "error", "warn"]
        : ["error"],
  });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

### 4.6 dotenv の読み込み設定

`prisma.config.ts` で環境変数を読み込むように設定します。

```typescript
// prisma.config.ts
import "dotenv/config";
import { defineConfig, env } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  engine: "classic",
  datasource: {
    url: env("DATABASE_URL"),
    // Optionally use a direct connection string for migrations against Supabase
    // directUrl: env("DIRECT_URL"),
  },
});
```

## 5. Auth.js の設定

### 5.1 型定義の拡張

Auth.js のセッション型にユーザーIDを追加します。

```typescript
// src/types/next-auth.d.ts
import { type DefaultSession } from "next-auth";

declare module "next-auth" {
  interface Session {
    user: {
      id: string;
    } & DefaultSession["user"];
  }
}
```

### 5.2 Auth.js の設定ファイル作成

```typescript
// src/auth.ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import bcrypt from "bcrypt";
import NextAuth, { type NextAuthConfig } from "next-auth";
import Credentials from "next-auth/providers/credentials";
import Google from "next-auth/providers/google";
import { z } from "zod";

import { prisma } from "@/lib/prisma";

export const authConfig = {
  adapter: PrismaAdapter(prisma),
  session: {
    strategy: "jwt", // Credentials プロバイダー使用時は JWT 必須
  },
  providers: [
    Google,
    Credentials({
      name: "Credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" },
      },
      authorize: async (creds) => {
        const parsed = z
          .object({ email: z.string().email(), password: z.string().min(6) })
          .safeParse(creds);
        if (!parsed.success) {
          return null;
        }

        const user = await prisma.user.findUnique({
          where: { email: parsed.data.email },
        });
        if (!user?.hashedPassword) {
          return null;
        }
        const isValid = await bcrypt.compare(
          parsed.data.password,
          user.hashedPassword,
        );
        if (!isValid) {
          return null;
        }
        return user;
      },
    }),
  ],
  callbacks: {
    async jwt({ token, user }) {
      if (user) {
        token.sub = user.id;
      }
      return token;
    },
    async session({ session, token }) {
      if (token.sub && session.user) {
        session.user.id = token.sub;
      }
      return session;
    },
  },
  trustHost: true,
  secret: process.env.AUTH_SECRET,
} satisfies NextAuthConfig;

export const { handlers, auth, signIn, signOut } = NextAuth(authConfig);
```

**重要**: Credentials プロバイダーは `session.strategy: "database"` と互換性がないため、`"jwt"` を使用します。

### 5.3 API ルートの作成

```typescript
// src/app/api/auth/[...nextauth]/route.ts
import { handlers } from "@/auth";

export const { GET, POST } = handlers;
```

## 6. ログイン画面の実装

shadcn/ui のコンポーネントを使ってログイン画面を作成します。

### 6.1 必要なコンポーネントの追加

```bash
npx shadcn@latest add card input label
```

### 6.2 ログインページの作成

```tsx
// src/app/login/page.tsx
"use client";

import { Loader2, Lock, LogIn, Mail } from "lucide-react";
import Link from "next/link";
import { signIn } from "next-auth/react";
import { useState } from "react";

import { Button } from "@/components/ui/button";
import {
  Card,
  CardContent,
  CardDescription,
  CardFooter,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

export default function LoginPage() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  async function onSubmit(e: React.FormEvent<HTMLFormElement>) {
    e.preventDefault();
    setLoading(true);
    setError(null);
    const formData = new FormData(e.currentTarget);
    const email = String(formData.get("email") || "");
    const password = String(formData.get("password") || "");
    const res = await signIn("credentials", {
      email,
      password,
      redirect: false,
      callbackUrl: "/",
    });
    setLoading(false);
    if (res?.ok) {
      window.location.href = res.url ?? "/";
    } else {
      setError("メールまたはパスワードが正しくありません");
    }
  }

  return (
    <main className="min-h-dvh grid place-items-center p-6">
      <Card className="w-full max-w-[420px]">
        <CardHeader>
          <CardTitle className="flex items-center gap-2">
            <LogIn className="size-5" />
            ログイン
          </CardTitle>
          <CardDescription>
            アカウントにサインインしてください。
          </CardDescription>
        </CardHeader>
        <CardContent>
          <form className="grid gap-4" onSubmit={onSubmit}>
            <div className="grid gap-2">
              <Label htmlFor="email">メールアドレス</Label>
              <div className="relative">
                <Mail className="pointer-events-none absolute left-3 top-1/2 -translate-y-1/2 size-4 text-muted-foreground" />
                <Input
                  id="email"
                  name="email"
                  type="email"
                  inputMode="email"
                  autoComplete="email"
                  placeholder="you@example.com"
                  required
                  className="pl-9"
                />
              </div>
            </div>
            <div className="grid gap-2">
              <div className="flex items-center justify-between">
                <Label htmlFor="password">パスワード</Label>
                <Link
                  href="#"
                  className="text-xs text-muted-foreground underline-offset-4 hover:underline"
                >
                  パスワードをお忘れの方
                </Link>
              </div>
              <div className="relative">
                <Lock className="pointer-events-none absolute left-3 top-1/2 -translate-y-1/2 size-4 text-muted-foreground" />
                <Input
                  id="password"
                  name="password"
                  type="password"
                  autoComplete="current-password"
                  required
                  className="pl-9"
                />
              </div>
            </div>
            <Button type="submit" disabled={loading} className="w-full">
              {loading ? (
                <>
                  <Loader2 className="animate-spin" />
                  サインイン中...
                </>
              ) : (
                "サインイン"
              )}
            </Button>
          </form>
          {error && <p className="mt-2 text-sm text-red-600">{error}</p>}
          <div className="mt-4 text-center text-sm text-muted-foreground">
            または
          </div>
          <div className="mt-4 grid gap-2">
            <Button
              type="button"
              variant="outline"
              className="w-full"
              onClick={() => signIn("google", { callbackUrl: "/" })}
            >
              <svg
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 48 48"
                className="size-4"
                aria-hidden="true"
                focusable="false"
              >
                <title>Google</title>
                <path
                  fill="#FFC107"
                  d="M43.611,20.083H42V20H24v8h11.303C33.621,32.091,29.224,35,24,35c-6.627,0-12-5.373-12-12 s5.373-12,12-12c3.059,0,5.842,1.153,7.961,3.039l5.657-5.657C34.046,5.1,29.268,3,24,3C12.955,3,4,11.955,4,23 s8.955,20,20,20s20-8.955,20-20C44,22.659,43.862,21.35,43.611,20.083z"
                />
                <path
                  fill="#FF3D00"
                  d="M6.306,14.691l6.571,4.819C14.655,16.108,18.961,13,24,13c3.059,0,5.842,1.153,7.961,3.039l5.657-5.657 C34.046,5.1,29.268,3,24,3C16.318,3,9.656,7.337,6.306,14.691z"
                />
                <path
                  fill="#4CAF50"
                  d="M24,43c5.166,0,9.86-1.977,13.409-5.193l-6.191-5.238C29.211,34.091,26.715,35,24,35 c-5.192,0-9.574-3.281-11.287-7.861l-6.53,5.033C9.5,39.556,16.227,43,24,43z"
                />
                <path
                  fill="#1976D2"
                  d="M43.611,20.083H42V20H24v8h11.303c-1.092,3.008-3.285,5.466-6.087,7.069l0.001-0.001l6.191,5.238 C34.242,40.205,44,34,44,23C44,22.659,43.862,21.35,43.611,20.083z"
                />
              </svg>
              Googleで続行
            </Button>
          </div>
        </CardContent>
        <CardFooter className="justify-center text-sm text-muted-foreground">
          アカウントをお持ちでないですか？
          <Link href="#" className="ml-1 underline underline-offset-4">
            新規登録
          </Link>
        </CardFooter>
      </Card>
    </main>
  );
}
```

### 6.3 トップページの作成（ログアウト機能付き）

```tsx
// src/app/page.tsx
import { auth, signOut } from "@/auth";
import { Button } from "@/components/ui/button";

export default async function Home() {
  const session = await auth();

  return (
    <main className="min-h-screen flex flex-col items-center justify-center p-6 gap-4">
      <div className="text-center">
        {session?.user ? (
          <>
            <p className="text-lg mb-4">
              ログイン中: {session.user.name ?? session.user.email}
            </p>
            <form
              action={async () => {
                "use server";
                await signOut({ redirectTo: "/login" });
              }}
            >
              <Button type="submit" variant="destructive">
                ログアウト
              </Button>
            </form>
          </>
        ) : (
          <>
            <p className="text-lg mb-4">未ログイン</p>
            <Button asChild>
              <a href="/login">ログイン</a>
            </Button>
          </>
        )}
      </div>
    </main>
  );
}
```

## 7. シードデータの作成

テストユーザーを作成するためのシードスクリプトを用意します。

### 7.1 シードスクリプトの作成

```typescript
// prisma/seed.ts
import "dotenv/config";
import { PrismaClient } from "@prisma/client";
import bcrypt from "bcrypt";

const prisma = new PrismaClient();

async function main() {
  const users = [
    { name: "Alice", email: "alice@example.com", password: "password123", emailVerified: true },
    { name: "Bob", email: "bob@example.com", password: "password123", emailVerified: true },
  ];

  for (const u of users) {
    const hashed = await bcrypt.hash(u.password, 10);
    await prisma.user.upsert({
      where: { email: u.email },
      update: { name: u.name },
      create: {
        name: u.name,
        email: u.email,
        hashedPassword: hashed,
        emailVerified: u.emailVerified ? new Date() : null,
      },
    });
  }

  console.log("Seed completed");
}

main()
  .catch((e) => {
    console.error("Seed failed:", e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

### 7.2 package.json にシード設定を追加

```json
{
  "prisma": {
    "seed": "tsx prisma/seed.ts"
  }
}
```

### 7.3 シードの実行

```bash
npx prisma db seed
```

## 8. 動作確認

### 8.1 開発サーバーの起動

```bash
npm run dev
```

### 8.2 ログインテスト

1. http://localhost:3000/login にアクセス
2. 以下のいずれかでログイン
   - alice@example.com / password123
   - bob@example.com / password123
3. ログイン成功後、トップページにユーザー名が表示される
4. ログアウトボタンでログアウトできる

### 8.3 セッション確認（デバッグ用）

セッション情報を確認するページを追加すると便利です。

```tsx
// src/app/debug/session/page.tsx
import { auth } from "@/auth";

export default async function SessionDebugPage() {
  const session = await auth();
  return (
    <pre className="p-4 text-sm whitespace-pre-wrap break-words">
      {JSON.stringify(session, null, 2)}
    </pre>
  );
}
```

http://localhost:3000/debug/session でセッション情報が JSON で確認できます。

## 9. トラブルシューティング

### 9.1 マイグレーションが止まる

**原因**: `DIRECT_URL` が pooler を向いている

**解決策**: `.env` の `DIRECT_URL` を non-pooler（supabase.com:5432）に変更

```env
DIRECT_URL="postgresql://postgres.<ref>:<password>@aws-0-ap-northeast-1.supabase.com:5432/postgres?sslmode=require"
```

### 9.2 ログイン後もセッションが null

**原因**: Credentials プロバイダーは database session と互換性がない

**解決策**: `src/auth.ts` で `session.strategy: "jwt"` を使用（本記事の設定済み）

### 9.3 クッキーが付かない

**原因**: アクセス URL と `NEXTAUTH_URL` のホストが異なる

**解決策**: 
- ブラウザで http://localhost:3000 にアクセス（127.0.0.1 は使わない）
- `.env` に `AUTH_URL="http://localhost:3000"` を追加

## まとめ

- Next.js 15 + Auth.js v5 (next-auth@beta) で認証機能を実装
- Supabase Postgres と Prisma で DB 接続
- JWT セッション戦略で Credentials 認証を実現
- shadcn/ui でログイン画面を実装
- シードデータでテストユーザーを作成

次回は「習慣トラッカーの基本機能（CRUD）実装」についてまとめる予定です。

---

## 参考リンク

- [Auth.js Documentation](https://authjs.dev/)
- [Prisma Documentation](https://www.prisma.io/docs)
- [Supabase Documentation](https://supabase.com/docs)
- [shadcn/ui](https://ui.shadcn.com/)
