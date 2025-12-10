# AIDA Architecture

## Overview

AIDAは**汎用AIエージェント オーケストレーションフレームワーク**です。

設計の核心は**プラットフォーム非依存性**にあります。Claude Code、Gemini CLI、Codex CLI、あるいは将来登場する新しいエージェントに対応できるよう、抽象化レイヤー（Agent Interface）を設けています。

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Human Operator                                 │
│                      タスク定義・承認・方向性決定                          │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              AINI                                        │
│                         状態可視化・操作UI                                │
│                    （別プロジェクト、連携可能）                            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ Markdown Protocol
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                          AIDA Core                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                       Orchestrator                               │    │
│  │  ┌───────────┐    ┌───────────┐    ┌───────────┐                │    │
│  │  │ Commander │ → │  Workers  │ → │ Aggregator │                │    │
│  │  │ (分割)    │    │  (実行)   │    │  (集約)   │                │    │
│  │  └───────────┘    └───────────┘    └───────────┘                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                 │                                        │
│                                 ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                      Agent Interface                             │    │
│  │                       (抽象化レイヤー)                            │    │
│  │                                                                  │    │
│  │   Execute() | GetStatus() | Stop() | Capabilities()             │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────┐
│                          Adapters                                        │
│                                │                                         │
│    ┌───────────┐    ┌──────────┴───┐    ┌───────────┐    ┌───────────┐  │
│    │  Claude   │    │    Gemini    │    │   Codex   │    │  Future   │  │
│    │   Code    │    │     CLI      │    │    CLI    │    │   Agent   │  │
│    └───────────┘    └──────────────┘    └───────────┘    └───────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Component Details

### 1. Commander

**責務**: タスクの分割と割り当て

```go
type Commander struct {
    taskQueue    chan Task
    workers      map[string]*Worker
    strategy     SplitStrategy
}

// タスクを分析し、サブタスクに分割
func (c *Commander) Split(task Task) ([]Task, error)

// 適切なWorkerにタスクを割り当て
func (c *Commander) Assign(task Task) error
```

**分割戦略**:
- **ByFile**: ファイル単位で分割
- **ByFeature**: 機能単位で分割
- **ByLayer**: レイヤー（frontend/backend/test）で分割

### 2. Worker

**責務**: タスクの実行

```go
type Worker struct {
    ID        string
    Agent     agent.Agent
    Worktree  *git.Worktree
    Status    WorkerStatus
    CurrentTask *Task
}

// タスクを実行
func (w *Worker) Execute(ctx context.Context, task Task) (Result, error)

// 状態を報告
func (w *Worker) Report() WorkerReport
```

**状態遷移**:
```
Idle → Assigned → Running → Completed/Failed → Idle
                    ↓
                  Stopped
```

### 3. Aggregator

**責務**: 結果の集約とマージ

```go
type Aggregator struct {
    results   map[string]Result
    conflicts []Conflict
}

// 結果を集約
func (a *Aggregator) Collect(result Result) error

// 競合を検出
func (a *Aggregator) DetectConflicts() []Conflict

// マージを実行
func (a *Aggregator) Merge() error
```

### 4. Agent Interface

**責務**: エージェント実装の抽象化

```go
// pkg/agent/interface.go
package agent

type Agent interface {
    // タスクを実行し結果を返す
    Execute(ctx context.Context, task Task) (Result, error)

    // 現在の状態を取得
    GetStatus() Status

    // 実行を停止
    Stop() error

    // エージェントの能力を取得
    Capabilities() []Capability
}

type AgentFactory interface {
    // 設定からエージェントを生成
    Create(config Config) (Agent, error)

    // エージェント名を返す
    Name() string
}
```

---

## Data Flow

### Task Lifecycle

```
1. Human/AINI がタスクを定義
       │
       ▼
2. Commander がタスクを受信
       │
       ▼
3. Commander がサブタスクに分割
       │
       ▼
4. 各サブタスクを Worker に割り当て
       │
       ▼
5. Worker が Git worktree を作成
       │
       ▼
6. Worker が Agent.Execute() を呼び出し
       │
       ▼
7. Agent が実際の作業を実行
       │
       ▼
8. 結果を Worker に返却
       │
       ▼
9. Worker が結果を Aggregator に報告
       │
       ▼
10. Aggregator が全結果を集約
       │
       ▼
11. 競合解決・マージ
       │
       ▼
12. 完了を Human/AINI に通知
```

---

## Markdown Protocol

### kanban.md Format

```markdown
# Project: {name}

## ⚙️ Meta
**Agent**: {default-agent}
**Started**: {ISO8601}
**Config**: {config-path}

## 📋 Tasks

### 🚀 In Progress
- [ ] #{id} {description} @{worker-id}
  - agent: {agent-name}
  - worktree: {branch-name}
  - started: {ISO8601}
  - progress: {percentage}%

### 📝 To Do
- [ ] #{id} {description}
  - priority: {1-5}
  - depends_on: #{other-id}

### ✅ Done
- [x] #{id} {description} @{worker-id}
  - completed: {ISO8601}
  - duration: {duration}

## 📊 Workers
| ID | Agent | Status | Current Task | Started |
|----|-------|--------|--------------|---------|
| {id} | {agent} | {status} | #{task-id} | {time} |

## 📝 Logs
### {worker-id}
```
{log-content}
```
```

### Protocol Rules

1. **ID Format**: `#task-XXX` (3桁ゼロパディング)
2. **Worker Format**: `@worker-X`
3. **Timestamps**: ISO8601 形式
4. **Status**: `To Do` → `In Progress` → `Done`

---

## Git Worktree Strategy

```
project/
├── .git/                    # メインリポジトリ
├── src/                     # メインブランチ
└── .aida/
    └── worktrees/
        ├── worker-1/        # feature/task-001
        ├── worker-2/        # feature/task-002
        └── worker-3/        # feature/task-003
```

### Worktree Lifecycle

```bash
# 1. Worker起動時にworktree作成
git worktree add .aida/worktrees/worker-1 -b feature/task-001

# 2. Worker作業完了後
git worktree remove .aida/worktrees/worker-1

# 3. Aggregatorがマージ
git merge feature/task-001
```

---

## Error Handling

### Error Types

```go
type AIDAError struct {
    Code    ErrorCode
    Message string
    Cause   error
    Context map[string]any
}

const (
    ErrAgentNotFound ErrorCode = iota
    ErrWorkerFailed
    ErrTaskTimeout
    ErrConflictDetected
    ErrWorktreeFailure
)
```

### Recovery Strategies

| Error | Strategy |
|-------|----------|
| Agent timeout | リトライ (max 3回) |
| Worker crash | 別Workerに再割り当て |
| Git conflict | 人間に通知、手動解決待ち |
| Agent not found | フォールバックエージェント使用 |

---

## Configuration

### Global Config (~/.aida/config.yaml)

```yaml
default_agent: claude-code

agents:
  claude-code:
    command: claude
    args: ["--print", "--output-format", "json"]
  gemini-cli:
    command: gemini
    args: ["--format", "json"]

workers:
  max_concurrent: 5
  timeout: 30m

git:
  use_worktree: true
  worktree_dir: .aida/worktrees
  auto_branch: true
  branch_prefix: feature/

protocol:
  kanban_file: kanban.md
  archive_file: archive.md
```

### Project Config (.aida/config.yaml)

```yaml
# プロジェクト固有の上書き
default_agent: gemini-cli
workers:
  max_concurrent: 3
```

---

## Extension Points

### Adding a New Agent

1. `adapters/{agent-name}/` ディレクトリ作成
2. `Agent` インターフェースを実装
3. `AgentFactory` を実装
4. `init()` で登録

```go
// adapters/newagent/adapter.go
package newagent

func init() {
    agent.Register(&Factory{})
}

type Factory struct{}

func (f *Factory) Name() string { return "new-agent" }

func (f *Factory) Create(config agent.Config) (agent.Agent, error) {
    return &NewAgent{config: config}, nil
}
```

---

## Security Considerations

1. **コマンドインジェクション防止**: タスク記述をサニタイズ
2. **ファイルアクセス制限**: worktree内のみ操作可
3. **タイムアウト**: 無限ループ防止
4. **ログの秘匿**: 機密情報のマスキング
