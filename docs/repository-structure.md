# リポジトリ構造定義書 (Repository Structure Document)

## プロジェクト構造

```
claude-code-book-chapter8/       # TaskCLI プロジェクトルート
├── src/                         # ソースコード
│   ├── cli/                     # CLIレイヤー（コマンド定義・表示）
│   ├── services/                # サービスレイヤー（ビジネスロジック）
│   ├── types/                   # 型定義
│   └── index.ts                 # エントリーポイント
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
├── .steering/                   # 作業単位のドキュメント（Git 管理外推奨）
├── .husky/                      # Git フック（コミット前 lint）
├── .task/                       # タスクデータ（実行時に自動生成）
│   ├── tasks.json               # タスクデータ本体
│   └── tasks.json.bak           # 直前バックアップ
├── CLAUDE.md                    # Claude Code 向けプロジェクト指示
├── README.md                    # プロジェクト概要
├── package.json                 # npm 設定・スクリプト
├── tsconfig.json                # TypeScript 設定
├── vitest.config.ts             # テスト設定
├── eslint.config.js             # ESLint 設定
├── .prettierrc                  # Prettier 設定
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

**命名規則**:
- `PascalCase`（例: `Task.ts`、`Config.ts`）

**依存関係**:
- 依存可能: なし（型定義のみのため）
- 依存禁止: `src/cli/`、`src/services/`（循環依存を防ぐ）

**例**:
```
src/types/
├── Task.ts     # Task インターフェース・列挙型
├── Config.ts   # Config インターフェース
└── Store.ts    # TaskStore インターフェース
```

#### src/index.ts

**役割**: CLI エントリーポイント。Commander.js のルートプログラムを生成し、各コマンドを登録して実行する。

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

### .task/ （実行時に自動生成）

**役割**: タスクデータの永続化。プロジェクトルートに生成される。

**注意事項**:
- `task add` など初回実行時に自動作成される
- Git 管理に含めるかどうかはチームの判断（チーム共有する場合は `.gitignore` から除外）
- デフォルトでは `.gitignore` に含める（個人のタスクデータを Git にコミットしない）

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

| ファイル種別 | 配置先 | 命名規則 |
|------------|--------|---------|
| TypeScript 設定 | プロジェクトルート | `tsconfig.json` |
| テスト設定 | プロジェクトルート | `vitest.config.ts` |
| ESLint 設定 | プロジェクトルート | `eslint.config.js` |
| Prettier 設定 | プロジェクトルート | `.prettierrc` |

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
src/cli/ → src/services/ → （外部ライブラリ・Node.js）  ✅
src/cli/ → src/types/                                   ✅
src/services/ → src/types/                              ✅
src/services/ → src/cli/                                ❌ 禁止
src/cli/ → ファイルシステム直接アクセス                  ❌ 禁止
```

### 循環依存の禁止

サービス間で循環が生じる場合は、共通の型を `src/types/` に抽出して解決する。

---

## 除外設定

### .gitignore に含めるべきファイル

```
node_modules/
dist/
.task/               # タスクデータ（チーム共有しない場合）
*.log
.DS_Store
.env
coverage/
```

### .prettierignore / ESLint の除外対象

```
dist/
node_modules/
.steering/
coverage/
```
