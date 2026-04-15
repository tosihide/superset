# Ubuntu 24.04 LTS ビルド＆起動マニュアル

SupersetデスクトップアプリをUbuntu 24.04 LTSでビルド・起動する手順です。

## 前提条件

- [Ubuntu 24.04 LTS インストールマニュアル](./ubuntu-install.md) を参考にOSセットアップ済み
- [Ubuntu操作マニュアル](./ubuntu-operations.md) を参考に基本操作を理解済み
- Git 2.20+

## 1. 依存パッケージのインストール

### 1-1. ユーザー権限でインストール

```bash
# JavaScript ランタイム
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc

# JSON プロセッサー
sudo apt-get install -y jq

# HTTP/2 リバースプロキシ（Electric SQL用）
sudo apt-get install -y caddy

# Neon データベース CLI（Neon使用時のみ）
npm install -g neonctl
```

### 1-2. root権限が必要なパッケージ（Neon Pro使用時のみ）

```bash
# Docker（Electric SQLコンテナの実行に必要）
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# dockerグループの反映にはログアウト→再ログイン、または `newgrp docker` が必要

# PostgreSQLクライアント（Neon DB のレプリケーションクリーンアップ用、任意）
sudo apt-get install -y postgresql-client
```

## 2. リポジトリのクローン

```bash
git clone <REPOSITORY_URL>
cd superset
git checkout linux_develop
```

## 3. .env の設定

`.env` ファイルはリポジトリルートに配置します。テンプレートとして `.env.workspace.ubuntu` を用意しています。

### 3-1. .env の作成

`.env.workspace.ubuntu` をコピーして `.env` を作成します:

```bash
cp .env.workspace.ubuntu .env
```

### 3-2. .env の設定内容

`.env` は2つの部分から構成されます:

#### ルート設定（APIキー・シークレット等）

`.env` の先頭部分に、各種APIキーやシークレットを設定します。**SQLiteのみモードで動かす場合、ほとんどの項目は空のままでOKです。**

| 設定値 | 必須度 | 説明 |
|---|---|---|
| `NEON_ORG_ID` | 任意 | Neon組織ID。SQLiteのみの場合は空欄 |
| `NEON_PROJECT_ID` | 任意 | NeonプロジェクトID。SQLiteのみの場合は空欄 |
| `NEON_API_KEY` | 任意 | Neon APIキー |
| `DATABASE_URL` | 任意 | PostgreSQL接続URL。SQLiteのみの場合は空欄でOK |
| `BETTER_AUTH_SECRET` | 任意 | 認証シークレット。未設定でも起動可能 |
| `RESEND_API_KEY` | 任意 | メール送信APIキー。未設定だとWeb画面でエラーが出るが起動は可能 |
| `SKIP_ENV_VALIDATION=1` | **必須** | APIサーバーの環境変数バリデーションをスキップ |
| `NO_SANDBOX=1` | **必須** | Electronのサンドボックスを無効化（Linux必須） |

#### ワークスペース設定（ポート・URL等）

`.env.workspace.ubuntu` の内容を `.env` の末尾に追記します。`setup.sh` を実行すると自動的に追記されます。

```bash
# ワークスペース設定を.envに追記（setup.shが自動で行います）
cat .env.workspace.ubuntu >> .env
```

**手動で追記する場合、`SUPERSET_HOME_DIR` を自分の環境に合わせて変更してください:**

```env
SUPERSET_HOME_DIR="/path/to/superset/superset-dev-data"
```

### 3-3. .env.workspace.ubuntu の内容一覧

| 設定値 | 説明 |
|---|---|
| `SKIP_ENV_VALIDATION=1` | APIサーバーの環境変数バリデーションをスキップ |
| `NO_SANDBOX=1` | Electronのサンドボックスを無効化 |
| `SUPERSET_WORKSPACE_NAME` | ワークスペース名 |
| `SUPERSET_HOME_DIR` | ワークスペースデータディレクトリ（要変更） |
| `ELECTRIC_PORT` / `ELECTRIC_URL` | Electric SQL設定（Neon Proのみ使用） |
| `WEB_PORT`〜`WRANGLER_PORT` | 各サービスのポート番号 |
| `NEXT_PUBLIC_*_URL` | クロスアプリURL |
| `STREAMS_URL` / `STREAMS_INTERNAL_URL` | ストリーミングURL |

## 4. セットアップ

```bash
export SUPERSET_ROOT_PATH="$PWD"
bash .superset/setup.sh
```

### SQLiteのみモード（Neon未設定時）

```
📊 Setup Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Skipped steps:
  - Seed local DB (no source DB)
  - Seed auth token (no source token)
  - Neon branch setup skipped (using local SQLite only)
  - Electric SQL skipped (no Neon branch — using local SQLite only)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 5. 開発サーバーの起動

```bash
NO_SANDBOX=1 bun run dev
```

### 起動するサービス

| サービス | ポート | URL | 説明 |
|---|---|---|---|
| Web | 3000 | http://localhost:3000 | Next.js フロントエンド |
| API | 3001 | http://localhost:3001 | Next.js APIサーバー |
| Desktop | 3005 | http://localhost:3005 | Electronデスクトップアプリ（Vite dev） |
| Caddy | 3010 | http://localhost:3010 | HTTP/2リバースプロキシ |
| Wrangler | 3013 | http://localhost:3013 | Electric SQLプロキシワーカー |

### 初回ビルド

初回起動時はネイティブモジュールのビルドとElectronのビルドが行われるため、約30〜60秒かかります。

### サービスの停止

```
Ctrl+C
```

ポートが残った場合は:

```bash
fuser -k 3000/tcp 3001/tcp 3005/tcp 3010/tcp 3013/tcp
```

## 6. トラブルシューティング

### Electronが起動しない（Sandboxエラー）

```
FATAL:sandbox/linux/suid/client/setuid_sandbox_host.cc:166
```

**対策**: `NO_SANDBOX=1 bun run dev` で起動。

### ポートが使用中（EADDRINUSE）

```bash
fuser -k 3000/tcp 3001/tcp 3005/tcp 3010/tcp 3013/tcp
```

### Web/APIがHTTP 500を返す

`RESEND_API_KEY` 等が未設定の場合に発生します。Desktopアプリ（ポート3005）自体は動作しているため、開発用途では問題ありません。

### Caddyがポート80でバインドエラー

`setup.sh` が生成するCaddyfileに `auto_https off` が設定されていることを確認してください。
