# Next.js Middleware（ミドルウェア）とは

## ミドルウェアの基礎知識

### ミドルウェアとは
ミドルウェアとは、リクエストとレスポンスの間に介在する処理層のことです。クライアントからのリクエストがサーバーに到達する前に、何らかの処理を挟むことができます。

```
クライアント → ミドルウェア → サーバー（ページ）
```

### よくある使用例
- 認証・認可（ログイン確認、Basic認証など）
- リダイレクト処理
- リクエストヘッダーの書き換え
- A/Bテスト
- 国際化対応（言語判定）
- ボット検出
- レート制限

## Next.js Middlewareの特徴

### 1. Edge Middlewareとして動作
Next.jsのmiddlewareは「Edge Middleware」として、CDNのエッジロケーションで実行されます。これにより、非常に高速に動作します。

### 2. ファイル配置
- プロジェクトルートの`middleware.ts`（または`src/middleware.ts`）に配置
- アプリケーション全体に適用される

### 3. 実行タイミング
- すべてのルートへのリクエスト前に実行
- 静的ファイル、画像、APIルートなど、すべてに対して実行可能

### 4. 制限事項
- Edge Runtimeで動作するため、Node.js APIの一部が使用不可
- `Buffer`の代わりに`atob()`/`btoa()`を使用
- ファイルシステムアクセス不可
- データベース接続は制限あり

## 基本的な使い方

### 最小構成のmiddleware.ts

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // ここでリクエストを処理
  console.log('リクエストパス:', request.nextUrl.pathname)

  // そのまま次の処理に進む
  return NextResponse.next()
}

// どのパスでmiddlewareを実行するか設定
export const config = {
  matcher: [
    /*
     * 以下を除くすべてのパスにマッチ:
     * - api (APIルート)
     * - _next/static (静的ファイル)
     * - _next/image (画像最適化ファイル)
     * - favicon.ico (faviconファイル)
     */
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

### リダイレクトの例

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // /oldページへのアクセスを/newにリダイレクト
  if (request.nextUrl.pathname === '/old') {
    return NextResponse.redirect(new URL('/new', request.url))
  }

  return NextResponse.next()
}
```

### ヘッダーの追加

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const response = NextResponse.next()

  // カスタムヘッダーを追加
  response.headers.set('x-custom-header', 'my-value')

  return response
}
```

### Rewriteの例

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // URLは変わらないが、別のページを表示
  if (request.nextUrl.pathname === '/about') {
    return NextResponse.rewrite(new URL('/about-us', request.url))
  }

  return NextResponse.next()
}
```

## matcherの詳細設定

### 特定のパスのみに適用

```typescript
export const config = {
  matcher: '/admin/:path*', // /adminとそのサブパス全てに適用
}
```

### 複数のパスを指定

```typescript
export const config = {
  matcher: [
    '/admin/:path*',
    '/dashboard/:path*',
  ],
}
```

### 正規表現を使用

```typescript
export const config = {
  matcher: [
    /*
     * 以下を除くすべてのパス:
     * - api (APIルート)
     * - _next/static (静的ファイル)
     * - _next/image (画像最適化)
     * - favicon.ico
     */
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

## 環境による処理の切り替え

開発環境と本番環境で処理を分ける例：

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // 本番環境でのみ実行
  if (process.env.NODE_ENV === 'production') {
    // Basic認証などの処理
    // ...
  }

  // 開発環境では何もせず通過
  return NextResponse.next()
}
```

## まとめ

- Middlewareはリクエスト処理の前に実行される
- 認証、リダイレクト、ヘッダー操作などに使用
- Edge Runtimeで動作するため高速だが制限あり
- `middleware.ts`に実装し、`matcher`で適用範囲を制御
- 環境変数で本番/開発の処理を切り替え可能
