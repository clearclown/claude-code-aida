# CLAUDE.md - AIDA Project Instructions

このドキュメントはClaude Codeがこのプロジェクトで作業する際のガイドラインです。

---

## Project Identity

**AIDA** (Agent Integration & Development Architecture)
- **役割**: 汎用AIエージェント オーケストレーションフレームワーク
- **設計思想**: プラットフォーム非依存（Claude Code以外のエージェントにも対応）

### 関連プロジェクト
| Project | 役割 | 依存関係 |
|---------|------|----------|
| **AIDA** (this) | オーケストレーション | 永続（エージェント非依存） |
| **AINI** | 状態可視化UI | 永続（エージェント非依存） |
| **Claude-Code-Profile-Manager** | Claude Code設定管理 | Claude Code依存（廃止可能） |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     AIDA Core (永続層)                          │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ Orchestrator│  │   Worker    │  │ Aggregator  │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         └────────────────┼────────────────┘                     │
│                          ▼                                      │
│                 ┌─────────────────┐                             │
│                 │ Agent Interface │  ← 抽象化レイヤー            │
│                 └────────┬────────┘                             │
└──────────────────────────┼──────────────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────────────┐
│                     Adapters (実装層)                            │
│                           │                                      │
│  ┌─────────────┐  ┌──────┴──────┐  ┌─────────────┐              │
│  │ claude-code │  │ gemini-cli  │  │  codex-cli  │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Component | Technology | 理由 |
|-----------|------------|------|
| Language | Go 1.22+ | プロセス管理、クロスプラットフォーム |
| CLI | cobra + viper | 標準的なCLIフレームワーク |
| TUI | bubbletea + lipgloss | リッチなターミナルUI |
| Config | YAML | 人間が読みやすい |
| Protocol | Markdown | Git管理可能、エージェントが読み書き可能 |

---

## Directory Structure

```
aida/
├── cmd/
│   └── aida/
│       └── main.go           # エントリーポイント
├── internal/
│   ├── orchestrator/         # オーケストレーションロジック
│   │   ├── commander.go      # タスク分割・指示
│   │   ├── worker.go         # Worker管理
│   │   └── aggregator.go     # 結果集約
│   ├── worktree/             # Git worktree統合
│   ├── protocol/             # Markdown Protocol実装
│   ├── tui/                  # Terminal UI
│   └── config/               # 設定管理
├── pkg/
│   └── agent/                # Agent Interface (公開API)
│       ├── interface.go      # インターフェース定義
│       ├── task.go           # タスク定義
│       └── result.go         # 結果定義
├── adapters/
│   ├── claudecode/           # Claude Code adapter
│   ├── geminicli/            # Gemini CLI adapter
│   └── codexcli/             # Codex CLI adapter
├── docs/
│   ├── architecture.md       # アーキテクチャ詳細
│   ├── agent-interface.md    # Agent Interface仕様
│   └── markdown-protocol.md  # Markdown Protocol仕様
├── go.mod
├── go.sum
├── README.md
├── CLAUDE.md                 # このファイル
└── LICENSE
```

---

## Core Interfaces

### Agent Interface

```go
// pkg/agent/interface.go
package agent

type Agent interface {
    Execute(ctx context.Context, task Task) (Result, error)
    GetStatus() Status
    Stop() error
    Capabilities() []Capability
}

type AgentFactory interface {
    Create(config Config) (Agent, error)
    Name() string
}
```

### Task Definition

```go
// pkg/agent/task.go
type Task struct {
    ID          string            `json:"id"`
    Description string            `json:"description"`
    WorkDir     string            `json:"work_dir"`
    Branch      string            `json:"branch,omitempty"`
    Context     map[string]any    `json:"context,omitempty"`
    Priority    int               `json:"priority"`
    DependsOn   []string          `json:"depends_on,omitempty"`
}
```

---

## Markdown Protocol

タスク状態の共有フォーマット（`kanban.md`）:

```markdown
# Project: {project-name}

## ⚙️ Meta
**Agent**: claude-code
**Started**: 2025-12-10T10:00:00Z

## 📋 Tasks

### 🚀 In Progress
- [ ] #task-001 認証機能の実装 @worker-1
  - agent: claude-code
  - worktree: feature/auth

### 📝 To Do
- [ ] #task-002 API設計

### ✅ Done
- [x] #task-000 初期化完了
```

---

## Implementation Priority

### Phase 1: Core Foundation (v0.1)
1. Agent Interface 定義
2. Claude Code adapter 実装
3. 基本的な Worker 管理
4. Markdown Protocol パーサー

### Phase 2: Orchestration (v0.2)
1. Commander（タスク分割）
2. Aggregator（結果集約）
3. Git worktree 統合
4. 並列実行制御

### Phase 3: TUI & Polish (v0.3)
1. bubbletea による TUI
2. リアルタイム状態表示
3. ログビューア
4. エラーハンドリング強化

### Phase 4: Multi-Agent (v0.4)
1. Gemini CLI adapter
2. Codex CLI adapter
3. エージェント間の相互運用
4. 動的エージェント選択

---

## Coding Standards

### Error Handling
```go
// Good: コンテキスト付きエラー
if err != nil {
    return fmt.Errorf("failed to spawn worker %s: %w", workerID, err)
}

// Bad: 素のエラー
if err != nil {
    return err
}
```

### Naming
- **Files**: snake_case (`worker_manager.go`)
- **Packages**: lowercase (`orchestrator`)
- **Exported**: PascalCase (`SpawnWorker`)
- **Unexported**: camelCase (`parseTask`)

### Testing
- Table-driven tests を優先
- モック用にインターフェースを活用
- `testify/assert` 使用可

---

## Key Design Decisions

### 1. エージェント非依存
AIDAのコアは特定のエージェントに依存しない。Claude Codeが廃れても、新しいadapterを追加するだけで対応可能。

### 2. Markdown Protocol
- 人間が直接編集可能
- Git管理可能
- どのエージェントも読み書き可能
- AINIとの連携が容易

### 3. Git Worktree
各Workerが独立したブランチで作業することで、競合を防ぎ並列作業を可能にする。

---

## Notes for Claude

- **段階的実装**: Phase 1から順に実装
- **テスト優先**: 各機能にテストを書く
- **インターフェース重視**: 具体実装より抽象化を優先
- **後方互換性**: Markdown Protocolの変更は慎重に
- **日本語コメント可**: コード内コメントは日本語でも可
