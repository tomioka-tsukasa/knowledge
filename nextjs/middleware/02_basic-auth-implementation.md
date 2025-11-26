# Next.js でのBasic認証実装ガイド

## Basic認証とは

Basic認証は、HTTPの標準的な認証方式の一つで、ユーザー名とパスワードを使ってアクセスを制限する仕組みです。

### 特徴
- ブラウザが標準でサポート
- 簡単に実装できる
- 開発・ステージング環境の保護に最適
- **本番環境の重要なデータには不向き**（より強固な認証が必要）

### 動作の流れ

1. クライアントがページにアクセス
2. サーバーが`401 Unauthorized`と`WWW-Authenticate`ヘッダーを返す
3. ブラウザが認証ダイアログを表示
4. ユーザーがユーザー名とパスワードを入力
5. ブラウザが`Authorization`ヘッダーに認証情報を付けて再リクエスト
6. サーバーが認証情報を検証
7. 正しければアクセス許可、間違っていれば再度401を返す

## 実装方法（Next.js Middleware使用）

### ステップ1: 環境変数の設定

`.env.local`ファイルを作成し、認証情報を設定：

```bash
# Basic認証用の認証情報
BASIC_AUTH_USER=admin
BASIC_AUTH_PASSWORD=your-secure-password
```

**重要:** `.env.local`は`.gitignore`に含めること！

### ステップ2: middleware.tsの作成

`src/middleware.ts`（または`middleware.ts`）を作成：

```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // 本番環境でのみBasic認証を有効化
  if (process.env.NODE_ENV !== 'production') {
    return NextResponse.next()
  }

  // Authorizationヘッダーを取得
  const authHeader = request.headers.get('authorization')

  // 認証ヘッダーがない場合、認証APIにリダイレクト
  if (!authHeader) {
    return NextResponse.rewrite(new URL('/api/auth', request.url))
  }

  // "Basic " の後の認証情報を取り出す
  const auth = authHeader.split(' ')[1]

  // Base64デコード（atobを使用、Buffer.fromは使わない）
  const [user, password] = atob(auth).split(':')

  // 環境変数から認証情報を取得
  const validUser = process.env.BASIC_AUTH_USER
  const validPassword = process.env.BASIC_AUTH_PASSWORD

  // 認証情報が正しいかチェック
  if (user === validUser && password === validPassword) {
    // 認証成功：次の処理に進む
    return NextResponse.next()
  }

  // 認証失敗：再度認証APIにリダイレクト
  return NextResponse.rewrite(new URL('/api/auth', request.url))
}

// 適用するパスを設定（静的ファイルなどは除外）
export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

### ステップ3: 認証APIルートの作成

`src/app/api/auth/route.ts`を作成：

```typescript
import { NextResponse } from 'next/server'

export function GET() {
  return new NextResponse('Authentication required', {
    status: 401,
    headers: {
      'WWW-Authenticate': 'Basic realm="Secure Area"',
    },
  })
}
```

### ステップ4: Vercelへのデプロイ設定

Vercelの環境変数設定画面で、以下を追加：

1. Vercelダッシュボードでプロジェクトを開く
2. Settings → Environment Variables
3. 以下を追加：
   - `BASIC_AUTH_USER`: admin（お好みの値）
   - `BASIC_AUTH_PASSWORD`: 強固なパスワード

## 重要なポイント

### 1. `Buffer.from`ではなく`atob()`を使用

❌ **古い記事の書き方（動かない）:**
```typescript
const [user, password] = Buffer.from(auth, 'base64').toString().split(':')
```

✅ **正しい書き方（Edge Runtimeで動作）:**
```typescript
const [user, password] = atob(auth).split(':')
```

**理由:** Next.js MiddlewareはEdge Runtimeで動作し、Node.jsの`Buffer`が使えないため。

### 2. 環境変数で認証情報を管理

```typescript
// ❌ ハードコードは絶対NG
if (user === 'admin' && password === 'password123') {
  // ...
}

// ✅ 環境変数から取得
const validUser = process.env.BASIC_AUTH_USER
const validPassword = process.env.BASIC_AUTH_PASSWORD
if (user === validUser && password === validPassword) {
  // ...
}
```

### 3. 本番環境でのみ有効化

```typescript
if (process.env.NODE_ENV !== 'production') {
  return NextResponse.next()
}
```

開発中は認証ダイアログが煩わしいため、本番環境でのみ有効にするのが一般的。

### 4. 静的ファイルは除外

```typescript
export const config = {
  matcher: [
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
}
```

画像やCSSなどの静的ファイルに認証をかけると、正しく表示されなくなる可能性があります。

## テスト方法

### ローカルでのテスト

1. `.env.local`を`.env.production.local`にコピー
2. `NODE_ENV=production npm run dev`で起動
3. ブラウザでアクセスし、認証ダイアログが表示されることを確認

### curlでのテスト

```bash
# 認証なし（401が返る）
curl -I https://your-site.com

# 認証あり（200が返る）
curl -I -u admin:password https://your-site.com
```

## セキュリティに関する注意事項

### Basic認証の限界
- 認証情報はBase64エンコードされているだけ（暗号化ではない）
- HTTPSを使わないと平文で送信される
- セッション管理がない（毎回認証情報を送る）

### 推奨される使用場面
✅ 開発環境・ステージング環境の保護
✅ 社内向けツール
✅ プレビュー環境

### 避けるべき使用場面
❌ 本番環境の重要なデータ保護
❌ ユーザーごとの権限管理が必要な場合
❌ セキュリティ要件が高いアプリケーション

## まとめ

1. Basic認証は簡単に実装できる軽量な認証方式
2. Next.js MiddlewareとAPIルートを組み合わせて実装
3. `atob()`を使用（`Buffer.from`は使わない）
4. 環境変数で認証情報を管理
5. 本番環境でのみ有効化するのが一般的
6. 重要なデータには、より強固な認証方式を使用すること
