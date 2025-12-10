# AIDA - Agent Integration & Development Architecture

**汎用AIエージェント オーケストレーションフレームワーク**

[![Go Version](https://img.shields.io/badge/Go-1.22+-00ADD8?style=flat&logo=go)](https://go.dev/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## Overview

AIDAは、複数のAIコーディングエージェントを協調させるオーケストレーションフレームワークです。

**特徴:**
- **エージェント非依存**: Claude Code, Gemini CLI, Codex CLI など任意のエージェントに対応
- **分業管理**: Commanderがタスクを分割し、複数のWorkerが並列実行
- **Git Worktree統合**: 各Workerが独立したブランチで作業
- **Markdown Protocol**: 状態共有にMarkdownを使用、人間が直接編集可能

```
┌─────────────┐
│  Commander  │  タスク分割・指示
└──────┬──────┘
       │
 ┌─────┼─────┐
 ▼     ▼     ▼
┌───┐ ┌───┐ ┌───┐
│W1 │ │W2 │ │W3 │  Workers（並列実行）
└───┘ └───┘ └───┘
       │
       ▼
┌─────────────┐
│ Aggregator  │  結果集約・マージ
└─────────────┘
```

---

## Related Projects

| Project | Description | Status |
|---------|-------------|--------|
| **AIDA** (this) | オーケストレーション | 開発中 |
| [AINI](../aini) | 状態可視化UI | 計画中 |
| [Claude-Code-Profile-Manager](../claude-code-profile-manager) | Claude Code固有設定管理 | 計画中 |

---

## Installation

```bash
# Go 1.22+ required
go install github.com/ablaze/aida/cmd/aida@latest

# Or build from source
git clone https://github.com/ablaze/aida.git
cd aida
go build -o aida ./cmd/aida
```

---

## Quick Start

```bash
# 1. プロジェクトディレクトリで初期化
aida init

# 2. タスクを定義（kanban.md を編集）
aida task add "認証機能の実装"

# 3. オーケストレーション開始
aida orchestrate --workers 3

# 4. 状態確認
aida status
```

---

## Commands

| Command | Description |
|---------|-------------|
| `aida init` | プロジェクト初期化（kanban.md等を生成） |
| `aida orchestrate` | オーケストレーション開始 |
| `aida spawn <n>` | Worker N個起動 |
| `aida status` | 全Worker状態表示 |
| `aida task add <desc>` | タスク追加 |
| `aida task list` | タスク一覧 |
| `aida aggregate` | 結果集約 |
| `aida logs <worker-id>` | Workerログ表示 |
| `aida kill <worker-id>` | Worker停止 |

---

## Configuration

```yaml
# ~/.aida/config.yaml
default_agent: claude-code
workers:
  max_concurrent: 5
  timeout: 30m
git:
  use_worktree: true
  auto_branch: true
```

---

## Supported Agents

| Agent | Status | Adapter |
|-------|--------|---------|
| Claude Code | ✅ Supported | `adapters/claude-code` |
| Gemini CLI | 🚧 Planned | `adapters/gemini-cli` |
| Codex CLI | 🚧 Planned | `adapters/codex-cli` |
| Cursor | 🚧 Planned | `adapters/cursor` |

新しいエージェントの追加は `Agent Interface` を実装するだけです。

---

## Architecture

詳細は [docs/architecture.md](docs/architecture.md) を参照。

---

## License

MIT License - see [LICENSE](LICENSE) for details.
