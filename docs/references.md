# External References & Inspirations

## Overview

AIDA/AINI の設計にあたり参考にしたプロジェクト・記事の一覧。

---

## Task Management & Visualization

### MarkdownTaskManager
- **URL**: https://github.com/ioniks/MarkdownTaskManager
- **概要**: ローカルMarkdownファイルベースのKanbanタスクマネージャー
- **特徴**:
  - 単一HTMLファイルで動作
  - File System Access API でローカルファイルに直接読み書き
  - Git管理可能
  - サーバー不要
- **AINI への影響**:
  - Markdown Protocol の設計思想
  - 単一HTMLバージョンの提供
  - ファイルベースの状態管理

### Vibe Kanban
- **URL**: https://github.com/BloopAI/vibe-kanban
- **概要**: AIコーディングエージェント向けのタスク管理・オーケストレーションツール
- **特徴**:
  - Claude Code, Gemini CLI, Codex CLI など複数エージェント対応
  - 並列・逐次のオーケストレーション
  - MCP設定の一元管理
  - SSH経由のリモートプロジェクトオープン
- **AIDA への影響**:
  - マルチエージェントオーケストレーション
  - Worker管理の設計
  - エージェント切り替え機能

---

## Agent Technologies (Anthropic)

### MCP (Model Context Protocol)
- **URL**: https://www.anthropic.com/news/model-context-protocol
- **発表**: 2024年11月
- **概要**: AIアシスタントと外部システムを接続するオープン標準
- **影響**: MCP統合、Agent Interface設計

### Building Effective Agents
- **URL**: https://www.anthropic.com/news/building-effective-agents
- **概要**: 効果的なエージェント構築のベストプラクティス
- **影響**: Commander/Worker/Aggregator パターン

### Multi-Agent Research System
- **URL**: https://www.anthropic.com/engineering/multi-agent-research-system
- **概要**: Anthropicのマルチエージェントリサーチシステム構築記
- **影響**: Subagent パターン、タスク分割戦略

### Claude Agent SDK
- **URL**: https://www.anthropic.com/engineering/building-agents-with-the-claude-agent-sdk
- **概要**: Claude Agent SDKを使ったエージェント構築
- **影響**: Agent Interface の設計

### Context Engineering
- **URL**: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- **発表**: 2025年9月
- **概要**: コンテキスト管理のベストプラクティス
- **影響**: Worker間のコンテキスト分離

### Agent Skills
- **URL**: https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills
- **発表**: 2025年10月
- **概要**: 手続き的知識のパッケージ化
- **影響**: Skills管理、CCPM設計

### Code Execution with MCP
- **URL**: https://www.anthropic.com/engineering/code-execution-with-mcp
- **発表**: 2025年11月
- **概要**: MCPツールをコード経由で呼び出す効率化
- **影響**: Worker内でのツール呼び出し設計

### Advanced Tool Use
- **URL**: https://www.anthropic.com/engineering/advanced-tool-use
- **発表**: 2025年11月
- **概要**: Tool Search、Programmatic calling、Tool use examples
- **影響**: Agent Interface のCapabilities設計

---

## Summary Article

### エージェント技術の進化まとめ (Qiita)
- **URL**: https://qiita.com/jugyo/items/afd684d7eeb0bf194843
- **概要**: MCP → Subagent → Context Engineering → Skills → Code Execution → Advanced Tool Use の時系列整理
- **影響**: 全体アーキテクチャの設計指針

---

## CLI & TUI Frameworks

### Cobra
- **URL**: https://github.com/spf13/cobra
- **概要**: Go製CLIフレームワーク
- **用途**: AIDA, CCPM のCLI

### Viper
- **URL**: https://github.com/spf13/viper
- **概要**: Go製設定管理ライブラリ
- **用途**: AIDA, CCPM の設定読み込み

### Bubbletea
- **URL**: https://github.com/charmbracelet/bubbletea
- **概要**: Go製TUIフレームワーク (Elm Architecture)
- **用途**: AIDA, CCPM のTUI

### Lipgloss
- **URL**: https://github.com/charmbracelet/lipgloss
- **概要**: Go製TUIスタイリングライブラリ
- **用途**: AIDA, CCPM のTUIスタイル

---

## Frontend Frameworks

### Svelte 5
- **URL**: https://svelte.dev/
- **概要**: コンパイル時最適化のUIフレームワーク
- **用途**: AINI フロントエンド

### Tauri
- **URL**: https://tauri.app/
- **概要**: Rust製軽量デスクトップアプリフレームワーク
- **用途**: AINI デスクトップ版

### Tailwind CSS
- **URL**: https://tailwindcss.com/
- **概要**: ユーティリティファーストCSSフレームワーク
- **用途**: AINI スタイリング

---

## Data & Communication

### Redis
- **URL**: https://redis.io/
- **概要**: インメモリデータストア
- **用途**: AIDA/AINI リアルタイム通信

### go-redis
- **URL**: https://github.com/redis/go-redis
- **概要**: Go製Redisクライアント
- **用途**: AIDA Redis接続

### ioredis / redis (Node)
- **URL**: https://github.com/redis/node-redis
- **概要**: Node.js製Redisクライアント
- **用途**: AINI Redis接続

---

## Git Integration

### go-git
- **URL**: https://github.com/go-git/go-git
- **概要**: Pure Go製Gitライブラリ
- **用途**: AIDA Git worktree操作

---

## AI Coding Agents (対応予定)

### Claude Code
- **URL**: https://claude.ai/claude-code
- **概要**: Anthropic公式CLIエージェント
- **状態**: Primary adapter

### Gemini CLI
- **URL**: https://github.com/google-gemini/gemini-cli
- **概要**: Google製CLIエージェント
- **状態**: Planned

### Codex CLI
- **URL**: https://github.com/openai/codex
- **概要**: OpenAI製CLIエージェント
- **状態**: Planned

### Cursor
- **URL**: https://cursor.sh/
- **概要**: AI統合IDE
- **状態**: Planned (API経由)

### Aider
- **URL**: https://github.com/paul-gauthier/aider
- **概要**: オープンソースAIコーディングアシスタント
- **状態**: Potential

### Continue
- **URL**: https://github.com/continuedev/continue
- **概要**: オープンソースAIコーディングアシスタント
- **状態**: Potential

---

## Similar Projects

### Sweep
- **URL**: https://github.com/sweepai/sweep
- **概要**: AI Junior Developer（PR自動作成）
- **参考点**: タスク分解、Git統合

### GPT Engineer
- **URL**: https://github.com/gpt-engineer-org/gpt-engineer
- **概要**: AIによるコード生成
- **参考点**: タスク分解パターン

### AutoGPT
- **URL**: https://github.com/Significant-Gravitas/AutoGPT
- **概要**: 自律型AIエージェント
- **参考点**: 自律実行フロー

### CrewAI
- **URL**: https://github.com/joaomdmoura/crewAI
- **概要**: マルチエージェントフレームワーク
- **参考点**: 役割分担パターン

### LangGraph
- **URL**: https://github.com/langchain-ai/langgraph
- **概要**: LLMワークフローグラフ
- **参考点**: ワークフロー設計

---

## Documentation & Standards

### Conventional Commits
- **URL**: https://www.conventionalcommits.org/
- **概要**: コミットメッセージ規約
- **用途**: Git操作時のコミットメッセージ

### Keep a Changelog
- **URL**: https://keepachangelog.com/
- **概要**: CHANGELOG形式
- **用途**: リリースノート

### Semantic Versioning
- **URL**: https://semver.org/
- **概要**: バージョニング規約
- **用途**: バージョン管理

---

## Notes

### 設計への主な影響

| 参考元 | 影響を受けた部分 |
|--------|------------------|
| MarkdownTaskManager | Markdown Protocol、ファイルベース状態管理 |
| Vibe Kanban | マルチエージェント、Worker管理 |
| Anthropic Blog | Agent Interface、Subagent パターン |
| Cobra/Bubbletea | CLI/TUI設計 |
| Svelte/Tauri | AINI UI |
| Redis | リアルタイム通信 |

### 差別化ポイント

1. **プラットフォーム非依存**: 特定エージェントに縛られない
2. **Markdown Protocol**: 人間が読める状態管理
3. **Human in the Loop**: 自律と人間の承認のバランス
4. **3プロジェクト分離**: 責務の明確化、寿命管理
