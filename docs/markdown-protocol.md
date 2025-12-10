# Markdown Protocol Specification

## Overview

Markdown Protocolは、AIDA/AINIとAIエージェント間で状態を共有するためのフォーマット仕様です。

**設計原則**:
- 人間が直接読み書き可能
- Git管理可能（差分が見やすい）
- 任意のAIエージェントが解析・更新可能
- シンプルで拡張しやすい

---

## File Structure

```
project/
├── kanban.md       # アクティブなタスク
├── archive.md      # 完了したタスクのアーカイブ
└── .aida/
    └── logs/       # Workerログ
        ├── worker-1.md
        └── worker-2.md
```

---

## kanban.md Format

### Full Example

```markdown
# Project: my-awesome-app

## ⚙️ Meta

**Agent**: claude-code
**Started**: 2025-12-10T10:00:00Z
**Updated**: 2025-12-10T12:30:00Z
**Config**: .aida/config.yaml

## 📋 Tasks

### 🚀 In Progress

- [ ] #task-001 認証機能の実装 @worker-1
  - agent: claude-code
  - worktree: feature/task-001
  - started: 2025-12-10T10:05:00Z
  - progress: 60%
  - files: src/auth/login.go, src/auth/session.go

- [ ] #task-002 API エンドポイント設計 @worker-2
  - agent: gemini-cli
  - worktree: feature/task-002
  - started: 2025-12-10T10:10:00Z
  - progress: 30%

### 📝 To Do

- [ ] #task-003 ユニットテスト追加
  - priority: 1
  - estimate: 2h
  - depends_on: #task-001

- [ ] #task-004 ドキュメント更新
  - priority: 3
  - tags: docs, low-priority

- [ ] #task-005 パフォーマンス最適化
  - priority: 2
  - files: src/database/query.go

### ✅ Done

- [x] #task-000 プロジェクト初期化 @worker-1
  - completed: 2025-12-10T09:50:00Z
  - duration: 5m
  - agent: claude-code

## 📊 Workers

| ID | Agent | Status | Current Task | Started | Progress |
|----|-------|--------|--------------|---------|----------|
| worker-1 | claude-code | running | #task-001 | 10:05 | 60% |
| worker-2 | gemini-cli | running | #task-002 | 10:10 | 30% |
| worker-3 | codex-cli | idle | - | - | - |

## 📈 Statistics

- **Total Tasks**: 6
- **Completed**: 1 (16.7%)
- **In Progress**: 2 (33.3%)
- **To Do**: 3 (50.0%)
- **Estimated Remaining**: 4h

## 📝 Notes

### 2025-12-10
- 10:00 オーケストレーション開始
- 10:05 認証機能の実装を開始
- 12:00 #task-001 の進捗が60%に到達
```

---

## Schema Definition

### Task Item

```
- [ ] #{id} {description} [@{worker-id}]
  - agent: {agent-name}
  - worktree: {branch-name}
  - started: {ISO8601}
  - completed: {ISO8601}
  - duration: {duration}
  - progress: {0-100}%
  - priority: {1-5}
  - estimate: {duration}
  - depends_on: #{task-id}[, #{task-id}...]
  - tags: {tag}[, {tag}...]
  - files: {path}[, {path}...]
```

### Field Definitions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | Yes | タスクID（`#task-XXX`形式） |
| `description` | string | Yes | タスクの説明 |
| `worker-id` | string | No | 割り当てられたWorker |
| `agent` | string | No | 使用するエージェント |
| `worktree` | string | No | Git worktreeのブランチ名 |
| `started` | ISO8601 | No | 開始時刻 |
| `completed` | ISO8601 | No | 完了時刻 |
| `duration` | duration | No | 所要時間 |
| `progress` | int | No | 進捗率（0-100） |
| `priority` | int | No | 優先度（1=最高、5=最低） |
| `estimate` | duration | No | 見積もり時間 |
| `depends_on` | string[] | No | 依存タスクID |
| `tags` | string[] | No | タグ |
| `files` | string[] | No | 関連ファイル |

### Duration Format

- `5m` - 5分
- `2h` - 2時間
- `1d` - 1日
- `1h30m` - 1時間30分

---

## Worker Table

```markdown
## 📊 Workers

| ID | Agent | Status | Current Task | Started | Progress |
|----|-------|--------|--------------|---------|----------|
| {worker-id} | {agent-name} | {status} | #{task-id} | {time} | {progress}% |
```

### Status Values

| Status | Description |
|--------|-------------|
| `idle` | 待機中 |
| `running` | 実行中 |
| `stopping` | 停止中 |
| `error` | エラー発生 |
| `offline` | オフライン |

---

## archive.md Format

```markdown
# Archive: my-awesome-app

> Completed tasks archive

## 📅 2025-12-10

### ✅ Completed

- [x] #task-000 プロジェクト初期化 @worker-1
  - completed: 2025-12-10T09:50:00Z
  - duration: 5m
  - agent: claude-code
  - summary: 基本構造の作成、依存関係の設定

## 📅 2025-12-09

### ✅ Completed

- [x] #task-prev-001 要件定義
  - completed: 2025-12-09T18:00:00Z
  - duration: 3h

## 📊 Monthly Summary

| Month | Completed | Total Time |
|-------|-----------|------------|
| 2025-12 | 2 | 3h 5m |
```

---

## Log Format

### Worker Log (.aida/logs/worker-X.md)

```markdown
# Worker Log: worker-1

## Session: 2025-12-10T10:05:00Z

### Task: #task-001 認証機能の実装

**Agent**: claude-code
**Branch**: feature/task-001

#### 10:05:00 - Started
```
Analyzing task requirements...
```

#### 10:10:00 - Progress Update (20%)
```
Created src/auth/login.go
Implementing login handler...
```

#### 10:30:00 - Progress Update (40%)
```
Created src/auth/session.go
Adding session management...
```

#### 10:50:00 - Progress Update (60%)
```
Implemented JWT token generation
Working on validation...
```

---

### Errors

#### 10:45:00 - Warning
```
Dependency conflict detected: jwt-go version mismatch
Resolved by updating go.mod
```
```

---

## Parsing Rules

### ID Extraction
```regex
#(task-\d{3,})
```

### Worker Extraction
```regex
@(worker-\d+)
```

### Checkbox State
```regex
- \[([ x])\] #
```
- `[ ]` = 未完了
- `[x]` = 完了

### Metadata Extraction
```regex
^\s+-\s+(\w+):\s+(.+)$
```

---

## State Transitions

```
To Do → In Progress → Done → Archive
          ↓
        Failed (stays in In Progress with error tag)
```

### Moving Tasks

**To Do → In Progress**:
1. Workerが割り当てられる（`@worker-X`追加）
2. `agent`, `worktree`, `started` が追加される
3. セクションを移動

**In Progress → Done**:
1. チェックボックスを `[x]` に変更
2. `completed`, `duration` を追加
3. セクションを移動

**Done → Archive**:
1. archive.md に移動
2. 日付セクションに追加

---

## AINI Integration

AINIは以下の方法でkanban.mdを監視・更新します:

### Read
```javascript
// ファイル監視
fs.watch('kanban.md', (eventType) => {
  if (eventType === 'change') {
    const content = fs.readFileSync('kanban.md', 'utf-8');
    const state = parseKanban(content);
    updateUI(state);
  }
});
```

### Write
```javascript
// タスク追加
function addTask(description, options) {
  const id = generateTaskId();
  const taskLine = `- [ ] #${id} ${description}`;
  const metadata = formatMetadata(options);

  insertIntoSection('📝 To Do', taskLine + '\n' + metadata);
}
```

### Sync
- **Conflict Resolution**: 最終更新時刻が新しい方を優先
- **Merge Strategy**: セクション単位でマージ
- **Lock**: 編集中は `.kanban.lock` を作成

---

## Best Practices

1. **ID は連番**: `#task-001`, `#task-002`... で管理しやすく
2. **説明は簡潔に**: 1行で内容がわかるように
3. **メタデータは必要最小限**: 全フィールドを埋める必要はない
4. **定期的にアーカイブ**: Done が溜まったら archive.md に移動
5. **コメントで補足**: `## 📝 Notes` セクションを活用
