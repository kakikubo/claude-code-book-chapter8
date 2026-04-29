# リポジトリ構造定義書 (Repository Structure Document)

## プロジェクト構造

```
claude-code-book-chapter8/       # TaskCLI プロジェクトルート
├── src/                         # ソースコード
│   ├── cli/                     # CLIレイヤー（コマンド定義・表示）
│   ├── services/                # サービスレイヤー（ビジネスロジック）
│   ├── types/                   # 型定義（エラークラス含む）
│   └── index.ts                 # エントリーポイント（#!/usr/bin/env node）
├── tests/                       # テストコード
│   ├── unit/                    # ユニットテスト
│   ├── integration/             # 統合テスト
│   └── e2e/                     # E2Eテスト
├── docs/                        # プロジェクトドキュメント
│   ├── ideas/                   # アイデア・壁打ちメモ
│   ├── product-requirements.md  # プロダクト要求定義書
│   ├── functional-design.md     # 機能設計書
│   ├── architecture.md          # アーキテクチャ設計書
│   ├── repository-structure.md  # 本ドキュメント
│   ├── development-guidelines.md# 開発ガイドライン
│   └── glossary.md              # 用語集
├── .claude/                     # Claude Code 設定
│   ├── commands/                # スラッシュコマンド定義
│   ├── skills/                  # タスクモード別スキル
│   └── agents/                  # サブエージェント定義
├── .steering/                   # 作業単位のドキュメント（Git 管理外、`.gitkeep` のみ追跡）
├── .husky/                      # Git フック（コミット前 lint）
├── .task/                       # タスクデータ（実行時に自動生成、デフォルトでは Git 管理対象）
│   ├── tasks.json               # タスクデータ本体
│   └── tasks.json.bak           # 直前バックアップ
├── dist/                        # tsc ビルド成果物（自動生成、`.gitignore` 済み）
│   └── index.js                 # `package.json` の "bin" / "main" エントリーポイント
├── coverage/                    # `npm run test:coverage` の出力（自動生成、`.gitignore` 済み）
├── CLAUDE.md                    # Claude Code 向けプロジェクト指示
├── README.md                    # プロジェクト概要
├── package.json                 # npm 設定・スクリプト・"bin" / "main" 定義（"type": "module"）
├── package-lock.json            # 依存ロックファイル（コミット対象）
├── tsconfig.json                # TypeScript 設定（outDir: ./dist、moduleResolution: bundler）
├── vitest.config.ts             # テスト設定（tests/** を対象）
├── eslint.config.js             # ESLint 設定
├── .prettierrc                  # Prettier 設定
├── .prettierignore              # Prettier 除外設定
└── .gitignore                   # Git 除外設定
```

---

## ディレクトリ詳細

### src/ （ソースコードディレクトリ）

#### src/cli/

**役割**: CLIレイヤー。ユーザー入力の受付・バリデーション・結果の表示を担当。Commander.js を使ってコマンドを定義する。

**配置ファイル**:
- `commands/` ディレクトリ以下に各コマンドの実装ファイル
- `Formatter.ts`: テーブル・カラー表示のユーティリティ

**命名規則**:
- コマンドファイル: `PascalCase` + `Command` サフィックス（例: `AddCommand.ts`）
- サポートクラス: `PascalCase`（例: `Formatter.ts`）

**依存関係**:
- 依存可能: `src/services/`、`src/types/`
- 依存禁止: 他の `src/cli/` ファイルからのサービスクラスの直接インスタンス化

**例**:
```
src/cli/
├── commands/
│   ├── AddCommand.ts       # task add
│   ├── ListCommand.ts      # task list
│   ├── ShowCommand.ts      # task show
│   ├── StartCommand.ts     # task start
│   ├── DoneCommand.ts      # task done
│   ├── DeleteCommand.ts    # task delete
│   ├── ArchiveCommand.ts   # task archive
│   ├── SearchCommand.ts    # task search
│   ├── ImportCommand.ts    # task import --github
│   └── SyncCommand.ts      # task sync
└── Formatter.ts            # テーブル・カラー出力
```

#### src/services/

**役割**: サービスレイヤー。ビジネスロジックを実装する。CLIレイヤーからのみ呼び出される。

**配置ファイル**:
- `TaskService.ts`: タスク CRUD のビジネスロジック
- `GitService.ts`: Git ブランチ操作の自動化
- `GitHubService.ts`: GitHub API（Issues・PR）との連携
- `StorageService.ts`: JSON ファイルへの読み書き・バックアップ
- `ConfigService.ts`: 設定ファイル（`~/.taskcli/config.json`）の管理

**命名規則**:
- `PascalCase` + `Service` サフィックス（例: `TaskService.ts`）

**依存関係**:
- 依存可能: `src/types/`、Node.js 標準モジュール、npm パッケージ
- 依存禁止: `src/cli/`（UIレイヤーへの逆依存禁止）

**例**:
```
src/services/
├── TaskService.ts
├── GitService.ts
├── GitHubService.ts
├── StorageService.ts
└── ConfigService.ts
```

#### src/types/

**役割**: TypeScript の型・インターフェース定義。複数のレイヤーで共有する型を集約する。

**配置ファイル**:
- `Task.ts`: `Task`・`TaskStatus`・`TaskPriority` などのコア型定義
- `Config.ts`: 設定ファイルの型定義
- `Store.ts`: `TaskStore`（JSON ファイルのルート構造）の型定義
- `errors.ts`: カスタムエラークラス（`ValidationError` / `NotFoundError` / `GitError`）

**命名規則**:
- 型・インターフェース定義ファイル: `PascalCase`（例: `Task.ts`、`Config.ts`）
- エラークラス集約ファイル: `errors.ts`（複数エラークラスを束ねるユーティリティのため小文字）

**依存関係**:
- 依存可能: なし（型定義のみのため）
- 依存禁止: `src/cli/`、`src/services/`（循環依存を防ぐ）

**例**:
```
src/types/
├── Task.ts       # Task インターフェース・列挙型
├── Config.ts     # Config インターフェース
├── Store.ts      # TaskStore インターフェース
└── errors.ts     # ValidationError / NotFoundError / GitError
```

#### src/index.ts

**役割**: CLI エントリーポイント。Commander.js のルートプログラムを生成し、各コマンドを登録して実行する。

**npm パッケージとしての配置**:
- 先頭にシェバン `#!/usr/bin/env node` を記述
- `tsc` で `dist/index.js` にコンパイルされ、`package.json` の `"bin": { "task": "dist/index.js" }` および `"main": "dist/index.js"` から参照される
- `npm install -g taskcli` 後は `task` コマンドとして利用可能（詳細は [architecture.md](./architecture.md) のビルド・配布節を参照）

```typescript
// src/index.ts の構成イメージ
#!/usr/bin/env node
import { program } from 'commander';
import { registerAddCommand } from './cli/commands/AddCommand';
// ... 他のコマンド登録
program.parse(process.argv);
```

---

### tests/ （テストディレクトリ）

#### tests/unit/

**役割**: ユニットテスト。外部依存（ファイルシステム・Git・GitHub API）はモックを使用。

**構造**:
```
tests/unit/
├── services/
│   ├── TaskService.test.ts
│   ├── GitService.test.ts        # toBranchName のロジックテスト
│   ├── GitHubService.test.ts     # Issue → Task マッピングのテスト
│   ├── ConfigService.test.ts     # トークン保存・読み込み・マスク表示
│   └── StorageService.test.ts
└── cli/
    └── Formatter.test.ts
```

**命名規則**:
- `[テスト対象ファイル名].test.ts`（例: `TaskService.ts` → `TaskService.test.ts`）

#### tests/integration/

**役割**: 統合テスト。一時ディレクトリを作成し、実際のファイル I/O を使ってサービス間の連携を確認する。

**構造**:
```
tests/integration/
└── task-crud.test.ts     # add → list → show → done の一連のデータフロー
```

**命名規則**:
- `[機能名].test.ts`（例: `task-crud.test.ts`）

#### tests/e2e/

**役割**: E2Eテスト。子プロセスで CLI コマンドを実際に実行し、出力を検証する。

**構造**:
```
tests/e2e/
├── user-workflow.test.ts   # MVPフロー: add → start → done
└── error-cases.test.ts     # バリデーションエラー・不正IDの処理
```

**命名規則**:
- `[シナリオ名].test.ts`（例: `user-workflow.test.ts`）

---

### docs/ （ドキュメントディレクトリ）

**配置ドキュメント**:
- `ideas/`: 壁打ち・ブレインストーミングの成果物（自由形式）
- `product-requirements.md`: プロダクト要求定義書（PRD）
- `functional-design.md`: 機能設計書
- `architecture.md`: アーキテクチャ設計書（技術仕様書）
- `repository-structure.md`: 本ドキュメント
- `development-guidelines.md`: 開発ガイドライン・コーディング規約
- `glossary.md`: ユビキタス言語・用語集

---

### 自動生成ディレクトリ

| ディレクトリ | 生成タイミング | Git 管理 | 役割 |
|------------|-------------|---------|------|
| `dist/` | `npm run build` / `npm run dev` | 除外（`.gitignore` 済み） | TypeScript コンパイル成果物。`package.json` の `"main"` / `"bin"` のターゲット |
| `coverage/` | `npm run test:coverage` | 除外（`.gitignore` 済み） | Vitest のカバレッジレポート |
| `.task/` | `task add` 等の初回実行時 | デフォルトでは含める | タスクデータ（`tasks.json`・`tasks.json.bak`）。チーム共有を想定するため、個人利用で除外したい場合は各自で `.gitignore` に追加 |
| `node_modules/` | `npm install` | 除外（`.gitignore` 済み） | npm 依存関係 |

---

### .steering/ （作業単位のドキュメント）

**役割**: 特定の開発作業における「今回何をするか」を定義。

**構造**:
```
.steering/
└── [YYYYMMDD]-[タスク名]/
    ├── requirements.md   # 今回の作業の要求内容
    ├── design.md         # 変更内容の設計
    └── tasklist.md       # タスクリスト・進捗管理
```

**命名規則**: `20250115-add-user-auth` 形式

---

## ファイル配置規則

### ソースファイル

| ファイル種別 | 配置先 | 命名規則 | 例 |
|------------|--------|---------|-----|
| コマンド定義 | `src/cli/commands/` | `PascalCase` + `Command.ts` | `AddCommand.ts` |
| 表示ユーティリティ | `src/cli/` | `PascalCase.ts` | `Formatter.ts` |
| サービスクラス | `src/services/` | `PascalCase` + `Service.ts` | `TaskService.ts` |
| 型定義 | `src/types/` | `PascalCase.ts` | `Task.ts` |
| エントリーポイント | `src/` | `index.ts` | `index.ts` |

### テストファイル

| テスト種別 | 配置先 | 命名規則 | 例 |
|-----------|--------|---------|-----|
| ユニットテスト | `tests/unit/[レイヤー]/` | `[対象].test.ts` | `TaskService.test.ts` |
| 統合テスト | `tests/integration/` | `[機能].test.ts` | `task-crud.test.ts` |
| E2Eテスト | `tests/e2e/` | `[シナリオ].test.ts` | `user-workflow.test.ts` |

### 設定ファイル

| ファイル種別 | 配置先 | 命名規則 | 備考 |
|------------|--------|---------|------|
| TypeScript 設定 | プロジェクトルート | `tsconfig.json` | `outDir: ./dist`、`moduleResolution: bundler`（拡張子省略を許容、Node.js ESM 互換） |
| テスト設定 | プロジェクトルート | `vitest.config.ts` | `tests/**` をテスト対象に向ける（`tsconfig.json` の `include` は `src/**` のみのため） |
| ESLint 設定 | プロジェクトルート | `eslint.config.js` | Flat Config 形式、ignores に `dist/**`・`.steering/**` を含む |
| Prettier 設定 | プロジェクトルート | `.prettierrc` | `.prettierignore` で `docs/`・`.claude/` 等を除外 |
| npm パッケージ設定 | プロジェクトルート | `package.json` | `"type": "module"`（ESM）、`"bin"` に CLI エントリーポイント定義 |

---

## 命名規則

### ディレクトリ名

- **レイヤーディレクトリ**: 複数形・kebab-case（例: `commands/`、`services/`）
- **機能サブディレクトリ**: 単数形・kebab-case（例: `task-management/`）
- **特殊ディレクトリ**: ドット始まり（例: `.claude/`、`.steering/`、`.task/`）

### ファイル名

- **クラスファイル**: PascalCase + 役割サフィックス（例: `TaskService.ts`、`AddCommand.ts`）
- **関数ファイル**: camelCase（例: `formatDate.ts`）
- **型定義ファイル**: PascalCase（例: `Task.ts`）
- **設定ファイル**: ツール規約に従う（例: `tsconfig.json`、`eslint.config.js`）
- **テストファイル**: `[対象].test.ts` 形式

---

## 依存関係のルール

### レイヤー間の依存

```
src/cli/ → src/services/（TaskService / GitService / GitHubService） ✅
src/cli/ → src/services/ConfigService（task config から直接呼び出し可） ✅
src/cli/ → src/types/                                   ✅
src/services/ → src/services/（StorageService / ConfigService 経由） ✅
src/services/ → src/types/                              ✅
src/services/ → src/cli/                                ❌ 禁止
src/cli/ → ファイルシステム直接アクセス                  ❌ 禁止
src/services/ → ファイルシステム直接アクセス             ❌ 禁止（StorageService / ConfigService 経由）
```

**補足**: 永続化（ファイル I/O）は `StorageService`（タスクデータ）と `ConfigService`（設定）に集約する。詳細は [architecture.md](./architecture.md) の「データアクセス境界」を参照。

### 循環依存の禁止

サービス間で循環が生じる場合は、共通の型を `src/types/` に抽出して解決する。

---

## 除外設定

### .gitignore（実ファイル準拠）

```
# 一時ファイル
tmp/

# Node.js / ログ
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# ビルド成果物
dist/
build/
*.tsbuildinfo

# 環境変数
.env
.env.local
.env.*.local

# 各種ログ
logs/
*.log

# OS / IDE
.DS_Store
Thumbs.db
.vscode/
.idea/
*.swp
*.swo
*~

# テスト出力
coverage/
.nyc_output/

# Steering files（作業単位ドキュメント。.gitkeep のみ追跡）
.steering/*
!.steering/.gitkeep

# Claude Code のローカル設定
.claude/settings.local.json
```

**`.task/` の扱い**: デフォルトでは `.gitignore` に含めない（プロジェクト内データはチーム共有を想定）。個人で Git に含めたくない場合は各自の判断で `.gitignore` に追加する。

### .prettierignore（実ファイル準拠）

```
# ドキュメント・設定ディレクトリ
.claude/
.steering/
docs/
CLAUDE.md

# ログファイル
*log.json

# 依存関係 / ビルド成果物
node_modules/
dist/
```

### ESLint ignores（`eslint.config.js` 準拠）

```
node_modules/**
dist/**
.steering/**
```
