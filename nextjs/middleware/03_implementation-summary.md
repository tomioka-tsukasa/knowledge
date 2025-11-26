# Basic認証実装の要約

## 実装の3ステップ

### ステップ1: 環境変数設定
`.env.local`に認証情報を追加：
```bash
BASIC_AUTH_USER=admin
BASIC_AUTH_PASSWORD=your-password
```

### ステップ2: Middleware作成
`src/middleware.ts`を作成：
```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  // 本番環境でのみ実行
  if (process.env.NODE_ENV !== 'production') {
    return NextResponse.next()
  }

  const authHeader = request.headers.get('authorization')

  if (!authHeader) {
    return NextResponse.rewrite(new URL('/api/auth', request.url))
  }

  const auth = authHeader.split(' ')[1]
  const [user, password] = atob(auth).split(':')

  if (
    user === process.env.BASIC_AUTH_USER &&
    password === process.env.BASIC_AUTH_PASSWORD
  ) {
    return NextResponse.next()
  }

  return NextResponse.rewrite(new URL('/api/auth', request.url))
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

### ステップ3: 認証APIルート作成
`src/app/api/auth/route.ts`を作成：
```typescript
import { NextResponse } from 'next/server'

export function GET() {
  return new NextResponse('Authentication required', {
    status: 401,
    headers: { 'WWW-Authenticate': 'Basic realm="Secure Area"' },
  })
}
```

## キーポイント

| 項目 | 内容 |
|------|------|
| 実行タイミング | リクエスト前（Middleware） |
| 認証方式 | HTTP Basic認証 |
| デコード方法 | `atob()`（`Buffer.from`は不可） |
| 認証情報管理 | 環境変数 |
| 適用環境 | 本番環境のみ（`NODE_ENV === 'production'`） |
| 除外パス | 静的ファイル、API、画像など |

## チェックリスト

実装時の確認事項：

- [ ] `.env.local`に`BASIC_AUTH_USER`と`BASIC_AUTH_PASSWORD`を設定
- [ ] `.gitignore`に`.env.local`が含まれている
- [ ] `middleware.ts`で`atob()`を使用（`Buffer.from`ではない）
- [ ] 環境変数から認証情報を取得している
- [ ] 本番環境でのみ有効化している
- [ ] matcherで静的ファイルを除外している
- [ ] `api/auth/route.ts`で401と`WWW-Authenticate`ヘッダーを返している
- [ ] Vercelの環境変数に認証情報を設定

## よくあるエラーと対処法

### エラー1: "Buffer is not defined"
**原因:** Edge Runtimeで`Buffer.from`を使用している
**対処:** `atob()`に変更

```typescript
// ❌NG
const [user, password] = Buffer.from(auth, 'base64').toString().split(':')

// ✅OK
const [user, password] = atob(auth).split(':')
```

### エラー2: 認証ダイアログが表示されない
**原因:** `WWW-Authenticate`ヘッダーが設定されていない
**対処:** APIルートで正しくヘッダーを返す

```typescript
return new NextResponse('Authentication required', {
  status: 401,
  headers: { 'WWW-Authenticate': 'Basic realm="Secure Area"' },
})
```

### エラー3: 画像やCSSが読み込まれない
**原因:** 静的ファイルにも認証がかかっている
**対処:** matcherで静的ファイルを除外

```typescript
export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

### エラー4: ローカルで認証がかからない
**原因:** `NODE_ENV`が`development`になっている
**対処:** 本番モードで起動するか、条件を変更

```bash
# 本番モードで起動
NODE_ENV=production npm run dev
```

または、middleware.tsの条件を一時的に変更：
```typescript
// テスト用に常に有効化
// if (process.env.NODE_ENV !== 'production') {
//   return NextResponse.next()
// }
```

## 参考リンク

- [Next.js Middleware公式ドキュメント](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [MDN - HTTP認証](https://developer.mozilla.org/ja/docs/Web/HTTP/Authentication)
- [Vercel - Environment Variables](https://vercel.com/docs/projects/environment-variables)

## さらに学ぶために

Basic認証を理解したら、次のステップ：

1. **NextAuth.js** - より強固な認証システム
2. **JWT認証** - トークンベースの認証
3. **OAuth** - Google、GitHubなどのソーシャルログイン
4. **セッション管理** - ログイン状態の保持
5. **RBAC（Role-Based Access Control）** - ロールベースの権限管理
