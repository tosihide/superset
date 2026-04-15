# Superset モノレポ — ファイル地図・ソース解析メモ・Linux ビルド手順

この文書は、本リポジトリ（Bun + Turborepo）を **ソース解析** する際の起点として使い、**Linux 上でのデスクトップアプリビルド** を再現するための手順をまとめたものです。各ファイルを 1 行ずつ列挙する代わりに、**ディレクトリ単位と主要エントリ** に整理しています（`apps/desktop` 単体で数千ファイルがあるため、全体の一覧は現実的ではありません）。

---

## 1. ファイル一覧（構成単位）と概要

### 1.1 リポジトリルート

| パス | 概要 |
|:-----|:-----|
| `package.json` | ワークスペース定義、`bun dev` / `bun build` / `bun test` などルートスクリプト。`build` は `@superset/desktop` のみ（`turbo build --filter=@superset/desktop`）。 |
| `bun.lock` | Bun のロックファイル。 |
| `turbo.jsonc` | Turborepo タスク定義（`build` の出力に `dist/**`, `release/**` 等）。 |
| `biome.json` / `biome.jsonc`（あれば） | Biome（フォーマット・リント）設定。 |
| `CLAUDE.md` / `AGENTS.md` | エージェント向け開発ガイド（構成・コマンド・規約）。 |
| `.env.example` | ルートの環境変数テンプレート。デスクトップの `electron.vite.config.ts` は `../../.env` を読む。 |
| `Caddyfile` / `Caddyfile.example` | 開発時リバースプロキシ（Electric SQL 等）。 |
| `scripts/postinstall.sh` | `sherif` と `@superset/desktop install:deps`（Electron ネイティブ再ビルド）。CI ではスキップ。 |
| `scripts/lint.sh` など | ルート品質スクリプト。 |
| `patches/` | 依存パッケージ向けパッチ（`patchedDependencies`）。 |

### 1.2 アプリ `apps/*`

| パス | 概要 |
|:-----|:-----|
| `apps/desktop/` | **Electron デスクトップアプリ**（本稿のビルド対象）。electron-vite + electron-builder。 |
| `apps/web/` | メイン Web アプリ（Next.js。`app.superset.sh` 向け）。 |
| `apps/api/` | API バックエンド。 |
| `apps/admin/` | 管理ダッシュボード。 |
| `apps/marketing/` | マーケティングサイト（`superset.sh`）。 |
| `apps/docs/` | ドキュメントサイト。 |
| `apps/mobile/` | React Native（Expo）モバイル。 |
| `apps/electric-proxy/` | Electric SQL 用プロキシ。 |
| `apps/streams/` | Streams 関連。 |

### 1.3 共有パッケージ `packages/*`

| パス | 概要 |
|:-----|:-----|
| `packages/ui/` | 共有 UI（Tailwind v4 + shadcn 系）。 |
| `packages/db/` | Drizzle スキーマ・DB 層。 |
| `packages/auth/` | 認証。 |
| `packages/trpc/` | 共有 tRPC ルータ型・定義。 |
| `packages/shared/` | 共有ユーティリティ。 |
| `packages/local-db/` | ローカル SQLite（Drizzle マイグレーション元など）。 |
| `packages/chat/` | チャット機能（デスクトップから参照）。Anthropic 等で `darwin` 分岐あり。 |
| `packages/host-service/` | ワークスペース内 HTTP/tRPC・ターミナル連携。**ターミナル・シェルは OS ごとに分岐**。 |
| `packages/workspace-client/` / `packages/workspace-fs/` | ワークスペースクライアント・ファイルシステム抽象。 |
| `packages/desktop-mcp/` | デスクトップ向け MCP。 |
| `packages/mcp/` | MCP 統合。 |
| `packages/panes/` | ペイン UI。 |
| `packages/email/` | メールテンプレート。 |
| `packages/cli/` / `packages/cli-framework/` | CLI。 |
| `packages/macos-process-metrics/` | macOS プロセスメトリクス用ネイティブ。**Linux ではビルドスキップも JS が `null` にフォールバック**。 |

### 1.4 デスクトップ `apps/desktop` — 重要ファイル・ディレクトリ

| パス | 概要 |
|:-----|:-----|
| `package.json` | `main` は `./dist/main/index.js`。`prebuild` でアイコン生成・`electron-vite build`・`copy-native-modules`・`validate-native-runtime`。`build` は `electron-builder`。 |
| `electron.vite.config.ts` | main / preload / renderer の Vite 設定。ルート `.env` を `dotenv` で読み込み、main のエントリは `src/main/index.ts` ほか（`terminal-host`, `pty-subprocess`, `git-task-worker`, `host-service`）。 |
| `electron-builder.ts` | パッケージ形式：**Linux は AppImage**（`linux.target: ["AppImage"]`）。macOS / Windows 向け設定あり。 |
| `runtime-dependencies.ts` | メインプロセスで **外部化** するネイティブ系（`better-sqlite3`, `node-pty`, `libsql`, `@parcel/watcher`, `@ast-grep/napi`, `@superset/macos-process-metrics` 等）と ASAR unpack ルール。 |
| `vite/helpers.ts` | `devPath`（通常 `dist`）へのリソースコピー等。 |
| `scripts/copy-native-modules.ts` | Bun の symlink を実体コピーに置換し、electron-builder が ASAR に乗せられるようにする。**`TARGET_PLATFORM` / `TARGET_ARCH` でターゲット指定可能**。 |
| `scripts/validate-native-runtime.ts` | ビルド後のバンドル検査と、**ホスト OS 向け** `@libsql/*`, `@parcel/watcher-*`, `@ast-grep/napi-*` の存在確認。 |
| `scripts/generate-file-icons.ts` | ビルド前アイコン生成。 |
| `scripts/clean-launch-services.ts` / `scripts/patch-dev-protocol.ts` | **macOS 向け開発補助**。`process.platform !== "darwin"` では実質スキップ。 |
| `src/main/` | Electron メインプロセス（ウィンドウ、トレイ、深いリンク、tRPC、ターミナルデーモン等）。エントリは `index.ts`。 |
| `src/preload/` | プリロード（`contextBridge`、IPC 経路）。 |
| `src/renderer/` | React UI。TanStack Router（`routes/`, `routeTree.gen.ts`）。 |
| `src/resources/` | アイコン、`build/`、ブラウザ拡張、`public/` 等の静的リソース。 |
| `src/lib/trpc/` | レンダラー側 tRPC ルータ実装の大部分。外部エディタ起動など **OS 分岐**（例: `routers/external/helpers.ts`）。 |
| `docs/`（`apps/desktop/docs/`） | 端末アーキテクチャ、外部ファイル一覧など開発者向けメモ。 |

### 1.5 補足ドキュメント（既存）

| パス | 概要 |
|:-----|:-----|
| `README.md` | プロジェクト概要・要件（OS は **macOS 推奨、Windows/Linux は未検証** と記載）。 |
| `apps/desktop/docs/EXTERNAL_FILES.md` | アプリが `~/.superset/` 等に書き込むファイル一覧。 |
| `docs/issues/linux-open-editor-fix.md` | Linux で「エディタで開く」が失敗していた問題と修正内容（`xdg-open` と `open -a` の差）。 |

---

## 2. ソース解析資料（解析時の参考となる整理）

### 2.1 実行時の三分割（Electron）

| 領域 | 技術 | 主な責務 |
|:-----|:-----|:---------|
| **Main** | Node（Electron main） | アプリライフサイクル、ネイティブ API、ターミナル PTY、`host-service` 子プロセス、自動更新。 |
| **Preload** | 分離コンテキスト | `preload/index.ts` からレンダラーへ安全な API 露出。 |
| **Renderer** | React + Vite | UI・ルーティング・状態。`src/renderer/`。 |

**読み方の目安**: 機能を追うときは「UI イベント（renderer）→ tRPC / IPC → main のどのモジュールか」を辿る。

### 2.2 ビルドパイプライン（デスクトップ）

1. `electron-vite build` → 出力は `package.json` の `main` に沿って **`dist/main`**, **`dist/preload`**, **`dist/renderer`**（`vite/helpers.ts` の `devPath` は通常 `dist`）。
2. `copy-native-modules` で `node_modules` のシンボリックリンクを解消。
3. `validate-native-runtime` でバンドルへの誤った取り込みを検査。
4. `electron-builder` で **Linux なら AppImage** などを `release/` に出力。

### 2.3 ネイティブ依存と注意点

- **Electron の Node と ABI** が一致するよう、`postinstall` / ビルド時に `@electron/rebuild` が走る（ログに `electronVersion=…` と出る）。
- **`runtime-dependencies.ts`** に列挙されたモジュールは main バンドルに埋め込まず、`require` で読み込む想定。ここを壊すと `validate-native-runtime` が失敗しやすい。
- **glibc と musl**: Linux では `@libsql/linux-x64-gnu` と `@libsql/linux-x64-musl` の両方や、`@parcel/watcher-linux-x64-glibc` / `musl` のいずれかが必要、という検証がある（Alpine 等の musl 環境では musl 側のパッケージが選ばれる想定）。

### 2.4 プラットフォーム固有の分岐（解析の手掛かり）

| 観点 | ファイル・場所の例 |
|:-----|:-------------------|
| `darwin` 専用 | `packages/chat/.../anthropic.ts`（`platform() !== "darwin"`）、`packages/host-service/.../resolveAnthropicCredential.ts` |
| ターミナル・シェル | `packages/host-service/src/terminal/*` |
| 外部エディタ起動 | `apps/desktop/src/lib/trpc/routers/external/helpers.ts`（Linux では CLI 名、`darwin` は `open -a`） |
| メトリクス | `packages/macos-process-metrics`（Linux ではネイティブ無しでも動作） |
| 開発用 plist / Launch Services | `apps/desktop/scripts/patch-dev-protocol.ts`（非 macOS ではスキップ） |

### 2.5 tRPC・ワークスペース

- デスクトップは `trpc-electron` 等で IPC 越しに tRPC 的な API を呼ぶ構成。
- `packages/trpc` と `apps/desktop/src/lib/trpc/` を対に読むと、**サーバ側手続きと UI の対応** が追いやすい。

---

## 3. Linux ビルド手順書

### 3.1 前提条件

| 項目 | 推奨 |
|:-----|:-----|
| OS | 本手順は **x86_64 Linux** で確認（glibc 系ディストリビューション）。**musl のみ**の環境では `@libsql` / `@parcel/watcher` の optional 依存の入り方が変わる。 |
| Bun | ルート `package.json` の `packageManager` に記載のバージョン（例: `bun@1.3.11`）に合わせる。 |
| ビルドツール | ネイティブモジュール（`better-sqlite3`, `node-pty` 等）のコンパイルのため、**Python 3**、**make**、**g++/clang** 等が一般的に必要。ディストリビューションによっては `build-essential`（Debian/Ubuntu 系）を入れる。 |
| ディスク・メモリ | 初回 `bun install` と `electron-vite build` は負荷が大きい。ルートの `@superset/desktop` の `compile:app` は `NODE_OPTIONS=--max-old-space-size=8192` を使用。 |

### 3.2 環境変数（.env）

1. `cp .env.example .env`
2. 手早くビルドだけ試す場合（本番用途では非推奨）: `.env` に **`SKIP_ENV_VALIDATION=1`** を追加すると、`src/main/env.main.ts` の検証が開発時緩和に使われる（README の Option B と同趣旨）。

API 向けの変数が未設定でも、デスクトップのビルド自体は上記で通ることが多い。

### 3.3 手順（デスクトップの Linux ビルド）

リポジトリルートで:

```bash
bun install
```

続けて（ルートの `package.json` の定義どおり）:

```bash
bun run build
```

これは **`turbo build --filter=@superset/desktop`** で、実質的に `@superset/desktop` の `build`（内部で `prebuild` → `electron-builder`）が実行される。

**成果物（Linux x64 の場合）**:

- `apps/desktop/release/` 配下に **`superset-<version>-x86_64.AppImage`**（`electron-builder.ts` の `linux` 設定に準拠）
- 展開用の `linux-unpacked/` 等

AppImage を実行する場合は実行権限付与が必要な場合がある: `chmod +x apps/desktop/release/superset-*.AppImage`

### 3.4 個別に段階実行したい場合

`apps/desktop` で:

```bash
bun run compile:app
bun run copy:native-modules
bun run validate:native-runtime
bun run build
```

開発サーバ（ホットリロード）を Linux で動かす場合はルートで `bun run dev`（API + web + desktop 等）。Caddy は別途 `Caddyfile` の準備が必要（README 参照）。

### 3.5 クロスビルド・環境変数（参考）

`scripts/copy-native-modules.ts` および `validate-native-runtime.ts` は **`TARGET_PLATFORM`** / **`TARGET_ARCH`** を参照する。ホストと異なるターゲット向けに資材を揃える場合に使用（完全なクロスコンパイル保証は electron-builder / 依存の制約に依存）。

### 3.6 トラブルシューティング（Linux で失敗しやすい点）

| 症状・ログの傾向 | 確認すること |
|:-----------------|:-------------|
| `node-gyp` / `g++` が見つからない | ビルド必須パッケージをインストール。 |
| `validate-native-runtime` が **@libsql** または **@parcel/watcher** のプラットフォームパッケージ不足で失敗 | **glibc と musl のどちらの環境か**。`bun install` が optional 依存を省略していないか。`copy:native-modules` を再実行。 |
| `Required materialized runtime dependency is still a symlink` | `bun run copy:native-modules` を実行してシンボリックリンクを実体に置換。 |
| `libsql` が main にバンドルされたと検出 | `runtime-dependencies.ts` と `electron.vite.config.ts` の external 設定が崩れている可能性（上流変更時に要確認）。 |
| パッケージの**公開日が新しすぎる**と、環境の「最低リリース日数」ポリシーでインストール失敗 | リポジトリの AGENTS.md の方針に従い、バージョンを下げるかポリシー管理者に相談。 |
| macOS 専用スクリプト | `predev` の plist 周りは Linux ではスキップされる前提。ビルド（`build`）経路には通常含まれない。 |

### 3.7 本環境での確認メモ（参考）

2026-04-15 時点、**Linux 6.17（x86_64）** 上で `bun install` および `bun run build` を実行し、**AppImage の生成まで成功**したログがある。ユーザ環境で失敗する場合は、上表のネイティブ依存・ツールチェーン・musl/glibc を優先的に切り分けるとよい。

---

## 変更履歴（メモ）

- 初版: リポジトリ構造・解析の起点・Linux デスクトップビルド手順を一体化した。
