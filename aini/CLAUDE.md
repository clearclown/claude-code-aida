# CLAUDE.md - AINI Project Instructions

このドキュメントはClaude Codeがこのプロジェクトで作業する際のガイドラインです。

---

## Project Identity

**AINI** (Agent Interface & Navigation Intelligence)
- **役割**: 汎用AIエージェント 状態可視化フレームワーク
- **設計思想**: プラットフォーム非依存（Claude Code以外のエージェントにも対応）

### 関連プロジェクト
| Project | 役割 | 依存関係 |
|---------|------|----------|
| **AIDA** | オーケストレーション | 永続（エージェント非依存） |
| **AINI** (this) | 状態可視化UI | 永続（エージェント非依存） |
| **Claude-Code-Profile-Manager** | Claude Code設定管理 | Claude Code依存（廃止可能） |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                          AINI                                    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      Frontend                             │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │   │
│  │  │ Kanban  │  │ Worker  │  │Timeline │  │ Config  │      │   │
│  │  │ Board   │  │ Monitor │  │  View   │  │  Panel  │      │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      State Store                          │   │
│  │  tasks | workers | timeline | settings                    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Markdown Protocol                       │   │
│  │  Parser | Serializer | File Watcher | Sync Engine        │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │   kanban.md     │
                    │   archive.md    │
                    │   logs/*.md     │
                    └─────────────────┘
```

---

## Tech Stack

| Component | Technology | 理由 |
|-----------|------------|------|
| Framework | Svelte 5 | 軽量、リアクティブ、学習コスト低 |
| Build | Vite | 高速ビルド |
| Desktop | Tauri | 軽量、Rust製 |
| Styling | Tailwind CSS | ユーティリティファースト |
| State | Svelte Stores | シンプルな状態管理 |
| Markdown | marked + custom | 拡張可能 |

---

## Directory Structure

```
aini/
├── frontend/
│   ├── src/
│   │   ├── components/
│   │   │   ├── kanban/
│   │   │   │   ├── Board.svelte
│   │   │   │   ├── Column.svelte
│   │   │   │   └── Card.svelte
│   │   │   ├── workers/
│   │   │   │   ├── Monitor.svelte
│   │   │   │   ├── WorkerCard.svelte
│   │   │   │   └── LogViewer.svelte
│   │   │   ├── timeline/
│   │   │   │   └── Timeline.svelte
│   │   │   └── common/
│   │   │       ├── Modal.svelte
│   │   │       └── Button.svelte
│   │   ├── stores/
│   │   │   ├── tasks.ts
│   │   │   ├── workers.ts
│   │   │   └── settings.ts
│   │   ├── lib/
│   │   │   ├── markdown/
│   │   │   │   ├── parser.ts
│   │   │   │   └── serializer.ts
│   │   │   ├── sync/
│   │   │   │   ├── file-watcher.ts
│   │   │   │   └── sync-engine.ts
│   │   │   └── aida/
│   │   │       └── client.ts     # AIDA連携
│   │   ├── App.svelte
│   │   └── main.ts
│   ├── public/
│   └── index.html
├── src-tauri/                    # Tauri (Desktop版)
│   ├── src/
│   │   ├── main.rs
│   │   ├── file_watcher.rs
│   │   └── commands.rs
│   ├── Cargo.toml
│   └── tauri.conf.json
├── standalone/
│   └── aini.html                 # 単一HTMLバージョン
├── docs/
│   └── features.md
├── package.json
├── README.md
├── CLAUDE.md
└── LICENSE
```

---

## Core Components

### 1. Markdown Parser

```typescript
// lib/markdown/parser.ts

interface Task {
  id: string;
  description: string;
  status: 'todo' | 'in_progress' | 'done';
  worker?: string;
  metadata: Record<string, string>;
}

interface KanbanState {
  projectName: string;
  meta: Record<string, string>;
  tasks: Task[];
  workers: Worker[];
}

function parseKanban(content: string): KanbanState {
  // Markdown Protocol に従って解析
}
```

### 2. File Watcher

```typescript
// lib/sync/file-watcher.ts

interface FileWatcherOptions {
  path: string;
  interval: number;
  debounce: number;
}

class FileWatcher {
  watch(callback: (content: string) => void): void;
  stop(): void;
}
```

### 3. Sync Engine

```typescript
// lib/sync/sync-engine.ts

class SyncEngine {
  // 双方向同期
  syncFromFile(): void;
  syncToFile(state: KanbanState): void;

  // 競合解決
  resolveConflict(local: KanbanState, remote: KanbanState): KanbanState;
}
```

---

## Implementation Priority

### Phase 1: Core UI (v0.1)
1. Kanban Board 基本実装
2. Markdown Parser
3. ファイル読み込み
4. 手動リロード

### Phase 2: Real-time Sync (v0.2)
1. File Watcher 実装
2. 双方向同期
3. 競合検出
4. 自動リロード

### Phase 3: Worker Monitor (v0.3)
1. Worker状態表示
2. ログビューア
3. 進捗バー
4. AIDA連携

### Phase 4: Desktop & Polish (v0.4)
1. Tauri統合
2. Timeline View
3. テーマ切り替え
4. 設定パネル

---

## Coding Standards

### Svelte Style

```svelte
<script lang="ts">
  // 型定義
  interface Props {
    task: Task;
    onUpdate: (task: Task) => void;
  }

  // Props
  let { task, onUpdate }: Props = $props();

  // Reactive
  let progress = $derived(task.metadata.progress ?? 0);
</script>

<div class="task-card">
  <h3>{task.description}</h3>
  <progress value={progress} max="100" />
</div>

<style>
  .task-card {
    @apply p-4 bg-white rounded-lg shadow;
  }
</style>
```

### TypeScript Style

```typescript
// Good: 型を明示
function parseTask(line: string): Task | null {
  // ...
}

// Good: 早期リターン
function validateTask(task: Task): boolean {
  if (!task.id) return false;
  if (!task.description) return false;
  return true;
}
```

### ファイル命名
- **Components**: PascalCase (`TaskCard.svelte`)
- **Stores**: camelCase (`tasks.ts`)
- **Utils**: camelCase (`parser.ts`)

---

## Markdown Protocol Compliance

AINIは [Markdown Protocol](../aida/docs/markdown-protocol.md) に準拠します。

### 必須対応
- [ ] タスクID解析 (`#task-XXX`)
- [ ] Worker解析 (`@worker-X`)
- [ ] チェックボックス状態
- [ ] メタデータ解析
- [ ] セクション認識

### 書き込み時のルール
- 既存のフォーマットを保持
- 不要な空行を追加しない
- メタデータの順序を維持

---

## AIDA Integration

```typescript
// lib/aida/client.ts

interface AIDAClient {
  // 接続
  connect(socketPath: string): Promise<void>;

  // コマンド
  startOrchestration(): Promise<void>;
  spawnWorker(count: number): Promise<void>;
  stopWorker(id: string): Promise<void>;

  // イベント
  onStateChange(callback: (state: any) => void): void;
  onLog(callback: (log: LogEntry) => void): void;
}
```

---

## Notes for Claude

- **段階的実装**: Phase 1から順に実装
- **Svelte 5使用**: Runes (`$state`, `$derived`, `$effect`) を活用
- **軽量優先**: 不要な依存を避ける
- **Markdown Protocol準拠**: 仕様を厳密に守る
- **AIDA連携を意識**: 将来の統合を考慮した設計
- **日本語コメント可**: コード内コメントは日本語でも可
