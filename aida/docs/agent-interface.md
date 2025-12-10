# Agent Interface Specification

## Overview

Agent Interfaceは、AIDAが様々なAIコーディングエージェントと通信するための抽象化レイヤーです。

この設計により、Claude Code、Gemini CLI、Codex CLIなど異なるエージェントを統一的に扱えます。

---

## Interface Definition

### Core Interface

```go
// pkg/agent/interface.go
package agent

import (
    "context"
)

// Agent はAIコーディングエージェントの抽象インターフェース
type Agent interface {
    // Execute はタスクを実行し結果を返す
    // ctx でキャンセル・タイムアウトを制御
    Execute(ctx context.Context, task Task) (Result, error)

    // GetStatus は現在の実行状態を返す
    GetStatus() Status

    // Stop は実行中のタスクを停止する
    Stop() error

    // Capabilities はエージェントがサポートする機能を返す
    Capabilities() []Capability
}

// AgentFactory はエージェントインスタンスを生成するファクトリ
type AgentFactory interface {
    // Create は設定からエージェントを生成する
    Create(config Config) (Agent, error)

    // Name はエージェントの識別名を返す
    Name() string

    // Version はエージェントのバージョンを返す
    Version() string
}
```

### Data Types

```go
// pkg/agent/task.go
package agent

// Task はエージェントに渡すタスクの定義
type Task struct {
    // ID はタスクの一意識別子
    ID string `json:"id"`

    // Description はタスクの説明（自然言語）
    Description string `json:"description"`

    // WorkDir は作業ディレクトリの絶対パス
    WorkDir string `json:"work_dir"`

    // Branch は作業ブランチ名（Git worktree使用時）
    Branch string `json:"branch,omitempty"`

    // Files は対象ファイルのリスト（ヒント用）
    Files []string `json:"files,omitempty"`

    // Context は追加のコンテキスト情報
    Context map[string]any `json:"context,omitempty"`

    // Priority はタスクの優先度 (1=最高, 5=最低)
    Priority int `json:"priority"`

    // Timeout はタスクのタイムアウト時間
    Timeout time.Duration `json:"timeout,omitempty"`

    // DependsOn は依存するタスクIDのリスト
    DependsOn []string `json:"depends_on,omitempty"`
}
```

```go
// pkg/agent/result.go
package agent

// Result はタスク実行の結果
type Result struct {
    // TaskID は対応するタスクのID
    TaskID string `json:"task_id"`

    // Status は実行結果のステータス
    Status ResultStatus `json:"status"`

    // Output はエージェントの出力テキスト
    Output string `json:"output"`

    // Files は変更されたファイルのリスト
    Files []FileChange `json:"files,omitempty"`

    // Duration は実行にかかった時間
    Duration time.Duration `json:"duration"`

    // Error はエラーがあった場合の詳細
    Error error `json:"error,omitempty"`

    // Metadata はエージェント固有のメタデータ
    Metadata map[string]any `json:"metadata,omitempty"`
}

// ResultStatus は実行結果のステータス
type ResultStatus string

const (
    StatusSuccess ResultStatus = "success"
    StatusFailed  ResultStatus = "failed"
    StatusPartial ResultStatus = "partial"  // 部分的に成功
    StatusSkipped ResultStatus = "skipped"  // スキップされた
)

// FileChange はファイルの変更内容
type FileChange struct {
    Path      string         `json:"path"`
    Operation FileOperation  `json:"operation"`
    Diff      string         `json:"diff,omitempty"`
}

type FileOperation string

const (
    FileCreated  FileOperation = "created"
    FileModified FileOperation = "modified"
    FileDeleted  FileOperation = "deleted"
)
```

```go
// pkg/agent/status.go
package agent

// Status はエージェントの現在状態
type Status struct {
    State      AgentState `json:"state"`
    TaskID     string     `json:"task_id,omitempty"`
    Progress   int        `json:"progress"`  // 0-100
    Message    string     `json:"message,omitempty"`
    StartedAt  time.Time  `json:"started_at,omitempty"`
}

type AgentState string

const (
    StateIdle     AgentState = "idle"
    StateRunning  AgentState = "running"
    StateStopping AgentState = "stopping"
    StateError    AgentState = "error"
)
```

```go
// pkg/agent/capability.go
package agent

// Capability はエージェントがサポートする機能
type Capability string

const (
    CapabilityCodeEdit    Capability = "code_edit"      // コード編集
    CapabilityCodeReview  Capability = "code_review"    // コードレビュー
    CapabilityTestGen     Capability = "test_gen"       // テスト生成
    CapabilityRefactor    Capability = "refactor"       // リファクタリング
    CapabilityGitOps      Capability = "git_ops"        // Git操作
    CapabilityMultiFile   Capability = "multi_file"     // 複数ファイル対応
    CapabilityInteractive Capability = "interactive"    // 対話モード
)
```

---

## Configuration

```go
// pkg/agent/config.go
package agent

// Config はエージェントの設定
type Config struct {
    // Name はエージェント識別名
    Name string `yaml:"name"`

    // Command は実行コマンド
    Command string `yaml:"command"`

    // Args はコマンド引数
    Args []string `yaml:"args,omitempty"`

    // Env は環境変数
    Env map[string]string `yaml:"env,omitempty"`

    // WorkDir はデフォルトの作業ディレクトリ
    WorkDir string `yaml:"work_dir,omitempty"`

    // Timeout はデフォルトのタイムアウト
    Timeout time.Duration `yaml:"timeout,omitempty"`

    // Extra はエージェント固有の設定
    Extra map[string]any `yaml:"extra,omitempty"`
}
```

YAML例:
```yaml
agents:
  claude-code:
    name: claude-code
    command: claude
    args:
      - "--print"
      - "--output-format"
      - "json"
    timeout: 30m
    extra:
      model: claude-sonnet-4-20250514

  gemini-cli:
    name: gemini-cli
    command: gemini
    args:
      - "--format"
      - "json"
    timeout: 30m
```

---

## Registry

```go
// pkg/agent/registry.go
package agent

var registry = make(map[string]AgentFactory)

// Register はファクトリを登録する
func Register(factory AgentFactory) {
    registry[factory.Name()] = factory
}

// Get は名前からファクトリを取得する
func Get(name string) (AgentFactory, bool) {
    f, ok := registry[name]
    return f, ok
}

// List は登録済みエージェント名のリストを返す
func List() []string {
    names := make([]string, 0, len(registry))
    for name := range registry {
        names = append(names, name)
    }
    return names
}

// Create は設定からエージェントを生成する
func Create(config Config) (Agent, error) {
    factory, ok := Get(config.Name)
    if !ok {
        return nil, fmt.Errorf("unknown agent: %s", config.Name)
    }
    return factory.Create(config)
}
```

---

## Adapter Implementation Example

### Claude Code Adapter

```go
// adapters/claudecode/adapter.go
package claudecode

import (
    "context"
    "encoding/json"
    "os/exec"

    "github.com/ablaze/aida/pkg/agent"
)

func init() {
    agent.Register(&Factory{})
}

type Factory struct{}

func (f *Factory) Name() string    { return "claude-code" }
func (f *Factory) Version() string { return "1.0.0" }

func (f *Factory) Create(config agent.Config) (agent.Agent, error) {
    return &ClaudeCodeAgent{
        config: config,
        status: agent.Status{State: agent.StateIdle},
    }, nil
}

type ClaudeCodeAgent struct {
    config  agent.Config
    status  agent.Status
    cmd     *exec.Cmd
    cancel  context.CancelFunc
}

func (a *ClaudeCodeAgent) Execute(ctx context.Context, task agent.Task) (agent.Result, error) {
    ctx, a.cancel = context.WithCancel(ctx)

    a.status = agent.Status{
        State:     agent.StateRunning,
        TaskID:    task.ID,
        StartedAt: time.Now(),
    }

    // Build prompt from task
    prompt := buildPrompt(task)

    // Execute claude command
    args := append(a.config.Args, "-p", prompt)
    a.cmd = exec.CommandContext(ctx, a.config.Command, args...)
    a.cmd.Dir = task.WorkDir

    output, err := a.cmd.Output()

    result := agent.Result{
        TaskID:   task.ID,
        Duration: time.Since(a.status.StartedAt),
    }

    if err != nil {
        result.Status = agent.StatusFailed
        result.Error = err
    } else {
        result.Status = agent.StatusSuccess
        result.Output = string(output)
        result.Files = parseFileChanges(output)
    }

    a.status.State = agent.StateIdle
    return result, nil
}

func (a *ClaudeCodeAgent) GetStatus() agent.Status {
    return a.status
}

func (a *ClaudeCodeAgent) Stop() error {
    if a.cancel != nil {
        a.cancel()
    }
    if a.cmd != nil && a.cmd.Process != nil {
        return a.cmd.Process.Kill()
    }
    return nil
}

func (a *ClaudeCodeAgent) Capabilities() []agent.Capability {
    return []agent.Capability{
        agent.CapabilityCodeEdit,
        agent.CapabilityCodeReview,
        agent.CapabilityTestGen,
        agent.CapabilityRefactor,
        agent.CapabilityGitOps,
        agent.CapabilityMultiFile,
    }
}

func buildPrompt(task agent.Task) string {
    // タスクからプロンプトを構築
    return fmt.Sprintf("Task: %s\n\nDescription: %s", task.ID, task.Description)
}

func parseFileChanges(output []byte) []agent.FileChange {
    // 出力からファイル変更を解析
    // 実装は出力フォーマットに依存
    return nil
}
```

---

## Usage Example

```go
package main

import (
    "context"
    "log"
    "time"

    "github.com/ablaze/aida/pkg/agent"
    _ "github.com/ablaze/aida/adapters/claudecode"  // 自動登録
)

func main() {
    // 設定からエージェントを生成
    config := agent.Config{
        Name:    "claude-code",
        Command: "claude",
        Args:    []string{"--print", "--output-format", "json"},
        Timeout: 30 * time.Minute,
    }

    ag, err := agent.Create(config)
    if err != nil {
        log.Fatal(err)
    }

    // タスクを定義
    task := agent.Task{
        ID:          "task-001",
        Description: "Add unit tests for the User model",
        WorkDir:     "/path/to/project",
        Files:       []string{"models/user.go"},
        Priority:    1,
    }

    // 実行
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
    defer cancel()

    result, err := ag.Execute(ctx, task)
    if err != nil {
        log.Fatal(err)
    }

    log.Printf("Task %s completed: %s", result.TaskID, result.Status)
    log.Printf("Changed files: %d", len(result.Files))
}
```

---

## Error Handling

```go
// pkg/agent/errors.go
package agent

import "errors"

var (
    ErrAgentNotFound    = errors.New("agent not found")
    ErrTaskTimeout      = errors.New("task timeout")
    ErrAgentBusy        = errors.New("agent is busy")
    ErrInvalidTask      = errors.New("invalid task")
    ErrExecutionFailed  = errors.New("execution failed")
)

// AgentError は詳細なエラー情報を持つ
type AgentError struct {
    Code    string
    Message string
    Cause   error
    TaskID  string
}

func (e *AgentError) Error() string {
    if e.Cause != nil {
        return fmt.Sprintf("%s: %s (task: %s, cause: %v)",
            e.Code, e.Message, e.TaskID, e.Cause)
    }
    return fmt.Sprintf("%s: %s (task: %s)", e.Code, e.Message, e.TaskID)
}

func (e *AgentError) Unwrap() error {
    return e.Cause
}
```

---

## Testing

```go
// pkg/agent/mock.go
package agent

// MockAgent はテスト用のモックエージェント
type MockAgent struct {
    ExecuteFunc      func(ctx context.Context, task Task) (Result, error)
    GetStatusFunc    func() Status
    StopFunc         func() error
    CapabilitiesFunc func() []Capability
}

func (m *MockAgent) Execute(ctx context.Context, task Task) (Result, error) {
    if m.ExecuteFunc != nil {
        return m.ExecuteFunc(ctx, task)
    }
    return Result{TaskID: task.ID, Status: StatusSuccess}, nil
}

func (m *MockAgent) GetStatus() Status {
    if m.GetStatusFunc != nil {
        return m.GetStatusFunc()
    }
    return Status{State: StateIdle}
}

func (m *MockAgent) Stop() error {
    if m.StopFunc != nil {
        return m.StopFunc()
    }
    return nil
}

func (m *MockAgent) Capabilities() []Capability {
    if m.CapabilitiesFunc != nil {
        return m.CapabilitiesFunc()
    }
    return []Capability{CapabilityCodeEdit}
}
```

テスト例:
```go
func TestWorkerExecute(t *testing.T) {
    mock := &agent.MockAgent{
        ExecuteFunc: func(ctx context.Context, task agent.Task) (agent.Result, error) {
            return agent.Result{
                TaskID: task.ID,
                Status: agent.StatusSuccess,
                Output: "Done",
            }, nil
        },
    }

    worker := NewWorker("worker-1", mock)
    result, err := worker.Execute(context.Background(), task)

    assert.NoError(t, err)
    assert.Equal(t, agent.StatusSuccess, result.Status)
}
```
