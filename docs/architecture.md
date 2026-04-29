# 技術仕様書 (Architecture Design Document)

## テクノロジースタック

### 言語・ランタイム

| 技術 | バージョン | 選定理由 |
|------|-----------|----------|
| Node.js | v18以上（開発環境: v24.15.0） | 非同期I/O処理に優れ、CLIツールのランタイムとして広く普及。npm エコシステムが充実しており必要なライブラリの入手が容易 |
| TypeScript | ~5.3.0 | 静的型付けによりコンパイル時にバグを検出でき保守性が向上。IDEの補完が強力で開発効率が高い |
| npm | 11.x | Node.js に標準搭載。package-lock.json による依存関係の厳密な管理が可能 |

### フレームワーク・ライブラリ（追加予定）

| 技術 | バージョン | 用途 | 選定理由 |
|------|-----------|------|----------|
| commander | ^12.0.0 | CLI コマンド定義・引数パース | 学習コストが低く機能十分。Node.js CLI の定番ライブラリ |
| chalk | ^5.0.0 | ターミナルカラー出力 | ESM 対応済み、軽量でシンプルな API |
| cli-table3 | ^0.6.0 | テーブル形式の出力 | Unicode ボーダー対応、列幅自動調整 |
| simple-git | ^3.0.0 | Git 操作の自動化 | Node.js から Git を安全に操作できる高水準 API。シェルインジェクションリスクなし |
| @octokit/rest | ^21.0.0 | GitHub REST API クライアント | GitHub 公式ライブラリ。型定義が充実しており安全に使用できる |
| uuid | ^11.0.0 | UUID v4 生成 | タスク ID の一意性を保証。標準的なライブラリ |

### 開発ツール

| 技術 | バージョン | 用途 | 選定理由 |
|------|-----------|------|----------|
| Vitest | ^2.0.0 | テストフレームワーク | TypeScript との相性が良く高速。既存の devDependencies に含まれている |
| ESLint | ^9.0.0 | 静的解析 | コード品質の担保。既存設定を活用 |
| Prettier | ^3.2.0 | コードフォーマット | チーム内のスタイル統一 |
| Husky + lint-staged | ^9.0.0 / ^15.0.0 | コミット前フック | コミット時に lint・format を自動実行し品質を担保 |

---

## アーキテクチャパターン

### レイヤードアーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│   CLIレイヤー (src/cli/)                              │
│   - ユーザー入力の受付・バリデーション・結果表示        │
│   - Commander.js でコマンドを定義                     │
├─────────────────────────────────────────────────────┤
│   サービスレイヤー (src/services/)                    │
│   - ビジネスロジックの実装                             │
│   - TaskService / GitService / GitHubService / etc.  │
├─────────────────────────────────────────────────────┤
│   データレイヤー (src/services/StorageService.ts)     │
│   - JSON ファイルへの読み書き                         │
│   - バックアップ管理                                  │
└─────────────────────────────────────────────────────┘
```

**依存方向の原則**:

```
CLIレイヤー → サービスレイヤー → データレイヤー  ✅
CLIレイヤー → データレイヤー（直接アクセス）      ❌
サービスレイヤー → CLIレイヤー                   ❌
```

#### CLIレイヤー（`src/cli/`）
- **責務**: ユーザー入力の受付、引数バリデーション、結果のフォーマット・表示、エラー表示
- **許可される操作**: サービスレイヤーの呼び出し
- **禁止される操作**: StorageService・ファイルシステムへの直接アクセス

#### サービスレイヤー（`src/services/`）
- **責務**: ビジネスロジックの実装（タスク CRUD、Git 操作、GitHub 連携）
- **許可される操作**: 他サービスの呼び出し、StorageService の呼び出し
- **禁止される操作**: ターミナル出力、Commander.js の直接依存

#### データレイヤー（`StorageService`）
- **責務**: JSON ファイルへの読み書き、バックアップ作成
- **許可される操作**: Node.js fs モジュールの使用
- **禁止される操作**: ビジネスロジックの実装

### システム全体構成図

```mermaid
graph TB
    User[ユーザー]

    subgraph CLI["CLIレイヤー (src/cli/)"]
        Commander[Commander.js]
        Formatter[Formatter / chalk + cli-table3]
    end

    subgraph Services["サービスレイヤー (src/services/)"]
        TaskService[TaskService]
        GitService[GitService]
        GitHubService[GitHubService]
        ConfigService[ConfigService]
        StorageService[StorageService]
    end

    subgraph External["外部システム"]
        GitRepo[(Gitリポジトリ)]
        GitHub[(GitHub API)]
        TaskJSON[(.task/tasks.json)]
        ConfigJSON[(~/.taskcli/config.json)]
    end

    User --> Commander
    Commander --> TaskService
    Commander --> GitService
    Commander --> GitHubService
    TaskService --> StorageService
    GitService --> GitRepo
    GitHubService --> GitHub
    GitHubService --> ConfigService
    ConfigService --> ConfigJSON
    StorageService --> TaskJSON
    Commander --> Formatter
    Formatter --> User
```

---

## データ永続化戦略

### ストレージ方式

| データ種別 | ストレージ | フォーマット | 場所 |
|-----------|----------|-------------|------|
| タスクデータ | ローカルファイル | JSON | `.task/tasks.json`（プロジェクト内） |
| タスクバックアップ | ローカルファイル | JSON | `.task/tasks.json.bak`（プロジェクト内） |
| 設定データ | ローカルファイル | JSON | `~/.taskcli/config.json`（ユーザーホーム） |

**設計上の考慮事項**:
- タスクデータはプロジェクト内（`.task/`）に保存するため、Git 管理下に置いてチームで共有できる
- `.task/tasks.json` を `.gitignore` に含めるかどうかはユーザーの選択とする
- 将来的に SQLite へ移行できるよう、`StorageService` のインターフェースを固定してデータレイヤーを抽象化する

### バックアップ戦略

- **タイミング**: 書き込み操作（create / update / delete）の直前に自動実行
- **世代管理**: 直前の1世代のみ保持（`.task/tasks.json.bak`）
- **復元方法**: `cp .task/tasks.json.bak .task/tasks.json` で手動復元
- **バックアップスキップ条件**: 前回バックアップと内容が同一の場合はスキップ（不要なI/Oを排除）

---

## パフォーマンス要件

### レスポンスタイム

| 操作 | 目標時間 | 測定環境 |
|------|---------|---------|
| `task add` | 100ms 以内 | CPU Core i5 相当、メモリ 8GB、SSD |
| `task list`（1,000件） | 1秒 以内 | 同上 |
| `task list`（10,000件） | 5秒 以内 | 同上 |
| `task start`（Gitブランチ作成含む） | 500ms 以内 | 同上 |
| `task import --github` | 5秒 以内 | GitHub API レスポンス次第。タイムアウト設定: 10秒 |

### リソース使用量

| リソース | 上限 | 理由 |
|---------|------|------|
| メモリ | 128MB | CLIツールとして常駐しない想定。Node.js 起動オーバーヘッドを含む |
| CPU | 一時的なスパイクのみ | 非同期処理でブロッキングを回避 |
| ディスク | 10MB（データ除く） | npm パッケージのインストールサイズ |

---

## セキュリティアーキテクチャ

### データ保護

- **GitHub Token の保管**: `~/.taskcli/config.json` に保存し、ファイルパーミッションを `600`（所有者のみ読み書き可）に強制設定
- **Token の漏洩防止**: エラーログ・標準出力に Token を一切出力しない。エラー表示時はマスクして表示
- **プロジェクトデータ**: `.task/tasks.json` のパーミッションは制限しない（Git 管理でチーム共有を想定するため）

### 入力検証

- **タイトル**: 空文字 / 200文字超をエラー
- **期限日**: ISO 8601 形式（`YYYY-MM-DD`）かつ有効な日付かを検証
- **タスク ID**: UUID v4 形式のみ受け付ける（外部入力で使用する場合）
- **優先度**: `high` / `medium` / `low` のいずれかのみ受け付ける

### コマンドインジェクション対策

- Git 操作は `simple-git` の API 経由のみ使用し、タスクタイトルをシェルコマンドに直接渡さない
- ブランチ名は英数字・ハイフン・スラッシュのみに正規化してから Git に渡す

---

## スケーラビリティ設計

### データ増加への対応

- **想定データ量**: 10,000件まで正常動作（パフォーマンス要件を維持）
- **10,000件超の対応**: `task archive` で完了タスクをアーカイブし、アクティブタスクを絞り込む
- **将来の SQLite 移行**: `StorageService` を抽象インターフェースとして定義し、将来の実装差し替えを容易にする

```typescript
interface IStorageService {
  load(): TaskStore;
  save(store: TaskStore): void;
  backup(): void;
}
// 現在: JsonStorageService implements IStorageService
// 将来: SqliteStorageService implements IStorageService
```

### 機能拡張性

- **GitHub 以外の Git ホスティング対応**: `GitHubService` を抽象化し、GitLab・Bitbucket 対応を将来実装できる設計とする
- **プラグインシステム**: MVP 範囲外だが、`~/.taskcli/plugins/` ディレクトリを将来のプラグイン配置場所として設計上確保する

---

## テスト戦略

### ユニットテスト（Vitest）

- **フレームワーク**: Vitest
- **対象**: `TaskService`・`StorageService`・`GitService.toBranchName`・入力バリデーション関数
- **カバレッジ目標**: サービスレイヤー 80% 以上
- **方針**: 外部依存（ファイルシステム・Git・GitHub API）はモックを使用

### 統合テスト

- **方法**: 一時ディレクトリを作成し、実際のファイル I/O を使ってテスト
- **対象**: `task add` → `task list` のデータフロー全体、`StorageService` のバックアップ動作

### E2E テスト

- **ツール**: Vitest + 子プロセスで CLI を呼び出す
- **シナリオ**:
  - MVP フロー: `task add` → `task start` → `task done` の一連の動作
  - エラーケース: 存在しない ID へのコマンド実行、バリデーションエラー

---

## 技術的制約

### 環境要件

- **OS**: macOS、Linux、Windows（Git Bash / WSL2）
- **必須ランタイム**: Node.js v18 以上（`node --version` で確認）
- **必須外部依存**: Git（`git --version` で確認）
- **オプション外部依存**: GitHub Personal Access Token（GitHub 連携機能のみ必要）
- **最小メモリ**: 256MB（Node.js 起動分を含む）
- **最小ディスク**: インストール先に 50MB の空き容量

### パフォーマンス制約

- Node.js の起動オーバーヘッド（50〜100ms）があるため、全コマンドで 100ms 以内の目標は起動後の処理時間を指す
- ネットワーク接続を必要とするコマンド（`task import --github`・`task sync`）は 100ms 目標の対象外

### セキュリティ制約

- GitHub Token をコード・ログ・標準出力に含めることを禁止
- ファイルシステムへの書き込みはプロジェクトの `.task/` ディレクトリと `~/.taskcli/` のみに限定

---

## 依存関係管理

| ライブラリ | 用途 | バージョン管理方針 | 理由 |
|-----------|------|-------------------|------|
| commander | CLI フレームワーク | `^12.0.0`（マイナーまで自動） | 安定した API、後方互換性あり |
| chalk | カラー出力 | `^5.0.0`（マイナーまで自動） | ESM 対応済み、v5 以降は安定 |
| cli-table3 | テーブル表示 | `^0.6.0`（マイナーまで自動） | 更新頻度が低く安定 |
| simple-git | Git 操作 | `^3.0.0`（マイナーまで自動） | 安定 API |
| @octokit/rest | GitHub API | `^21.0.0`（マイナーまで自動） | GitHub 公式、定期更新あり |
| uuid | UUID 生成 | `^11.0.0`（マイナーまで自動） | 安定 API |
| typescript | ビルド | `~5.3.0`（パッチのみ） | 破壊的変更リスクを避けるため保守的に管理 |
| vitest | テスト | `^2.0.0`（マイナーまで自動） | 活発に開発中のため最新パッチを受け取る |
