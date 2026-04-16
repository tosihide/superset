# Superset ソース解析の手引き書

本書は Superset モノレポのソースコードを初めて読む開発者向けの読解ガイドです。
どのファイルから読み始め、どの順序で理解を広げていけばよいかを示します。

---

## 1. 全体像

Superset は **Bun + Turborepo** で管理されるモノレポで、以下から構成されます。

- **apps/**: エンドユーザー向けアプリケーション (7 本)
- **packages/**: アプリ間で共有するライブラリ (18 本前後)
- **tooling/**: TypeScript 設定などの共通ツール

主要技術スタック:

| 領域 | 技術 |
|------|------|
| パッケージ管理 | Bun v1.3.x (`bunfig.toml` で isolated linker) |
| ビルド | Turborepo (`turbo.jsonc`) |
| フロント | Next.js 16 / React 19 / Expo 55 / Electron 40 |
| API | Next.js Route Handler + tRPC |
| DB | PostgreSQL (Neon) + Drizzle ORM、デスクトップは SQLite |
| 認証 | Better Auth v1.5 (GitHub / Google / Stripe / API Key) |
| UI | TailwindCSS v4 + Radix UI + shadcn/ui |
| Lint / Format | Biome (ルート一括実行) |

---

## 2. 読み始めるべき順序

初学者は次の順にファイルを開くと全体像が掴めます。

1. `README.md` — プロジェクトの目的
2. `AGENTS.md` / `CLAUDE.md` — 開発ルールとディレクトリ規約
3. `package.json` (root) — workspaces と主要スクリプト
4. `turbo.jsonc` — タスク依存関係とキャッシュ
5. `docs/internal/JA_REPOSITORY_GUIDE.md` — 既存の日本語ガイド
6. `packages/db/src/schema/` — データモデル (ドメインの輪郭)
7. `packages/trpc/src/root.ts` — 全 API の入り口
8. `apps/web/src/app/layout.tsx` — 代表アプリの起点

---

## 3. ルート設定ファイル

| ファイル | 役割 |
|----------|------|
| `package.json` | workspaces 定義、`dev` / `build` / `test` / `lint:fix` などのスクリプト |
| `turbo.jsonc` | タスクグラフ、キャッシュ、グローバル env |
| `biome.jsonc` | フォーマッタ / リンタ。ルートで一括実行 |
| `bunfig.toml` | Bun のインストーラ設定 (native module 対策で isolated) |
| `tooling/typescript/` | 共有 tsconfig ベース |

---

## 4. apps/ の解析ポイント

各アプリは独立した `package.json` と `src/app/` を持ちます。どのアプリも **tRPC クライアント経由で `apps/api` に到達する** 構造です (Desktop のみ IPC)。

### apps/web — メイン Web (app.superset.sh)
- エントリ: `src/app/layout.tsx` → `src/app/providers.tsx`
- tRPC クライアント: `src/trpc/client.ts` (HTTP batch link → `${API_URL}/api/trpc`)
- RSC 用サーバ caller: `src/trpc/server.tsx`

### apps/api — バックエンド
- tRPC ハンドラを `/api/trpc` にマウント
- コンテキスト生成: `src/trpc/context.ts` (user / db / session を注入)
- ここは薄く、実ロジックは `packages/trpc` に集約

### apps/admin — 管理ダッシュボード
- web と同構成。組織管理・課金・ユーザー管理に特化

### apps/desktop — Electron デスクトップ
- Main: `src/main/index.ts`
- Preload: `src/preload/index.ts`
- Renderer: `src/renderer/` (React + TanStack Router)
- tRPC は `trpc-electron` による IPC。**subscription は observable のみ** (async generator は不可)。詳細は `apps/desktop/AGENTS.md`

### apps/marketing — ランディング (superset.sh)
- Next.js + MDX + Three.js。`packages/email` (Resend) を利用

### apps/docs — ドキュメントサイト (Fumadocs)
- `src/app/(home)/`、`src/app/docs/`

### apps/mobile — Expo (iOS/Android)
- ルーティングは `src/app/` (Expo Router)、画面ロジックは `src/screens/`
- 詳細は `apps/mobile/AGENTS.md`

---

## 5. packages/ の解析ポイント

| package | 読むべきファイル | 役割 |
|---------|------------------|------|
| `ui` | `src/components/ui/*.tsx`, `src/globals.css` | shadcn/ui ベースの共有コンポーネント |
| `db` | `src/schema/index.ts`, `src/client.ts` | Drizzle スキーマと DB クライアント |
| `auth` | `src/server.ts`, `src/client.ts` | Better Auth のサーバ / クライアント設定 |
| `trpc` | `src/root.ts`, `src/router/*` | 全 tRPC ルータ定義 |
| `shared` | `src/constants.ts`, `src/auth/index.ts` | 定数、ロール判定、Zod 型 |
| `local-db` | `src/schema/index.ts` | Desktop オフライン用 SQLite スキーマ |
| `host-service` | `src/` | Desktop 用ホストサーバ (端末 / git) |
| `workspace-fs` | `src/` | OS 依存のファイルシステム抽象 |
| `chat` | `src/` | Anthropic SDK 連携、ストリーミング応答 |
| `mcp` / `desktop-mcp` | `src/index.ts`, `src/bin.ts` | Model Context Protocol 実装 |
| `email` | `src/emails/*.tsx` | React Email テンプレート |
| `panes` | `src/` | 汎用ペイン UI |
| `scripts` | `src/` | CLI ツール |
| `durable-session` | `src/` | 永続セッション管理 |

---

## 6. データベーススキーマ (`packages/db/src/schema/`)

| ファイル | 内容 |
|----------|------|
| `schema.ts` | projects / workspaces / tasks / agents / integrations 等の業務テーブル |
| `auth.ts` | users / sessions / accounts / verifications |
| `github.ts` | repositories / pull_requests / issues / webhooks |
| `ingest.ts` | ログ / 埋め込み / 分析用テーブル |
| `enums.ts` | Role / Status 等の enum |
| `relations.ts` | Drizzle relations (FK / eager load) |
| `zod.ts` | Drizzle から派生した Zod スキーマ |
| `types.ts` | `SelectUser` / `InsertProject` などの推論型 |

> 注意: `packages/db/drizzle/` のマイグレーションファイルは自動生成なので **手で編集しない**。スキーマを変更したら `bunx drizzle-kit generate --name="..."` を実行します。

---

## 7. tRPC の配線

**サーバ側 (`packages/trpc`)**
- `src/root.ts` の `appRouter` が全サブルータを束ねる (admin / agent / task / user / project / workspace / billing / organization / integration ほか)
- 各サブルータは `src/router/<domain>/index.ts` に query / mutation / subscription を定義
- コンテキストは `apps/api/src/trpc/context.ts` で構築 (認証済みユーザー・DB・組織情報)

**クライアント側**
- Web: `apps/web/src/trpc/{client.ts, react.tsx, server.tsx}`
- Desktop: `apps/desktop/src/lib/trpc/` (IPC リンク)
- Mobile: Expo 内の tRPC プロバイダ

型は `AppRouter` を `packages/trpc` から export しており、クライアントは型を自動推論します。

---

## 8. 認証 (Better Auth)

- サーバ設定: `packages/auth/src/server.ts`
  - Drizzle アダプタ、OAuth (GitHub / Google / GitHub App)、Stripe 連携、Resend メール送信
  - plugin: organization / customSession / jwt / bearer / apiKey / deviceAuthorization
- クライアント: `packages/auth/src/client.ts` — `useSession()` フックを提供
- スキーマ: `packages/db/src/schema/auth.ts`
- trustedOrigins には web / api / desktop / mobile / プレビュー URL を列挙

---

## 9. データフロー全体

```
ブラウザ / Desktop Renderer / Expo
        │
        ▼
Next.js / React Native (UI 層)
        │  useQuery / useMutation (tRPC React)
        ▼
tRPC クライアント  ── HTTP batch link ──▶ apps/api /api/trpc
                         または
                   trpc-electron IPC ──▶ Desktop main プロセス
        │
        ▼
packages/trpc (appRouter)
        │
        ├── packages/auth       (セッション検証)
        ├── packages/db         (Drizzle → Neon Postgres)
        ├── packages/workspace-fs / host-service (Desktop)
        └── 外部 API (GitHub / Linear / Slack / Stripe / Anthropic)
        │
        ▼
戻り値 → React Query キャッシュ → UI 更新
```

---

## 10. 解析の進め方 (推奨ワークフロー)

1. **目的のユースケースを 1 つ選ぶ** (例: 「タスクを作成する」)
2. UI から入る: `apps/web/src/app/...` の該当ページを検索
3. 呼び出している tRPC プロシージャ名を特定 (例: `trpc.task.create.useMutation`)
4. `packages/trpc/src/router/task/` を開き、入力バリデーションとロジックを読む
5. `packages/db/src/schema/schema.ts` で関連テーブルを確認
6. 認可が関わる場合は `packages/auth` と `packages/shared/src/auth` を参照
7. 必要なら `grep` で `appRouter` 経由の型伝播を追跡

---

## 11. よく使うコマンド

```bash
bun dev                # 全アプリを起動
bun run lint:fix       # Biome で整形 + 自動修正
bun run typecheck      # 型検査
bun test               # テスト
bun run build          # 全ビルド
```

---

## 12. 参考ドキュメント

- `README.md` — プロダクト概要
- `AGENTS.md` / `CLAUDE.md` — 開発ルール、ディレクトリ構造規約
- `docs/internal/JA_REPOSITORY_GUIDE.md` — 既存の日本語詳細ガイド
- `apps/desktop/AGENTS.md` — Desktop 固有パターン (observable subscription)
- `apps/mobile/AGENTS.md` — Mobile (Expo Router) 構成
- `CONTRIBUTING.md` — 貢献ガイド
