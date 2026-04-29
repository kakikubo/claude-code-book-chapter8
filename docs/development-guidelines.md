# 開発ガイドライン (Development Guidelines)

## コーディング規約

### 命名規則

#### 変数・関数

```typescript
// ✅ 良い例
const taskList = await taskService.listTasks(filter);
const isGitRepo = await gitService.isGitRepo();
function toBranchName(taskId: string, title: string): string { }

// ❌ 悪い例
const data = await svc.list();
function convert(id: string, t: string): string { }
```

**原則**:
- 変数: camelCase、名詞または名詞句
- 関数: camelCase、動詞で始める
- 定数: `UPPER_SNAKE_CASE`
- Boolean: `is`・`has`・`should`・`can` で始める

#### クラス・インターフェース

```typescript
// クラス: PascalCase + 役割サフィックス
class TaskService { }
class GitService { }
class StorageService { }

// インターフェース（原則）: PascalCase、I接頭辞なし
interface Task { }
interface Config { }
interface TaskStore { }

// 型エイリアス: PascalCase
type TaskStatus = 'open' | 'in_progress' | 'completed' | 'archived';
type TaskPriority = 'high' | 'medium' | 'low';
```

**例外: I 接頭辞の使用**

実装クラスと抽象を同一ファイル / 同一名空間内で並存させ、両者を呼び分ける必要がある場合に限り I 接頭辞を使用します。

```typescript
// 抽象（永続化境界）と具象（JSON / SQLite）を区別する場合に限り I 接頭辞を使う
interface IStorageService {
  load(): TaskStore;
  save(store: TaskStore): void;
}
class JsonStorageService implements IStorageService { /* ... */ }
// 将来: class SqliteStorageService implements IStorageService { ... }
```

#### ファイル名

| 種別 | 規則 | 例 |
|------|------|----|
| クラスファイル | PascalCase + サフィックス | `TaskService.ts`、`AddCommand.ts` |
| 型定義ファイル | PascalCase | `Task.ts`、`Config.ts` |
| テストファイル | `[対象].test.ts` | `TaskService.test.ts` |
| 設定ファイル | ツール規約に従う | `tsconfig.json`、`vitest.config.ts` |

---

### コードフォーマット

- **インデント**: 2スペース（Prettier で強制）
- **行の長さ**: 最大80文字（`.prettierrc` の `printWidth` で設定）
- **セミコロン**: あり（`semi: true`）
- **クォート**: シングルクォート（`singleQuote: true`）
- **末尾カンマ**: ES5 互換（`trailingComma: "es5"`）
- **アロー関数の括弧**: 常時付与（`arrowParens: "always"`）

フォーマットは `npm run format`（Prettier）で自動整形されます。設定は `.prettierrc` に集約されています。

---

### 型定義

**`any` の原則禁止（ESLint で警告）**:

`eslint.config.js` で `@typescript-eslint/no-explicit-any: 'warn'` を設定しており、`any` の使用は警告として検出されます。原則として `any` は使用せず、やむを得ず使う場合は理由コメント付きで `eslint-disable-next-line` を明示してください。

```typescript
// ✅ 良い例: 具体的な型
function loadStore(): TaskStore {
  return JSON.parse(content) as TaskStore;
}

// ✅ 例外的に許容: 理由を明示する
// 外部 API の型が動的に変化するため any で受ける
// eslint-disable-next-line @typescript-eslint/no-explicit-any
function handleExternal(payload: any): void {
  // ...
}

// ❌ 悪い例: 理由なく any を使用
function loadStore(): any {
  return JSON.parse(content);
}
```

**インターフェース vs 型エイリアス**:

```typescript
// オブジェクト型はインターフェース
interface Task {
  id: string;
  title: string;
  status: TaskStatus;
}

// ユニオン型や単純なエイリアスは型エイリアス
type TaskStatus = 'open' | 'in_progress' | 'completed' | 'archived';
type TaskId = string;
```

**オプショナルフィールド**:

```typescript
interface Task {
  id: string;           // 必須
  title: string;        // 必須
  description?: string; // オプション（?を使用）
  dueDate?: string;     // オプション
}
```

---

### 関数設計

**単一責務の原則**:

```typescript
// ✅ 良い例: 1関数1責務
function validateTitle(title: string): void {
  if (!title || title.length === 0) {
    throw new ValidationError('タイトルは必須です', 'title', title);
  }
  if (title.length > 200) {
    throw new ValidationError('タイトルは200文字以内です', 'title', title);
  }
}

function generateId(): string {
  return randomUUID();
}
```

**パラメータ数**:

```typescript
// ✅ 良い例: オブジェクトでまとめる（3個以上の場合）
interface CreateTaskInput {
  title: string;
  priority?: TaskPriority;
  dueDate?: string;
  description?: string;
}

function createTask(input: CreateTaskInput): Task { }

// ❌ 悪い例: パラメータが多すぎる
function createTask(
  title: string,
  priority: string,
  dueDate: string,
  description: string
): Task { }
```

**関数の長さ**:
- 目標: 30行以内
- 30〜50行: リファクタリングを検討
- 50行以上: 関数の分割を強く推奨

---

### エラーハンドリング

**カスタムエラークラスの定義**（`src/types/errors.ts`）:

```typescript
export class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: unknown
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

export class NotFoundError extends Error {
  constructor(public resource: string, public id: string) {
    super(`${resource}が見つかりません (ID: ${id})`);
    this.name = 'NotFoundError';
  }
}

export class GitError extends Error {
  constructor(message: string, public cause?: Error) {
    super(message);
    this.name = 'GitError';
  }
}
```

**エラーハンドリングのパターン**:

```typescript
// ✅ 良い例: 予期されるエラーを適切に処理
async function startTask(id: string): Promise<Task> {
  const task = taskService.getTask(id);  // NotFoundError をスロー可能

  try {
    const branchName = gitService.toBranchName(id, task.title, 'feature');
    await gitService.createAndCheckoutBranch(branchName);
  } catch (error) {
    if (error instanceof GitError) {
      // Gitエラー: タスクのステータス変更をロールバックしてユーザーに通知
      throw new GitError(`ブランチ作成に失敗しました: ${error.message}`, error);
    }
    throw error; // 予期しないエラーは上位に伝播
  }

  return taskService.updateTask(id, { status: 'in_progress', branch: branchName });
}
```

**エラーメッセージの原則**:
- 何が起きたか + 推奨される対処法を含める
- 技術的な詳細はログに記録し、ユーザー向けメッセージはシンプルに

```typescript
// ✅ 具体的で解決策を示す
throw new ValidationError(
  'タイトルは1〜200文字で入力してください（現在: 250文字）',
  'title',
  title
);

// ❌ 曖昧
throw new Error('Invalid input');
```

---

### コメント規約

**「何をしているか」ではなく「なぜそうするか」を書く**:

```typescript
// ✅ 良い例: 理由を説明
// Gitリポジトリが存在しない環境でも基本操作を継続するためスキップ
if (!isGitRepo) {
  return taskService.startTask(id);
}

// ❌ 悪い例: コードを繰り返すだけ
// isGitRepoがfalseかどうかチェック
if (!isGitRepo) { ... }
```

**TSDoc コメント（公開 API に記述）**:

```typescript
/**
 * タスクタイトルからGitブランチ名を生成する
 *
 * @param taskId - タスクのID（ブランチ名に含まれる）
 * @param title - タスクのタイトル（英数字・ハイフンに正規化）
 * @param prefix - ブランチプレフィックス（デフォルト: "feature"）
 * @returns "feature/task-1-title-slug" 形式のブランチ名
 */
toBranchName(taskId: string, title: string, prefix: string): string
```

---

### セキュリティ

**GitHub Token の扱い**:

```typescript
// ✅ 良い例: トークンをログに出力しない
catch (error) {
  console.error('[ERROR] GitHub認証に失敗しました。トークンを確認してください');
  // error.message にトークンが含まれる可能性があるため出力しない
}

// ❌ 悪い例: トークンがログに漏洩
console.error('GitHub API error:', error.message);
```

**入力バリデーション**:

```typescript
// ✅ 良い例: 境界値で必ずバリデーション
function validateDueDate(dueDate: string): void {
  const dateRegex = /^\d{4}-\d{2}-\d{2}$/;
  if (!dateRegex.test(dueDate)) {
    throw new ValidationError(
      '期限日は YYYY-MM-DD 形式で入力してください',
      'dueDate',
      dueDate
    );
  }
  if (isNaN(Date.parse(dueDate))) {
    throw new ValidationError('無効な日付です', 'dueDate', dueDate);
  }
}
```

---

## Git 運用ルール

### ブランチ戦略

TaskCLI は個人開発を主な対象とするため、シンプルなトランクベース開発を採用します。

```
main (安定版・常にデプロイ可能な状態)
├── feature/[機能名]   # 新機能開発
├── fix/[修正内容]     # バグ修正
└── docs/[対象]        # ドキュメント更新
```

**運用ルール**:
- `main` への直接コミットは禁止。必ず PR を通じてマージ
- `feature/*`・`fix/*` は `main` から分岐し、作業完了後 PR で `main` にマージ
- ブランチ名は kebab-case（例: `feature/task-priority`、`fix/branch-name-encoding`）
- タスクと連携する作業ブランチは `task start` コマンドで自動生成（例: `feature/task-1-user-auth`）

---

### コミットメッセージ規約（Conventional Commits）

**フォーマット**:

```
<type>(<scope>): <subject>

<body>（オプション）

<footer>（オプション）
```

**Type 一覧**:

| Type | 説明 |
|------|------|
| `feat` | 新機能 |
| `fix` | バグ修正 |
| `docs` | ドキュメントのみの変更 |
| `style` | コードの動作に影響しない変更（フォーマット等） |
| `refactor` | リファクタリング |
| `test` | テスト追加・修正 |
| `chore` | ビルド設定・依存関係更新 |

**良い例**:

```
feat(task): Gitブランチ自動作成機能を追加

task start コマンド実行時に feature/task-{id}-{slug} 形式の
Gitブランチを自動作成・チェックアウトする機能を実装した。

実装内容:
- GitService.createAndCheckoutBranch() を追加
- GitService.toBranchName() でブランチ名を正規化
- Gitリポジトリが存在しない場合はスキップ

Closes #12
```

---

### プルリクエストのプロセス

**PR テンプレート**:

```markdown
## 変更の種類
- [ ] 新機能 (feat)
- [ ] バグ修正 (fix)
- [ ] リファクタリング (refactor)
- [ ] ドキュメント (docs)
- [ ] その他 (chore)

## 変更内容
### 何を変更したか
[簡潔な説明]

### なぜ変更したか
[背景・理由]

## テスト
- [ ] ユニットテスト追加・更新
- [ ] 手動テスト実施

## 関連
Closes #[番号]
```

**PR 作成前のチェック**:
- [ ] `npm run typecheck` がパスする
- [ ] `npm run lint` がパスする
- [ ] `npm test` が全てパスする
- [ ] PR のタイトルが Conventional Commits 形式になっている

**マージ方針**: Squash merge を推奨（コミット履歴をクリーンに保つ）

---

## テスト戦略

### 設定ファイル

テスト設定は `vitest.config.ts` に集約します。`tsconfig.json` の `include` は `src/**` のみのため、`tests/**` を Vitest 側で明示的に対象に含めています。

### テストの種類と比率（テストピラミッド）

```
       /\
      /E2E\       10%（主要フローのみ）
     /------\
    / 統合   \    20%
   /----------\
  / ユニット   \  70%
 /--------------\
```

**カバレッジ目標**:

| 対象 | 目標 |
|------|------|
| `src/services/`（サービスレイヤー全体） | 80% 以上 |
| `src/cli/`（CLIレイヤー） | 60% 以上 |
| 主要なユーザーフロー（E2E） | 100% |

### テストの書き方（Given-When-Then パターン）

```typescript
describe('TaskService', () => {
  describe('createTask', () => {
    it('正常なデータでタスクを作成できる', () => {
      // Given: 準備
      const storage = createMockStorage([]);
      const service = new TaskService(storage);

      // When: 実行
      const task = service.createTask({ title: 'テストタスク' });

      // Then: 検証
      expect(task.id).toBeDefined();
      expect(task.title).toBe('テストタスク');
      expect(task.status).toBe('open');
    });

    it('タイトルが空の場合は ValidationError をスローする', () => {
      // Given
      const service = new TaskService(createMockStorage([]));

      // When / Then
      expect(() => service.createTask({ title: '' }))
        .toThrow(ValidationError);
    });
  });
});
```

**テスト命名規則**:
- `[対象メソッド]_[条件]_[期待する結果]` または日本語の自然な文
- 例: `createTask_emptyTitle_throwsValidationError`
- 例: `'タイトルが空の場合は ValidationError をスローする'`

### モックの使用方針

```typescript
// ✅ 良い例: インターフェースベースのモック
const mockStorage: IStorageService = {
  load: vi.fn(() => ({ version: '1.0', tasks: [] })),
  save: vi.fn(),
  backup: vi.fn(),
  exists: vi.fn(() => true),
  initialize: vi.fn(),
};

// ユニットテストでは外部依存（ファイルシステム・Git・GitHub API）をモック化
// 統合テストでは実際のファイルI/Oを使用（一時ディレクトリで実施）
```

---

## コードレビュー基準

### レビューポイント

**機能性**:
- [ ] PRDの受け入れ条件を満たしているか
- [ ] エッジケースが考慮されているか（境界値・null・空配列など）
- [ ] エラーハンドリングが適切か

**パフォーマンス**:
- [ ] [architecture.md のパフォーマンス要件](./architecture.md) を満たしているか（`task add` 100ms、`task list` 1,000件で 1 秒、`task sync` 5 秒以内など）
- [ ] N+1 や不要なファイル I/O が発生していないか（同一コマンド実行中の `tasks.json` 多重読み込みなど）

**セキュリティ**:
- [ ] GitHub Token がログ・標準出力に露出していないか
- [ ] 入力バリデーションが実装されているか
- [ ] シェルインジェクションのリスクがないか

**可読性・保守性**:
- [ ] 命名が明確か（略語・単一文字変数を使っていないか）
- [ ] レイヤー間の依存方向が正しいか（CLI → Service → データアクセス境界）
- [ ] 1ファイル300行以内か

### レビューコメントの書き方

```markdown
## ✅ 良い例
[必須] セキュリティ: GitHub Tokenが error.message に含まれ、ログに出力される可能性があります。
トークンを含む可能性があるエラーメッセージは直接出力しないようにしてください。

## ❌ 悪い例
このコードは良くないです。
```

**優先度の明示**:
- `[必須]`: マージ前に修正が必要
- `[推奨]`: 修正推奨（マージ自体は可）
- `[提案]`: 将来的に検討してほしい
- `[質問]`: 意図を確認したい

---

## 開発環境セットアップ

### 必要なツール

| ツール | 最低バージョン | 開発環境バージョン | 確認方法 |
|--------|-------------|------------------|---------|
| Node.js | v18 | v24.11.0 | `node --version` |
| npm | v9 | 11.x | `npm --version` |
| Git | v2.30 | v2.40 以降推奨 | `git --version` |

**プロジェクト構成**:

- TypeScript: `~5.3.0`（`package.json` の `devDependencies` に記載、パッチのみ自動更新）
- モジュール形式: ESM（`package.json` の `"type": "module"`）
- 実行ランタイム: Node.js v18 以上で動作保証。詳細は [architecture.md](./architecture.md#テクノロジースタック) 参照

### セットアップ手順

```bash
# 1. リポジトリのクローン
git clone <repository-url>
cd claude-code-book-chapter8

# 2. 依存関係のインストール
npm install

# 3. 型チェック・ lint・テストの確認
npm run typecheck
npm run lint
npm test
```

### 主要コマンド

```bash
npm run build        # TypeScript をコンパイル
npm run dev          # ウォッチモードでコンパイル
npm run typecheck    # 型チェックのみ（ビルドなし）
npm run lint         # ESLint の実行
npm run format       # Prettier でフォーマット
npm test             # テストの実行（1回）
npm run test:watch   # ウォッチモードでテスト実行
npm run test:coverage# カバレッジ計測
npm run test:ui      # Vitest UI を起動（ブラウザでテスト結果を確認）
```

### 自動化（Husky + lint-staged）

コミット前に以下が自動実行されます（`.husky/pre-commit`）:

1. `npx lint-staged` — 変更ファイル単位で以下を実行
    - `eslint --fix`（`*.{ts,tsx}` のみ）
    - `prettier --write`（`*.{ts,tsx}` のみ）
2. `npm run typecheck` — リポジトリ全体の `tsc --noEmit` による型チェック

**注意**: `npm test` はコミット時には実行されません。PR 作成前に手動で確認してください。

---

## 品質自動化（CI/CD）

**現状**: CI は未整備（`.github/workflows/` 配下にワークフロー未定義）。コミット時の品質担保は Husky + lint-staged + `npm run typecheck` に依存しています（前述の「自動化」セクション参照）。

**CI 整備時のテンプレート**: 以下を `.github/workflows/ci.yml` として配置することで、push / PR をトリガーに型チェック・lint・テスト・ビルドを実行できます。

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '24'
          cache: 'npm'
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npm test
      - run: npm run build
```
