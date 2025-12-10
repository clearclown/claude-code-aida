# Project Overview - Human-Directed Autonomous AI System

## Vision

**人間主導の自律型AI**による効率的なタスク遂行を実現する3つのプロジェクト。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Human Operator                                   │
│                    (意思決定・方向性・承認)                               │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                      │
          ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│      AINI       │    │      AIDA       │    │      CCPM       │
│   可視化・UI     │←──→│ オーケストレーション│←──→│ プロファイル管理 │
│                 │    │                 │    │                 │
│  永続・汎用     │    │   永続・汎用     │    │ Claude Code依存 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
          │                      │
          └──────────┬───────────┘
                     │
                     ▼
          ┌─────────────────────┐
          │   Agent Interface   │
          │    (抽象化レイヤー)   │
          └──────────┬──────────┘
                     │
     ┌───────────────┼───────────────┐
     ▼               ▼               ▼
┌─────────┐    ┌─────────┐    ┌─────────┐
│ Claude  │    │ Gemini  │    │  Codex  │
│  Code   │    │   CLI   │    │   CLI   │
└─────────┘    └─────────┘    └─────────┘
```

---

## Three Projects

### 1. AIDA (Agent Integration & Development Architecture)

**役割**: AIエージェントのオーケストレーション

| 項目 | 内容 |
|------|------|
| 寿命 | **永続**（プラットフォーム非依存） |
| 言語 | Go |
| 主機能 | タスク分割、Worker管理、結果集約 |

**特徴**:
- Agent Interface による抽象化
- 任意のAIエージェントに対応
- Git worktree による並列作業
- Markdown Protocol による状態共有

### 2. AINI (Agent Interface & Navigation Intelligence)

**役割**: AIエージェントの状態可視化

| 項目 | 内容 |
|------|------|
| 寿命 | **永続**（プラットフォーム非依存） |
| 言語 | TypeScript (Svelte) |
| 主機能 | Kanban、Worker Monitor、Timeline |

**特徴**:
- Markdown Protocol に基づくファイル連携
- リアルタイム状態更新
- 単一HTMLでも動作可能
- AIDA連携（WebSocket）

### 3. Claude Code Profile Manager (CCPM)

**役割**: Claude Code設定のプロファイル管理

| 項目 | 内容 |
|------|------|
| 寿命 | **Claude Code依存**（廃止可能） |
| 言語 | Go |
| 主機能 | MCP/Settings/Hooks の層管理 |

**特徴**:
- Base + Layers + Project の階層設定
- AIDA adapters/claudecode から利用
- Claude Code廃止時は不要になる

---

## Design Principles

### 1. Platform Independence

AIDA と AINI は特定のAIエージェントに依存しない。

```
Claude Code が廃れても:
├── AIDA → 新エージェントの adapter を追加するだけ
├── AINI → 変更不要（Markdown Protocol は共通）
└── CCPM → 役割終了（削除可能）
```

### 2. Human in the Loop

AIは自律的に作業するが、最終決定権は人間にある。

```
Human:  タスク定義 → 承認 → フィードバック → 最終確認
          ↓           ↑         ↓            ↑
AI:    タスク分解 → 実行 → 報告 → 修正実行
```

### 3. Markdown as Protocol

状態共有にMarkdownを使用することで:

- **人間が読める**: 特別なツールなしで状態確認
- **Git管理可能**: 履歴追跡、diff可視化
- **AIが操作可能**: どのエージェントも読み書き可
- **ツール非依存**: AINIがなくても状態は保持される

---

## Data Flow

```
1. Human が kanban.md にタスクを追加（直接編集 or AINI経由）
                    │
                    ▼
2. AIDA が kanban.md を監視、新タスクを検出
                    │
                    ▼
3. Commander がタスクを分析・分割
                    │
                    ▼
4. Worker を起動、各自 Git worktree で作業
   ├── Worker-1: feature/task-001
   ├── Worker-2: feature/task-002
   └── Worker-3: feature/task-003
                    │
                    ▼
5. 各 Worker が Agent Interface 経由で AI エージェントを呼び出し
   ├── Worker-1 → Claude Code
   ├── Worker-2 → Gemini CLI
   └── Worker-3 → Claude Code
                    │
                    ▼
6. 進捗を kanban.md に記録（AINI がリアルタイム表示）
                    │
                    ▼
7. 完了後、Aggregator が結果をマージ
                    │
                    ▼
8. Human がレビュー・承認
                    │
                    ▼
9. マージ完了、タスクを Done に移動、archive.md に記録
```

---

## Repository Structure

```
github.com/ablaze/
├── aida/                           # オーケストレーション
│   ├── cmd/aida/
│   ├── internal/
│   ├── pkg/agent/                  # Agent Interface (公開)
│   ├── adapters/
│   │   ├── claudecode/             # → CCPM を利用
│   │   ├── geminicli/
│   │   └── codexcli/
│   └── docs/
│       ├── architecture.md
│       ├── agent-interface.md
│       └── markdown-protocol.md
│
├── aini/                           # 状態可視化UI
│   ├── frontend/
│   ├── src-tauri/
│   ├── standalone/
│   └── docs/
│       └── features.md
│
└── claude-code-profile-manager/    # Claude Code設定管理
    ├── cmd/ccpm/
    ├── internal/
    ├── pkg/profile/                # AIDA連携用API
    └── configs/builtin-layers/
```

---

## Implementation Roadmap

### Phase 1: Foundation
- [ ] AIDA: Agent Interface 定義
- [ ] AIDA: Claude Code adapter
- [ ] AINI: Kanban Board 基本実装
- [ ] CCPM: プロファイル読み書き

### Phase 2: Core Features
- [ ] AIDA: Commander / Worker / Aggregator
- [ ] AIDA: Git worktree 統合
- [ ] AINI: Worker Monitor
- [ ] CCPM: Layer マージ

### Phase 3: Integration
- [ ] AIDA ↔ AINI 連携
- [ ] AIDA ↔ CCPM 連携
- [ ] Markdown Protocol 完全準拠

### Phase 4: Polish
- [ ] AIDA: TUI
- [ ] AINI: Desktop版 (Tauri)
- [ ] CCPM: TUI
- [ ] 追加エージェント adapter

---

## Success Criteria

1. **人間は意思決定に集中できる**: 細かい作業はAIに任せられる
2. **状態が常に可視化されている**: 何が起きているか把握できる
3. **エージェントを自由に選べる**: ベンダーロックインなし
4. **履歴が残る**: Markdownベースでgit管理可能
5. **安全**: バックアップ、ロールバック、承認フロー

---

## Naming Origin

| Name | Meaning |
|------|---------|
| **AIDA** | Agent Integration & Development Architecture |
| **AINI** | Agent Interface & Navigation Intelligence |
| **CCPM** | Claude Code Profile Manager |

※ AIDA/AINI は汎用的な名前であり、Claude Codeに縛られない。
