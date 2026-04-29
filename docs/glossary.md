# プロジェクト用語集 (Glossary)

## 概要

このドキュメントは、TaskCLI プロジェクト内で使用される用語の定義を管理します。

**更新日**: 2026-04-29

---

## ドメイン用語

### タスク (Task)

**定義**: ユーザーが完了すべき作業の単位。

**説明**: TaskCLI における管理の基本単位。タイトル・ステータス・優先度・期限・紐付くGitブランチなどの属性を持つ。タスクは `task add` コマンドで作成され、`open` ステータスから始まる。

**関連用語**: [タスクステータス](#タスクステータス-task-status)、[タスクID](#タスクid-task-id)、[ブランチ連携](#ブランチ連携-branch-integration)

**使用例**:
- 「タスクを追加する」: `task add "ユーザー認証機能の実装"` でタスクを新規登録する
- 「タスクを開始する」: `task start 1` でステータスを `in_progress` に変更しGitブランチを作成する
- 「タスクを完了する」: `task done 1` でステータスを `completed` に変更する

**英語表記**: Task

**データモデル**: `src/types/Task.ts`

---

### タスクステータス (Task Status)

**定義**: タスクの進行状態を示す4段階の値。

**取りうる値**:

| ステータス | 意味 | 遷移条件 | 次の状態 |
|----------|------|---------|---------|
| `open` | 未着手 | タスク作成時の初期状態 | `in_progress`、`archived` |
| `in_progress` | 作業中 | `task start <id>` を実行 | `completed` |
| `completed` | 完了 | `task done <id>` を実行 | `archived` |
| `archived` | アーカイブ | `task archive <id>` を実行 | （終端） |

**状態遷移図**:

```mermaid
stateDiagram-v2
    [*] --> open: task add
    open --> in_progress: task start
    in_progress --> completed: task done
    completed --> archived: task archive
    open --> archived: task archive
    in_progress --> open: task reopen（P2・将来実装）
    completed --> open: task reopen（P2・将来実装）
```

**ビジネスルール**:
- `open` → `completed` への直接遷移は禁止（`task start` を経由する必要がある）
- `archived` 状態のタスクはそれ以上のステータス変更は行わない
- `task list` はデフォルトで `archived` を除外して表示する
- `task reopen`（P2・将来実装）は `in_progress` または `completed` から `open` に戻すために使用する

**英語表記**: Task Status

---

### タスクID (Task ID)

**定義**: タスクを一意に識別するUUID v4 形式の識別子。

**説明**: `task add` 実行時に自動生成される。コマンドで参照する際は先頭8文字を使用可能。リスト表示では先頭8文字を表示する。

**使用例**:
- `task show a1b2c3d4` — IDの先頭8文字でタスクを指定

**英語表記**: Task ID

---

### 優先度 (Priority)

**定義**: タスクの重要度を示す3段階の指標。

**取りうる値**:

| 値 | 意味 | 判断基準 |
|----|------|---------|
| `high` | 高 | 緊急かつ重要。期限が近い、または他の作業をブロックしている |
| `medium` | 中 | 重要だが緊急ではない（デフォルト値） |
| `low` | 低 | 重要度・緊急度ともに低い。時間があれば対応 |

**使用例**:
```bash
task add "セキュリティ脆弱性の修正" --priority high
task list --sort priority
```

**英語表記**: Priority

---

### ブランチ連携 (Branch Integration)

**定義**: タスクとGitブランチを1対1で紐付け、タスク開始時にブランチを自動作成・チェックアウトする機能。

**説明**: `task start <id>` を実行すると、`{prefix}/task-{id}-{slug}` 形式のGitブランチが自動作成される。ブランチ名はタスクタイトルを英数字・ハイフンに正規化したもの（slug）を使用する。

**ブランチ命名規則**:
- 形式: `feature/task-{id}-{title-slug}`
- 例: タイトル「ユーザー認証機能の実装」→ `feature/task-1-user-authentication`
- プレフィックス（`feature`）は設定で変更可能

**関連用語**: [スラッグ](#スラッグ-slug)

**英語表記**: Branch Integration

---

### スラッグ (Slug)

**定義**: タスクタイトルをGitブランチ名として使用できる形式に正規化した文字列。

**変換ルール**:
1. 日本語・特殊文字を対応する英語または省略形に変換
2. スペースと記号をハイフンに置換
3. 小文字化
4. 連続ハイフンを単一ハイフンに統一
5. 先頭・末尾のハイフンを除去
6. 最大50文字に切り詰め

**使用例**:
- `"ユーザー認証機能の実装"` → `user-authentication`
- `"Fix login bug"` → `fix-login-bug`

**英語表記**: Slug

---

### コンテキストスイッチ (Context Switch)

**定義**: 開発中にターミナルからGUIツール（ブラウザ、Trelloなど）へ切り替えること。

**説明**: コンテキストスイッチは集中力を途切れさせ、開発効率を低下させる。TaskCLI はこのコンテキストスイッチを排除し、ターミナル内でタスク管理を完結させることを目的とする。

**英語表記**: Context Switch

---

### ステアリングファイル (Steering File)

**定義**: 特定の開発作業のために作成する一時的なドキュメントセット。

**説明**: `.steering/[YYYYMMDD]-[タスク名]/` ディレクトリに配置される。`requirements.md`（要求内容）・`design.md`（実装アプローチ）・`tasklist.md`（進捗管理）の3ファイルで構成される。

**関連用語**: [永続ドキュメント](#永続ドキュメント-persistent-document)

**ディレクトリ構造**:
```
.steering/
└── 20250115-add-task-priority/
    ├── requirements.md
    ├── design.md
    └── tasklist.md
```

**英語表記**: Steering File

---

### 永続ドキュメント (Persistent Document)

**定義**: プロジェクト全体を通じて保持される設計・仕様ドキュメント。

**説明**: `docs/` ディレクトリに配置される。PRD・機能設計書・アーキテクチャ設計書・リポジトリ構造・開発ガイドライン・用語集の6種類。ステアリングファイルと異なり、削除せず継続的に更新する。

**関連用語**: [ステアリングファイル](#ステアリングファイル-steering-file)

**英語表記**: Persistent Document

---

## 技術用語

### Commander.js

**定義**: Node.js 向けのCLIフレームワーク。コマンドの定義・引数パース・ヘルプ生成を担う。

**本プロジェクトでの用途**: `src/index.ts` でルートプログラムを作成し、各コマンドを登録する。

**バージョン**: ^12.0.0

**選定理由**: 学習コストが低く機能十分。Node.js CLI の定番ライブラリ。

**関連ドキュメント**: [アーキテクチャ設計書](./architecture.md)

---

### simple-git

**定義**: Node.js からGitを操作するための高水準ラッパーライブラリ。

**本プロジェクトでの用途**: `GitService` でブランチ作成・チェックアウト・リポジトリ存在確認に使用する。

**バージョン**: ^3.0.0

**選定理由**: シェルインジェクションのリスクなく安全にGit操作できる。生のシェルコマンド実行より安全。

**設定ファイル**: `src/services/GitService.ts`

---

### Octokit

**定義**: GitHub REST API の公式 Node.js クライアントライブラリ。

**本プロジェクトでの用途**: `GitHubService` でGitHub Issues のインポート・同期・PR作成に使用する。

**バージョン**: `@octokit/rest` ^21.0.0

**関連ドキュメント**: [機能設計書 GitHubService](./functional-design.md)

---

### Vitest

**定義**: Vite ベースの高速なTypeScript対応テストフレームワーク。

**本プロジェクトでの用途**: ユニットテスト・統合テスト・E2Eテストの実行。カバレッジ計測（`@vitest/coverage-v8`）も使用。

**バージョン**: ^2.0.0

**選定理由**: TypeScript と ESM をネイティブサポートし、設定が最小限で済む。既存の `package.json` に含まれていた。

**設定ファイル**: `vitest.config.ts`

---

### Husky

**定義**: Gitフックを簡単に設定するためのツール。コミット前に自動でコマンドを実行する。

**本プロジェクトでの用途**: `pre-commit` フックで以下の 2 段階を実行し、品質を保つ。

1. `npx lint-staged` — 変更ファイル単位で ESLint + Prettier を適用
2. `npm run typecheck` — リポジトリ全体の `tsc --noEmit` による型チェック

**バージョン**: ^9.0.0

**設定ファイル**: `.husky/pre-commit`

**関連ドキュメント**: [開発ガイドライン](./development-guidelines.md)（自動化セクション）

---

### chalk

**定義**: ターミナルに色付きテキストを出力するための Node.js ライブラリ。

**本プロジェクトでの用途**: `Formatter` でステータスごとのカラーコーディング（`in_progress` 黄色 / `completed` 緑 / 期限超過の赤など）に使用する。

**バージョン**: ^5.0.0

**選定理由**: ESM 対応済み、軽量でシンプルな API。

**関連用語**: [Formatter](#formatter)

---

### cli-table3

**定義**: ターミナルにテーブル形式の表を描画する Node.js ライブラリ。Unicode 罫線対応・列幅自動調整が可能。

**本プロジェクトでの用途**: `task list` の一覧表示で、ID・ステータス・タイトル・ブランチをテーブルとして整形する。

**バージョン**: ^0.6.0

**関連用語**: [Formatter](#formatter)

---

### Formatter

**定義**: CLI レイヤーの表示ユーティリティ。タスクのテーブル表示・詳細表示・成功/警告/エラーメッセージの整形を担う。

**本プロジェクトでの実装**: `src/cli/Formatter.ts`

**主な責務**:
- `displayTable(tasks)` — タスク一覧をテーブル形式で表示（cli-table3）
- `displayTask(task)` — タスク詳細を表示
- カラーコーディング — chalk を使ったステータス・期限警告の色分け
- メッセージプレフィックス — `[OK]` / `[WARNING]` / `[ERROR]` の整形

**関連用語**: [chalk](#chalk)、[cli-table3](#cli-table3)、[CLI](#cli)

---

## 略語・頭字語

### CLI

**正式名称**: Command Line Interface

**意味**: コマンドラインから文字入力で操作するインターフェース。

**本プロジェクトでの使用**: TaskCLI のメインインターフェース。ユーザーは `task add "タスク"` のようなコマンドでタスクを操作する。

**実装**: `src/cli/` ディレクトリ

---

### CRUD

**正式名称**: Create, Read, Update, Delete

**意味**: データの基本操作4種（作成・読み取り・更新・削除）。

**本プロジェクトでの使用**: タスクの基本操作（`task add`・`task show`・`task done`・`task delete`）を指す場面で使用する。

---

### ESM

**正式名称**: ECMAScript Modules

**意味**: ECMAScript 標準のモジュール形式。`import` / `export` 構文を使用する。CommonJS（`require` / `module.exports`）に代わる現代的な形式。

**本プロジェクトでの使用**: `package.json` の `"type": "module"` で ESM を有効化している。`tsconfig.json` の `module: "ESNext"`・`moduleResolution: "bundler"` と併用する。

**関連設定**:
- `package.json`: `"type": "module"`
- `tsconfig.json`: `"module": "ESNext"`、`"moduleResolution": "bundler"`（拡張子省略を許容）

**関連ドキュメント**: [リポジトリ構造定義書](./repository-structure.md)、[アーキテクチャ設計書](./architecture.md)

---

### MVP

**正式名称**: Minimum Viable Product

**意味**: 最小限の機能で成立するプロダクト。

**本プロジェクトでの使用**: P0 優先度の機能（タスクCRUD・ステータス管理・ブランチ連携・一覧表示）を指す。

---

### PAT

**正式名称**: Personal Access Token

**意味**: GitHubへのアクセス権限を持つ個人用トークン。

**本プロジェクトでの使用**: `GitHubService` がGitHub APIを呼び出す際の認証に使用。`~/.taskcli/config.json` に保存（パーミッション 600）。

---

### PRD

**正式名称**: Product Requirements Document

**意味**: プロダクト要求定義書。プロダクトの目的・ターゲットユーザー・機能要件・非機能要件を定義する文書。

**本プロジェクトでの使用**: `docs/product-requirements.md`

---

### UUID

**正式名称**: Universally Unique Identifier

**意味**: グローバルに一意な識別子。Version 4はランダム生成。

**本プロジェクトでの使用**: タスクIDの生成に使用（`uuid` ライブラリの `randomUUID()`）。

---

## アーキテクチャ用語

### レイヤードアーキテクチャ (Layered Architecture)

**定義**: システムを役割ごとに複数の層（レイヤー）に分割し、上位層から下位層への一方向の依存関係を持たせる設計パターン。

**本プロジェクトでの適用**: 3層構造を採用している。

```
CLIレイヤー (src/cli/)                     ← ユーザー入力・表示
    ↓
サービスレイヤー (src/services/)            ← ビジネスロジック
    ↓
データアクセス境界 (StorageService /        ← ファイル I/O 集約
                   ConfigService)
```

**メリット**: 関心の分離・各レイヤーを独立してテスト可能・変更の影響範囲が限定的

**依存関係のルール**:
- ✅ CLIレイヤー → サービスレイヤー（TaskService / GitService / GitHubService）
- ✅ CLIレイヤー → ConfigService（`task config` コマンドからの直接呼び出しを許可）
- ✅ サービスレイヤー → データアクセス境界（StorageService / ConfigService 経由）
- ❌ サービスレイヤー → CLIレイヤー（禁止）
- ❌ CLIレイヤー → ファイルシステム直接アクセス（禁止）
- ❌ サービスレイヤー → ファイルシステム直接アクセス（禁止、StorageService / ConfigService 経由のみ）

**関連用語**: [データアクセス境界](#データアクセス境界-data-access-boundary)、[サービスレイヤー](#サービスレイヤー-service-layer)、[IStorageService](#istorageservice)

**関連ドキュメント**: [アーキテクチャ設計書](./architecture.md)、[リポジトリ構造定義書](./repository-structure.md)

---

### サービスレイヤー (Service Layer)

**定義**: ビジネスロジックを担うレイヤー。CLIレイヤーからの呼び出しを受け、データアクセス境界を通じてデータを操作する。

**本プロジェクトでの実装**:

| クラス | 種別 | 役割 |
|------|------|------|
| `TaskService` | ビジネスロジック | タスク CRUD・ステータス遷移 |
| `GitService` | ビジネスロジック | Git ブランチ操作（simple-git ラッパー） |
| `GitHubService` | ビジネスロジック | GitHub API 連携（Issues / PR） |
| `StorageService` | データアクセス境界 | `.task/tasks.json` の読み書き・バックアップ |
| `ConfigService` | データアクセス境界 | `~/.taskcli/config.json` の管理 |

**配置上の注意**: `StorageService` と `ConfigService` は物理的にはサービスレイヤー（`src/services/`）に配置されるが、論理的には永続化を担う「データアクセス境界」として位置付けられる（[データアクセス境界](#データアクセス境界-data-access-boundary) 参照）。

**英語表記**: Service Layer

---

### データアクセス境界 (Data Access Boundary)

**定義**: ファイルシステム I/O を集約する責務を持つクラス群。永続化の関心事をここに閉じ込め、他のサービスから `fs` モジュールを直接呼び出すことを禁止する。

**本プロジェクトでの該当クラス**:
- `StorageService` — タスクデータ（`.task/tasks.json`）の読み書き・バックアップ
- `ConfigService` — 設定ファイル（`~/.taskcli/config.json`）の読み書き、パーミッション 600 の強制

**設計意図**:
- 永続化処理を 2 クラスに集約することで、将来 SQLite に移行する際の影響範囲を限定する
- ビジネスロジック（タスク状態遷移・バリデーション等）はここに置かない
- `IStorageService` 抽象を満たす実装クラスを差し替えるだけで、ストレージ方式の切り替えを可能にする

**配置**: `src/services/StorageService.ts`、`src/services/ConfigService.ts`

**関連用語**: [サービスレイヤー](#サービスレイヤー-service-layer)、[IStorageService](#istorageservice)、[TaskStore](#taskstore)

**関連ドキュメント**: [アーキテクチャ設計書](./architecture.md)（データ永続化戦略・スケーラビリティ設計）

**英語表記**: Data Access Boundary

---

### IStorageService

**定義**: タスクデータの永続化を抽象化するインターフェース。具象クラス（`JsonStorageService` など）はこの契約を満たす実装を提供する。

**シグネチャ**:

```typescript
interface IStorageService {
  load(): TaskStore;
  save(store: TaskStore): void;
  backup(): void;
}
```

**設計意図**: 将来の SQLite 移行を容易にするため、ストレージ実装をインターフェースで抽象化する。MVP では `JsonStorageService` のみを実装し、移行時には `SqliteStorageService` を新規実装して差し替える。

**現在の実装**: `JsonStorageService implements IStorageService`（`src/services/StorageService.ts`）

**テストでの活用**: ユニットテストでは `IStorageService` 型のモックを作成し、ファイル I/O を発生させずにビジネスロジックを検証する（[development-guidelines.md](./development-guidelines.md) のモック使用方針参照）。

**関連用語**: [データアクセス境界](#データアクセス境界-data-access-boundary)、[TaskStore](#taskstore)

**英語表記**: IStorageService

---

### TaskStore

**定義**: タスクデータを永続化するJSONファイル（`.task/tasks.json`）のルートデータ構造。

**説明**: `version`（データフォーマットバージョン）と `tasks`（タスク配列）を持つ。将来のSQLiteへの移行時、または JSON フォーマット変更時にバージョン番号でマイグレーション要否を判定する。

```typescript
interface TaskStore {
  version: string;  // "1.0"
  tasks: Task[];
}
```

**マイグレーション戦略**: メジャーバージョンアップ時は `StorageService.migrate()` で旧フォーマットを変換し、変換前データを `.task/tasks.json.v[旧バージョン].bak` として退避する（[architecture.md データマイグレーション戦略](./architecture.md) 参照）。

**英語表記**: TaskStore

---

### Config

**定義**: TaskCLI の設定ファイル（`~/.taskcli/config.json`）のルートデータ構造。

**説明**: GitHub 連携情報（PAT・owner・repo）と Git ブランチ命名のデフォルトプレフィックスを保持する。ファイルパーミッションは 600（所有者のみ読み書き）に強制設定される。

```typescript
interface Config {
  github?: {
    token: string;        // Personal Access Token
    owner: string;        // GitHub ユーザー名 / Org 名
    repo: string;         // リポジトリ名
  };
  git?: {
    defaultBranchPrefix: string; // デフォルト: "feature"
  };
}
```

**保存先**: `~/.taskcli/config.json`（パーミッション 600）

**設定コマンド**: `task config --github-token <token>`、`task config --show`（トークンは先頭4文字以外をマスク表示）

**関連用語**: [PAT](#pat)、[データアクセス境界](#データアクセス境界-data-access-boundary)（ConfigService）

**英語表記**: Config

---

### task reopen（P2・将来実装）

**定義**: `in_progress` または `completed` 状態のタスクを `open` に戻すコマンド。

**用途**: レビュー差し戻しなどで作業の再開が必要になった場合に使用する。

**ステータス**: P2（Post-MVP）。MVP 範囲外のため現時点では未実装。

**コマンド**: `task reopen <id>`

**関連用語**: [タスクステータス](#タスクステータス-task-status)

**関連ドキュメント**: [プロダクト要求定義書](./product-requirements.md)（P2 機能）

---

## エラー・例外

### ValidationError

**クラス名**: `ValidationError`

**継承元**: `Error`

**発生条件**: ユーザーの入力がバリデーションに違反した場合（タイトルが空・200文字超、期限日の形式不正など）。

**エラーメッセージフォーマット**: `[フィールド名] の値が不正です: [詳細]`

**対処方法**:
- ユーザー: エラーメッセージに従って入力を修正する
- 開発者: `field` プロパティを参照して違反したフィールドを特定する

**実装箇所**: `src/types/errors.ts`

**使用例**:
```typescript
throw new ValidationError(
  'タイトルは1〜200文字で入力してください（現在: 250文字）',
  'title',
  title
);
```

---

### NotFoundError

**クラス名**: `NotFoundError`

**継承元**: `Error`

**発生条件**: 指定されたIDのタスクが `.task/tasks.json` に存在しない場合。

**エラーメッセージフォーマット**: `タスクが見つかりません (ID: {id})`

**対処方法**:
- ユーザー: `task list` で正しいタスクIDを確認する
- 開発者: IDの渡し方とストレージの整合性を確認する

**実装箇所**: `src/types/errors.ts`

---

### GitError

**クラス名**: `GitError`

**継承元**: `Error`

**発生条件**: Gitコマンドの実行に失敗した場合（ブランチ作成失敗、チェックアウト失敗など）。

**対処方法**:
- ユーザー: エラーメッセージの詳細を確認し、Gitリポジトリの状態（コンフリクト・権限など）を確認する
- 開発者: `cause` プロパティで元のエラーを参照する。タスクのステータス変更をロールバックする

**実装箇所**: `src/types/errors.ts`

---

## 索引

### あ行
- [アーカイブ（タスクステータス）](#タスクステータス-task-status)

### か行
- [コマンダー → Commander.js](#commanderjs)
- [コンテキストスイッチ](#コンテキストスイッチ-context-switch)
- [Config](#config)

### さ行
- [サービスレイヤー](#サービスレイヤー-service-layer)
- [スラッグ](#スラッグ-slug)
- [ステアリングファイル](#ステアリングファイル-steering-file)

### た行
- [タスク](#タスク-task)
- [タスクID](#タスクid-task-id)
- [タスクステータス](#タスクステータス-task-status)
- [task reopen](#task-reopenp2将来実装)
- [TaskStore](#taskstore)
- [データアクセス境界](#データアクセス境界-data-access-boundary)

### な行
- [NotFoundError → な行参照](#notfounderror)

### は行
- [バリデーションエラー → ValidationError](#validationerror)
- [フォーマッター → Formatter](#formatter)
- [ブランチ連携](#ブランチ連携-branch-integration)

### ま行
- [MVP](#mvp)

### や行
- [優先度](#優先度-priority)

### ら行
- [レイヤードアーキテクチャ](#レイヤードアーキテクチャ-layered-architecture)

### わ行
- [永続ドキュメント](#永続ドキュメント-persistent-document)

### A-Z
- [chalk](#chalk)
- [CLI](#cli)
- [cli-table3](#cli-table3)
- [Commander.js](#commanderjs)
- [Config](#config)
- [CRUD](#crud)
- [ESM](#esm)
- [Formatter](#formatter)
- [GitError](#giterror)
- [Husky](#husky)
- [IStorageService](#istorageservice)
- [NotFoundError](#notfounderror)
- [Octokit](#octokit)
- [PAT](#pat)
- [PRD](#prd)
- [simple-git](#simple-git)
- [task reopen](#task-reopenp2将来実装)
- [TaskStore](#taskstore)
- [UUID](#uuid)
- [ValidationError](#validationerror)
- [Vitest](#vitest)
