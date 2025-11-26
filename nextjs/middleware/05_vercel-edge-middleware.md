# Vercel Edge Middleware でBasic認証を実装する

## Vercel Edge Middlewareとは

Vercelが提供する、**静的サイト（`output: 'export'`）でも動作するMiddleware**です。

### Next.js Middlewareとの違い

| 項目 | Next.js Middleware | Vercel Edge Middleware |
|------|-------------------|----------------------|
| ファイル名 | `src/middleware.ts` | `middleware.js`（ルート直下） |
| パッケージ | `next/server` | `@vercel/edge` |
| output: 'export' | ❌ 動かない | ✅ 動く |
| デプロイ先 | どこでもOK | Vercel専用 |
| 用途 | Next.jsアプリ全般 | 静的サイトに認証追加 |

## 仕組み

```
静的サイト（output: 'export'）
    ↓
Vercelがルート直下のmiddleware.jsを検出
    ↓
Edge Runtime（CDN）でMiddlewareを実行
    ↓
認証チェック → 静的ファイルを返す
```

**重要:** Vercelのインフラが`middleware.js`を特別扱いしてくれるため、静的サイトでも動作します。

## 実装方法

### ステップ1: パッケージインストール

```bash
npm install @vercel/edge basic-auth static-auth
```

### ステップ2: middleware.js作成

**ルートディレクトリ**（`package.json`と同じ階層）に`middleware.js`を作成：

```javascript
import { next } from '@vercel/edge'

export const config = {
  matcher: '/(.*)',  // すべてのパスに適用
}

export default function middleware(request) {
  // 本番環境でのみBasic認証を有効化
  if (process.env.NODE_ENV !== 'production') {
    return next()
  }

  const authorizationHeader = request.headers.get('authorization')

  if (authorizationHeader) {
    const basicAuth = authorizationHeader.split(' ')[1]
    const [user, password] = atob(basicAuth).toString().split(':')

    if (
      user === process.env.BASIC_AUTH_USER &&
      password === process.env.BASIC_AUTH_PASSWORD
    ) {
      return next()  // 認証成功
    }
  }

  // 認証失敗または未認証
  return new Response('Authentication required', {
    status: 401,
    headers: { 'WWW-Authenticate': 'Basic realm="Secure Area"' },
  })
}
```

### ステップ3: 環境変数設定

`.env.local`に追加：

```bash
BASIC_AUTH_USER=admin
BASIC_AUTH_PASSWORD=your-secure-password
```

### ステップ4: Vercelに環境変数を設定

1. Vercelダッシュボード → プロジェクト選択
2. **Settings** → **Environment Variables**
3. 以下を追加：
   - `BASIC_AUTH_USER`: `admin`
   - `BASIC_AUTH_PASSWORD`: 強固なパスワード
   - Environment: **Production** を選択

### ステップ5: デプロイ

```bash
git add .
git commit -m "Add Vercel Edge Middleware for Basic Auth"
git push
```

Vercelが自動的にデプロイし、Basic認証が有効になります。

## ファイル配置の重要性

### ❌ 間違った配置

```
project/
├── src/
│   └── middleware.js  ← ここに置くとNext.js Middlewareとして扱われる
├── package.json
└── next.config.ts
```

### ✅ 正しい配置

```
project/
├── middleware.js  ← ルート直下に配置！
├── src/
│   └── app/
├── package.json
└── next.config.ts
```

**理由:** Vercelは**ルートディレクトリの`middleware.js`**を特別なファイルとして認識します。

## コードの詳細解説

### 1. next()関数

```javascript
import { next } from '@vercel/edge'

// 認証成功時
return next()  // 次の処理（静的ファイル返却）に進む
```

### 2. matcher設定

```javascript
export const config = {
  matcher: '/(.*)',  // すべてのパスに適用
}
```

**除外パターンの例:**

```javascript
export const config = {
  // 静的ファイルを除外
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
}
```

ただし、Vercelの場合は静的ファイルは自動的に最適化されるため、`'/(.*)'`でも問題ありません。

### 3. 環境による分岐

```javascript
if (process.env.NODE_ENV !== 'production') {
  return next()  // 開発環境では認証スキップ
}
```

開発中は認証ダイアログが煩わしいため、本番環境でのみ有効化します。

### 4. 認証ロジック

```javascript
// "Basic YWRtaW46cGFzc3dvcmQ=" から "YWRtaW46cGFzc3dvcmQ=" を取り出す
const basicAuth = authorizationHeader.split(' ')[1]

// Base64デコード: "YWRtaW46cGFzc3dvcmQ=" → "admin:password"
const [user, password] = atob(basicAuth).toString().split(':')

// 環境変数と照合
if (user === process.env.BASIC_AUTH_USER && password === process.env.BASIC_AUTH_PASSWORD) {
  return next()
}
```

## ローカルでのテスト方法

### 方法1: 本番モードで起動

```bash
NODE_ENV=production npm run dev
```

ブラウザでアクセスすると認証ダイアログが表示されます。

### 方法2: 一時的に条件を変更

```javascript
// middleware.js
export default function middleware(request) {
  // 一時的にコメントアウト
  // if (process.env.NODE_ENV !== 'production') {
  //   return next()
  // }

  // ... 認証処理
}
```

## よくある問題と解決法

### 問題1: 認証ダイアログが表示されない

**原因:**
- `middleware.js`の配置場所が間違っている
- `@vercel/edge`がインストールされていない

**解決:**
- ルートディレクトリに配置されているか確認
- `npm install @vercel/edge`を実行

### 問題2: "Middleware cannot be used with output: export"

**原因:**
- `src/middleware.ts`が残っている（Next.js Middleware）

**解決:**
- `src/middleware.ts`を削除
- `middleware.js`をルートに配置

### 問題3: ローカルで動作しない

**原因:**
- `NODE_ENV`が`development`になっている

**解決:**
```bash
NODE_ENV=production npm run dev
```

### 問題4: 画像やCSSが読み込まれない

**原因:**
- matcherで除外していない

**解決:**
通常は問題ありませんが、もし発生したら：

```javascript
export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|assets).*)'],
}
```

## Vercelでの動作確認

### デプロイ後の確認

1. デプロイ完了後、サイトにアクセス
2. 認証ダイアログが表示される
3. ユーザー名・パスワードを入力
4. サイトが表示される

### curlでのテスト

```bash
# 認証なし（401が返る）
curl -I https://your-site.vercel.app

# 認証あり（200が返る）
curl -I -u admin:password https://your-site.vercel.app
```

### URLに直接認証情報を含める（テスト用）

```
https://admin:password@your-site.vercel.app
```

**注意:** 本番環境では使用しないでください（URLに認証情報が表示される）

## セキュリティに関する注意

### 推奨される使用場面

✅ 開発・ステージング環境の保護
✅ プレビュー環境
✅ 社内向けツール
✅ デモサイト

### 避けるべき使用場面

❌ 本番環境の重要なデータ
❌ 個人情報を扱うサイト
❌ ユーザーごとの権限管理が必要

### より強固なセキュリティが必要な場合

- **NextAuth.js** - セッション管理付き認証
- **Auth0** - サードパーティ認証サービス
- **Clerk** - 認証UI付きサービス
- **Supabase Auth** - データベース統合認証

## まとめ

### Vercel Edge Middlewareの特徴

- 📦 静的サイト（`output: 'export'`）でも動作
- 🚀 Edge Runtimeで高速
- 🔒 簡単にBasic認証を追加できる
- ☁️ Vercel専用（他のホスティングでは動かない）

### 実装チェックリスト

- [ ] `@vercel/edge`をインストール
- [ ] **ルートディレクトリ**に`middleware.js`を作成
- [ ] `.env.local`に`BASIC_AUTH_USER`と`BASIC_AUTH_PASSWORD`を設定
- [ ] Vercelの環境変数に認証情報を設定
- [ ] デプロイして動作確認

### Next.js Middlewareとの使い分け

| 状況 | 使うべきMiddleware |
|------|------------------|
| 静的サイト（output: 'export'） | Vercel Edge Middleware |
| SSR/ISRを使っている | Next.js Middleware |
| Vercel以外にデプロイ | Next.js Middleware |
| 複雑な認証ロジック | Next.js Middleware |

## 参考リンク

- [Vercel Edge Middleware ドキュメント](https://vercel.com/docs/functions/edge-middleware)
- [@vercel/edge NPM](https://www.npmjs.com/package/@vercel/edge)
- [元記事（Qiita）](https://qiita.com/nakagami5963/items/d647d9f81382c89dfde0)
