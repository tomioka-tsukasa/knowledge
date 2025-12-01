# GitHub SSH キーのセットアップ手順

## 1. SSH キーの生成

```bash
# SSH キーペアを生成（Ed25519アルゴリズム推奨）
ssh-keygen -t ed25519 -C "your_email@example.com"

# または RSA を使用する場合
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

プロンプトが表示されたら:
- ファイルの保存場所を確認（デフォルト: `~/.ssh/id_ed25519`）
- パスフレーズを入力（推奨、省略も可能）

## 2. ssh-agent の設定

### ssh-agent の起動と SSH キーの追加

```bash
# ssh-agent をバックグラウンドで起動
eval "$(ssh-agent -s)"

# SSH 秘密鍵を ssh-agent に追加
ssh-add ~/.ssh/id_ed25519

# RSA を使用した場合
ssh-add ~/.ssh/id_rsa
```

### macOS の場合: キーチェーンへの自動追加設定

```bash
# ~/.ssh/config ファイルを作成/編集
touch ~/.ssh/config
```

`~/.ssh/config` に以下を追加:

```
Host github.com
  HostName github.com
  User git
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

キーをキーチェーンに追加:

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

## 3. .zshrc への記述

`~/.zshrc` に以下を追加して、シェル起動時に自動的に ssh-agent を設定:

```bash
# SSH Agent の自動起動設定
if [ -z "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" > /dev/null
fi

# macOS の場合はキーチェーンから自動的に読み込まれる
# Linux の場合は以下も追加
# ssh-add ~/.ssh/id_ed25519 2>/dev/null
```

変更を反映:

```bash
source ~/.zshrc
```

## 4. GitHub ブラウザでの設定手順

### 公開鍵のコピー

```bash
# 公開鍵の内容をクリップボードにコピー (macOS)
pbcopy < ~/.ssh/id_ed25519.pub

# Linux の場合
cat ~/.ssh/id_ed25519.pub
# 出力された内容を手動でコピー
```

### GitHub での設定

1. GitHub にログイン
2. 右上のプロフィールアイコンをクリック → **Settings** を選択
3. 左サイドバーの **SSH and GPG keys** をクリック
4. **New SSH key** または **Add SSH key** ボタンをクリック
5. 以下の情報を入力:
   - **Title**: キーの識別名（例: "MacBook Pro", "Work Laptop"）
   - **Key type**: Authentication Key（デフォルト）
   - **Key**: コピーした公開鍵を貼り付け
6. **Add SSH key** をクリック
7. パスワードの確認を求められた場合は GitHub パスワードを入力

## 5. 接続テスト

```bash
# GitHub への SSH 接続をテスト
ssh -T git@github.com
```

成功すると以下のようなメッセージが表示されます:

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

## トラブルシューティング

### Permission denied エラーが出る場合

```bash
# SSH キーが正しく追加されているか確認
ssh-add -l

# キーが表示されない場合は再度追加
ssh-add ~/.ssh/id_ed25519

# 詳細なデバッグ情報を表示
ssh -vT git@github.com
```

### 複数の GitHub アカウントを使用する場合

`~/.ssh/config` に複数のホスト設定を追加:

```
Host github.com-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

Host github.com-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
```

リポジトリのクローン時:

```bash
git clone git@github.com-personal:username/repo.git
git clone git@github.com-work:company/repo.git
```

## 参考リンク

- [GitHub Docs - Generating a new SSH key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
- [GitHub Docs - Adding a new SSH key to your GitHub account](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
