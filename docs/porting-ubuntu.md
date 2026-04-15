# Superset: Mac から Ubuntu 24.04 LTS への移植ガイド

このドキュメントでは、Supersetデスクトップアプリを macOS 環境から Ubuntu 24.04 LTS に移植する手順を説明します。

## 前提条件

- Ubuntu 24.04 LTS がインストール済み
- Git 2.20+
- インターネット接続

## 1. 依存パッケージのインストール

### 1-1. root権限不要（ユーザーインストール）

```bash
# JavaScript ランタイム
curl -fsSL https://bun.sh/install | bash

# JSON プロセッサー
sudo apt-get install -y jq

# HTTP/2 リバースプロキシ（Electric SQL用）
sudo apt-get install -y caddy

# Neon データベース CLI
npm install -g neonctl
```

### 1-2. root権限が必要（sudo が必要）

```bash
# Docker（Electric SQLコンテナの実行に必要、Neon Pro のみ）
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# ※ dockerグループの反映にはログアウト→再ログイン、または `newgrp docker` が必要

# PostgreSQLクライアント（Neon DB のレプリケーションクリーンアップ用、任意）
sudo apt-get install -y postgresql-client
```

## 2. Neon データベースの設定

NeonはPostgreSQLのマネージドサービスです。SQLiteのみで動かす場合はこのステップをスキップできます。

### 2-1. Neonアカウント作成

1. https://neon.tech にアクセス
2. GitHubアカウントまたはメールアドレスでサインアップ（無料）

### 2-2. 認証

```bash
neonctl auth
# ブラウザが開き、Neonにログイン
```

### 2-3. プロジェクト作成

```bash
neonctl projects create --name superset-dev --org-id <YOUR_ORG_ID>
```

`<YOUR_ORG_ID>` は `neonctl orgs` で確認できます。

作成後、プロジェクトID（例: `patient-tree-34029841`）をメモしてください。

### 2-4. Neon Pro の場合のみ: 論理レプリケーション有効化

Electric SQL（リアルタイム同期）を使用するにはNeon Proプランが必要です。Freeプランでは論理レプリケーション（`wal_level=logical`）が利用できません。

```bash
neonctl projects update <PROJECT_ID> --logical-replication true
```

## 3. リポジトリのセットアップ

### 3-1. クローン

```bash
git clone <REPOSITORY_URL>
cd superset
```

### 3-2. .env の設定

`.env` ファイルに以下を設定します。

**必須設定:**

```env
SUPERSET_ROOT_PATH=/path/to/superset  # リポジトリのルートパス
SKIP_ENV_VALIDATION=1                  # 環境変数バリデーションをスキップ
NO_SANDBOX=1                         # Electronサンドボックスを無効化（Linux必須）
```

**Neonを使用する場合:**

```env
NEON_ORG_ID=<YOUR_ORG_ID>
NEON_PROJECT_ID=<YOUR_PROJECT_ID>
```

**SQLiteのみで動かす場合:**

```env
NEON_ORG_ID=
NEON_PROJECT_ID=
```

### 3-3. セットアップスクリプト実行

```bash
export SUPERSET_ROOT_PATH="$PWD"
bash .superset/setup.sh
```

#### SQLiteのみモード（Neon Free / 未設定）

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

#### Neon Proモード（Electric SQL有効）

```
📊 Setup Summary
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Skipped steps:
  - Seed local DB (no source DB)
  - Seed auth token (no source token)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

## 4. 開発サーバーの起動

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
| Wrangler | 3013 | http://localhost:3013 | Electric SQLプロキシワーバー |

### 初回ビルド

初回起動時はネイティブモジュールのビルドとElectronのビルドが行われるため、約30〜60秒かかります。

### アプリの終了

Ctrl+C で全サービスが停止します。ポートが残った場合は:

```bash
fuser -k 3000/tcp 3001/tcp 3005/tcp 3010/tcp 3013/tcp
```

## 5. 変更・追加ファイル一覧

コミット `96907486` で変更・追加した全ファイルのリストです。

### 5-1. 変更ファイル（既存コードの修正）

| ファイル | 種類 | 説明 |
|---|---|---|
| `.superset/setup.sh` | セットアップスクリプト | Ubuntu版ステップファイル（`steps-ubuntu.sh`）が存在する場合に自動的に読み込むように変更。Mac版（`steps.sh`）はフォールバックとして使用 |
| `.superset/teardown.sh` | ティアダウンススクリプト | セットアップと同様に、Ubuntu版ティアダウンステップファイルの自動検出を追加 |
| `.superset/lib/setup/steps.sh` | Mac版セットアップステップ | neonctl, docker, caddy を必須から警告に変更（`error` → `warn`）。Neon分支未設定時はスキップ（`error` → `warn`）。Electric SQL未設定時は自動スキップ。`SKIP_ELECTRIC=1` 環境変数に対応 |
| `.superset/teardown.sh` | （Mac版ティアダウン） | `.superset/lib/teardown/steps.sh` は変更なし。対応するUbuntu版は `.superset/lib/teardown/steps-ubuntu.sh` を新規作成 |
| `CLAUDE.md` | プロジェクト指示 | Ubuntu 24.04 LTS への移植指示を記載 |
| `apps/api/src/app/api/electric/[...path]/route.ts` | APIルート | Electric SQLのプロキシルート。`ELECTRIC_URL`または`ELECTRIC_SECRET`が未設定の場合、503エラーを返すガードを追加。未設定時のクラッシュを防止 |
| `apps/desktop/src/lib/electron-app/factories/app/setup.ts` | Electronメインプロセス | Linux環境で `--no-sandbox` と `ELECTRON_ENABLE_LOGGING=1` を自動設定。AppArmorのuser namespace制限によりSUID sandboxが動かない問題に対応 |

### 5-2. 追加ファイル（新規作成）

| ファイル | 種類 | 説明 |
|---|---|---|
| `.superset/lib/setup/steps-ubuntu.sh` | セットアップステップ | Mac版 `steps.sh` をベースにUbuntu対応。パッケージマネージャーをbrew→apt-getに変更。Neon/Docker/Electric SQLをオプショナル化。Caddyfile生成で`auto_https off`を追加。`.env`コピー時に同一パスの場合の対応を追加 |
| `.superset/lib/teardown/steps-ubuntu.sh` | ティアダウンスステップ | Mac版 `steps.sh` をベースにUbuntu対応。Dockerコンテナの停止/削除処理をUbuntu環境に対応 |
| `docs/porting-ubuntu.md` | ドキュメント | このファイル。Mac→Ubuntuの移植手順と変更内容の記録 |
| `docs/ubuntu-install.md` | ドキュメント | Ubuntu 24.04 LTSのインストールから初期設定までの手順 |
| `docs/ubuntu-operations.md` | ドキュメント | Macユーザー向けのUbuntu基本操作マニュアル |

### 5-3. `.env` ファイルの変更

`.env` ファイルはgitにコミットされていませんが、以下の値を設定する必要があります。

| 設定値 | 必須度 | 説明 |
|---|---|---|
| `SKIP_ENV_VALIDATION=1` | 必須 | APIサーバーの環境変数バリデーションをスキップ。未設定だと起動不可 |
| `NO_SANDBOX=1` | 必須 | Electronのサンドボックスを無効化。未設定だとLinuxでElectronが起動不可 |
| `NEON_PROJECT_ID` | 任意 | NeonプロジェクトID。未設定時はSQLiteのみモードで動作 |
| `NEON_ORG_ID` | 任意 | Neon組織ID。未設定可 |
| `SUPERSET_ROOT_PATH` | 起動時必須 | リポジトリのルートパス。`export` または`.env`に設定 |

### 5-4. `chrome-sandbox` ファイルの無効化

Electronのサンドボックスバイナリはリネームして無効化しています。この変更はコミットされていません（インストール済みパッケージ内のファイルのため）。

```
node_modules/.bun/electron@40.8.5/node_modules/electron/dist/chrome-sandbox
→ chrome-sandbox.disabled にリネーム
```

`--no-sandbox` フラグにより代替パスが使用されるため、このファイルは不要です。

### 5-5. 主要な変更点のまとめ

1. **プラットフォーム検出**: Mac(brew)とUbuntu(apt-get)の自動切り替え
2. **依存関係のオプショナル化**: neonctl, docker, caddyがなくてもセットアップ可能
3. **Neon分支のスキップ**: `NEON_PROJECT_ID`未設定時はSQLiteのみで動作
4. **Electric SQLの自動スキップ**: Neon分支がない場合は自動的にスキップ
5. **Electronサンドボックス**: Linuxでは`--no-sandbox`を自動設定（`NO_SANDBOX=1`環境変数 + コードレベル）
6. **Caddyfile**: Linux用に`auto_https off`を追加（ポート80へのバインドを回避）

## 6. トラブルシューティング

### Electronが起動しない（Sandboxエラー）

```
FATAL:sandbox/linux/suid/client/setuid_sandbox_host.cc:166
```

**対策**: `NO_SANDBOX=1 bun run dev` で起動、または `.env` に `NO_SANDBOX=1` を設定。

### ポートが使用中（EADDRINUSE）

```bash
fuser -k 3000/tcp 3001/tcp 3005/tcp 3010/tcp 3013/tcp
```

### Caddyがポート80でバインドエラー

`.superset/lib/setup/steps-ubuntu.sh` のCaddyfile生成部分に `auto_https off` が設定されていることを確認してください。

### Electric SQLが60秒以内に起動しない

Neon Freeプランでは論理レプリケーションが無効なため、Electric SQLは動作しません。SQLiteのみモードで使用してください。
