# OSS Adoption Plan - パクれる要素まとめ

関連OSSプロジェクトを調査し、AIDAに取り入れるべき設計・実装パターンをまとめる。

---

## 調査対象プロジェクト

| Project | 言語 | Stars | 主な特徴 |
|---------|------|-------|----------|
| [Vibe Kanban](https://github.com/BloopAI/vibe-kanban) | Rust + TS | 3k+ | マルチエージェントオーケストレーション |
| [MarkdownTaskManager](https://github.com/ioniks/MarkdownTaskManager) | HTML/JS | - | 単一HTMLファイル、AI統合ワークフロー |
| [Aider](https://github.com/paul-gauthier/aider) | Python | 30k+ | Repo Map、Git自動コミット |
| [CrewAI](https://github.com/crewAIInc/crewAI) | Python | 25k+ | マルチエージェント協調 |
| [go-git](https://github.com/go-git/go-git) | Go | 6k+ | Pure Go Git実装 |
| [Bubbletea](https://github.com/charmbracelet/bubbletea) | Go | 28k+ | Elm Architecture TUI |

---

## 1. Vibe Kanban からパクれる要素

### 1.1 マルチエージェント並列実行

```
特徴:
- 複数のコーディングエージェント（Claude Code, Gemini CLI, Codex等）を切り替え可能
- 並列または逐次実行のオーケストレーション
- Git worktreeを使った独立したブランチ作業

パクるべき設計:
- エージェント切り替えのUI/UX
- worktree自動クリーンアップ（DISABLE_WORKTREE_ORPHAN_CLEANUP環境変数）
- リモートSSH経由でのプロジェクトオープン
```

### 1.2 MCP設定の中央管理

```
特徴:
- コーディングエージェントのMCP設定を一元管理
- プロジェクト単位での設定切り替え

パクるべき設計:
- MCP config統合（AIDAの設定層として活用）
```

---

## 2. MarkdownTaskManager からパクれる要素

### 2.1 AI_WORKFLOW.md パターン（最重要）

```markdown
# 厳格なタスクフォーマット

### TASK-XXX | タスクタイトル

**Priority**: High | **Category**: Backend | **Assigned**: @worker-1
**Created**: 2025-01-20 | **Started**: 2025-01-21 | **Finished**: 2025-01-22
**Tags**: #feature #api

タスクの説明文（## や ### は禁止）

**Subtasks**:
- [ ] サブタスク1
- [x] 完了したサブタスク

**Notes**:
**Result**: 実装完了
**Modified files**: src/api.go (lines 42-58)
```

**パクるべき設計:**
- `TASK-XXX` 形式のユニークID（自動採番）
- `<!-- Config: Last Task ID: XXX -->` によるID管理
- `**Subtasks**:` と `**Notes**:` のセクション分け
- Git連携: `git commit -m "feat: Add feature (TASK-042 - 3/5)"`

### 2.2 AI設定ファイルパターン

```
各AIアシスタント用の設定ファイル配置:
- Claude: CLAUDE.md（プロジェクトルート）
- Copilot: .github/copilot-instructions.md
- Gemini: .gemini/instructions.md

共通ワークフローは AI_WORKFLOW.md に集約
```

**パクるべき設計:**
- エージェント非依存の共通ワークフロー定義
- エージェント固有設定は別ファイルに分離

### 2.3 アーカイブ戦略

```
ワークフロー:
1. 完了タスク → "✅ Done" に残す
2. ユーザー要求時のみ → archive.md に移動
3. 自動アーカイブしない

理由:
- 人間が確認できる
- 履歴として参照しやすい
```

---

## 3. Aider からパクれる要素

### 3.1 Repo Map（コードベースマッピング）

```
特徴:
- コードベース全体の構造を把握
- 大規模プロジェクトでも効率的に動作
- シンボル（関数、クラス）の関係性を理解

パクるべき設計:
- ファイル構造 + シンボルのインデックス化
- タスク割り当て時のコンテキスト提供
```

### 3.2 Git自動コミット

```
特徴:
- 変更を自動で sensible なコミットメッセージでコミット
- 差分の管理が容易

パクるべき設計:
- Worker完了時の自動コミット
- TASK-XXX 参照付きコミットメッセージ
```

### 3.3 100+言語サポートのアーキテクチャ

```
特徴:
- 言語ごとにパーサーを切り替え
- Tree-sitterベースのシンボル抽出

検討:
- AIDAでは言語サポートはエージェント側に委譲
- Repo Mapは汎用的なファイル構造把握に留める
```

---

## 4. CrewAI からパクれる要素

### 4.1 Agent/Task/Crew 構造

```python
# CrewAIの概念モデル
Agent = 役割 + 目標 + バックストーリー + ツール
Task = 説明 + 期待出力 + 担当Agent
Crew = Agents[] + Tasks[] + Process（sequential/hierarchical）
```

**パクるべき設計:**
```go
// AIDAでの対応
type Worker struct {
    ID          string
    Role        string        // 役割（例: "frontend", "backend"）
    Agent       AgentAdapter  // 実際のエージェント（Claude Code等）
    Capabilities []Capability
}

type Task struct {
    ID          string
    Description string
    AssignedTo  string        // Worker ID
    DependsOn   []string      // 依存タスク
    Context     map[string]any
}

type Orchestrator struct {
    Workers []Worker
    Tasks   []Task
    Mode    ExecutionMode  // Sequential | Parallel | Hierarchical
}
```

### 4.2 Flows（イベント駆動制御）

```
CrewAI Flows:
- 細粒度のイベント駆動制御
- 単一LLM呼び出しの精密なタスクオーケストレーション
- Crewsとネイティブに連携

検討:
- AIDAでは Redis Pub/Sub でイベント駆動を実現
- Commander → Worker → Aggregator のフロー
```

---

## 5. go-git からパクれる要素

### 5.1 Worktree操作

```go
// Worktree作成
w, err := repo.Worktree()

// 新しいworktreeを追加
repo.AddWorktree("../feature-branch", &git.AddWorktreeOptions{
    Branch: plumbing.NewBranchReferenceName("feature-branch"),
})
```

**パクるべき設計:**
- 各Workerに独立したworktreeを割り当て
- ブランチ競合を防止
- マージ時の競合検出

### 5.2 In-memory操作

```go
// メモリ上でリポジトリ操作（高速）
r, err := git.Clone(memory.NewStorage(), nil, &git.CloneOptions{
    URL: "https://github.com/example/repo",
})
```

**検討:**
- 差分計算やログ取得はin-memoryで高速化可能

---

## 6. Bubbletea からパクれる要素

### 6.1 Elm Architecture パターン

```go
// Model: アプリケーション状態
type model struct {
    workers  []Worker
    tasks    []Task
    cursor   int
    selected map[int]struct{}
}

// Init: 初期コマンド
func (m model) Init() tea.Cmd {
    return loadTasks
}

// Update: イベントハンドリング
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "q":
            return m, tea.Quit
        case "enter":
            return m, assignTask(m.tasks[m.cursor])
        }
    case taskCompletedMsg:
        m.tasks[msg.id].Status = "done"
    }
    return m, nil
}

// View: UI描画
func (m model) View() string {
    return renderKanban(m.tasks)
}
```

### 6.2 Bubbles コンポーネント

```
利用可能なコンポーネント:
- spinner: ローディング表示
- progress: 進捗バー
- table: タスク一覧表示
- viewport: スクロール可能なログビュー
- textinput: コマンド入力

パクるべき設計:
- 既存コンポーネントを組み合わせてTUIを構築
- カスタムスタイルは lipgloss で定義
```

---

## 実装優先度

### Phase 1（即時採用）
| 要素 | ソース | 理由 |
|------|--------|------|
| Markdown Protocol強化 | MarkdownTaskManager | TASK-XXX形式、厳格フォーマット |
| AI_WORKFLOW.md | MarkdownTaskManager | エージェント非依存のワークフロー定義 |
| Elm Architecture TUI | Bubbletea | Go標準のTUIパターン |

### Phase 2（v0.2で採用）
| 要素 | ソース | 理由 |
|------|--------|------|
| Worktree管理 | go-git | 並列作業の基盤 |
| Git自動コミット | Aider | TASK参照付きコミット |
| Agent/Task構造 | CrewAI | オーケストレーションモデル |

### Phase 3（v0.3以降）
| 要素 | ソース | 理由 |
|------|--------|------|
| Repo Map | Aider | 大規模プロジェクト対応 |
| MCP設定中央管理 | Vibe Kanban | マルチエージェント設定 |
| イベント駆動フロー | CrewAI Flows | 高度なオーケストレーション |

---

## 具体的なコード参照

### MarkdownTaskManager
- `AI_WORKFLOW.md` - 完全なワークフロー定義（そのまま参考可）
- `CLAUDE.md.exemple` - エージェント設定テンプレート

### go-git
- `_examples/` ディレクトリ - 全操作のサンプルコード
- `worktree.go` - Worktree操作の実装

### Bubbletea
- `tutorials/basics/` - チュートリアルコード
- `examples/` - 様々なUIパターン

### CrewAI
- `src/crewai/agent.py` - Agent定義
- `src/crewai/task.py` - Task定義
- `src/crewai/crew.py` - オーケストレーション

---

## 注意点

1. **ライセンス確認**: 各OSSのライセンスを確認し、適切に帰属表示
2. **過度な抽象化回避**: 必要な部分だけを取り入れる
3. **Go idiom優先**: Python/Rust由来の概念はGoらしく再設計
4. **エージェント非依存維持**: 特定エージェントに依存する設計は避ける
