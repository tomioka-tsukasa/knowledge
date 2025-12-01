# dotfiles + シェルエイリアス設定ガイド

## 概要
dotfilesとシェルエイリアスを使って、よく使うコマンドを効率的に管理する方法

## セットアップ手順

### 1. dotfilesディレクトリ作成
```bash
mkdir ~/.dotfiles
cd ~/.dotfiles
```

### 2. エイリアスファイル作成
```bash
# ~/.dotfiles/aliases.sh
# Claude Code関連
alias cc-mcp-figma="claude mcp add --name figma-dev-mode-mcp-server --command npx --args @figmaapi/figma-dev-mode-mcp-server"
alias cc-mcp-list="claude mcp list"
alias cc-mcp-remove="claude mcp remove"

# Git関連
alias gcp="git add . && git commit -m"
alias gpd="git push origin develop"
alias gpm="git push origin main"
alias gst="git status"
alias gdf="git diff"

# 開発関連
alias nrd="npm run dev"
alias nrb="npm run build"
alias nrt="npm run test"

# ナレッジ管理
alias kb="code ~/knowledge"
alias kb-sync="cd ~/knowledge && git add . && git commit -m 'update knowledge' && git push"
```

### 3. 環境変数ファイル作成（オプション）
```bash
# ~/.dotfiles/exports.sh
export EDITOR=code
export NODE_ENV=development
export PATH="$HOME/.local/bin:$PATH"
```

### 4. ~/.zshrc に読み込み設定追加
```bash
# ~/.zshrc の最後に追加
source ~/.dotfiles/aliases.sh
source ~/.dotfiles/exports.sh
```

### 5. 設定を反映
```bash
source ~/.zshrc
```

## Git管理（推奨）

### 初期設定
```bash
cd ~/.dotfiles
git init
git add .
git commit -m "Initial dotfiles setup"
git remote add origin https://github.com/your-name/dotfiles
git push -u origin main
```

### 新PC移行時
```bash
# dotfilesをクローン
git clone https://github.com/your-name/dotfiles ~/.dotfiles

# ~/.zshrc に読み込み設定追加
echo "source ~/.dotfiles/aliases.sh" >> ~/.zshrc
echo "source ~/.dotfiles/exports.sh" >> ~/.zshrc

# 設定を反映
source ~/.zshrc
```

## 使い方

### エイリアス追加
1. `~/.dotfiles/aliases.sh` を編集
2. `source ~/.zshrc` で反映

### よく使うエイリアス例
```bash
# コマンド確認
cc-mcp-list

# Figma MCP追加
cc-mcp-figma

# 簡単コミット
gcp "feat: 新機能追加"

# ナレッジ編集
kb
```

## トラブルシューティング

### エイリアスが効かない場合
```bash
# 設定ファイルをリロード
source ~/.zshrc

# ファイルパスを確認
ls -la ~/.dotfiles/
```

### コマンドが見つからない場合
```bash
# パスを確認
echo $PATH

# コマンドの場所を確認
which claude
which npm
```

