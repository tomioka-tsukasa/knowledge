# rewrite方式 vs 直接返す方式の比較

## パターン1: APIルートにrewriteする方式（記事の方式）

### メリット
- middlewareがシンプル
- 401レスポンスの管理が一箇所
- エラーメッセージのカスタマイズが容易

### デメリット
- 仕組みが分かりにくい
- ファイルが2つ必要

### コード

**src/middleware.ts:**
```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  if (process.env.NODE_ENV !== 'production') {
    return NextResponse.next()
  }

  const authHeader = request.headers.get('authorization')

  if (!authHeader) {
    // APIルートにrewrite
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

  // 認証失敗もAPIルートにrewrite
  return NextResponse.rewrite(new URL('/api/auth', request.url))
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

**src/app/api/auth/route.ts:**
```typescript
import { NextResponse } from 'next/server'

export function GET() {
  return new NextResponse('Authentication required', {
    status: 401,
    headers: { 'WWW-Authenticate': 'Basic realm="Secure Area"' },
  })
}
```

---

## パターン2: middlewareから直接返す方式（シンプル）

### メリット
- 仕組みが分かりやすい
- ファイルが1つだけで完結

### デメリット
- middlewareが少し長くなる

### コード

**src/middleware.ts:**
```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  if (process.env.NODE_ENV !== 'production') {
    return NextResponse.next()
  }

  const authHeader = request.headers.get('authorization')

  // 認証ヘッダーがない場合、直接401を返す
  if (!authHeader) {
    return new NextResponse('Authentication required', {
      status: 401,
      headers: { 'WWW-Authenticate': 'Basic realm="Secure Area"' },
    })
  }

  const auth = authHeader.split(' ')[1]
  const [user, password] = atob(auth).split(':')

  if (
    user === process.env.BASIC_AUTH_USER &&
    password === process.env.BASIC_AUTH_PASSWORD
  ) {
    return NextResponse.next()
  }

  // 認証失敗の場合も直接401を返す
  return new NextResponse('Authentication required', {
    status: 401,
    headers: { 'WWW-Authenticate': 'Basic realm="Secure Area"' },
  })
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}
```

**APIルートは不要！**

---

## どちらを選ぶべきか

### 小規模プロジェクト・シンプル重視
→ **パターン2（直接返す方式）**がおすすめ

### 大規模プロジェクト・拡張性重視
→ **パターン1（rewrite方式）**がおすすめ

### 判断基準

| 項目 | パターン1（rewrite） | パターン2（直接） |
|------|---------------------|------------------|
| ファイル数 | 2つ | 1つ |
| 分かりやすさ | △ | ◎ |
| middlewareの行数 | 少ない | やや多い |
| カスタマイズ性 | 高い | 低い |
| エラーメッセージ変更 | APIルートのみ修正 | middleware修正 |

---

## rewriteの動作を理解する

### rewriteとは

```typescript
NextResponse.rewrite(new URL('/api/auth', request.url))
```

**意味:**
- ユーザーには元のURL（例: `/about`）を見せたまま
- 実際には `/api/auth` の処理を実行する
- URLバーは変わらない

### redirectとの違い

```typescript
// redirect: URLが変わる（ブラウザが再リクエスト）
NextResponse.redirect(new URL('/login', request.url))
// ユーザーは /login にリダイレクトされる

// rewrite: URLは変わらない（サーバー側で処理を切り替え）
NextResponse.rewrite(new URL('/api/auth', request.url))
// ユーザーは元のURLのまま、/api/auth の処理が実行される
```

---

## まとめ

- **rewrite方式**: APIルートに処理を委譲する（元記事の方式）
- **直接返す方式**: middlewareで完結する（シンプル）
- 小規模なら直接返す方式がおすすめ
- どちらも動作は同じ（認証ダイアログが表示される）
