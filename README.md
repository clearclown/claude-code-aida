# Human-Directed Autonomous AI System

**人間主導の自律型AI** による効率的なタスク遂行を実現する統合フレームワーク

---

## Projects

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Human Operator                                   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│      AINI       │    │      AIDA       │    │      CCPM       │
│   可視化・UI     │◄──►│ オーケストレーション│◄──►│ プロファイル管理 │
│                 │    │                 │    │                 │
│  永続・汎用     │    │   永続・汎用     │    │ Claude Code依存 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

| Project | Description | Status |
|---------|-------------|--------|
| [**AIDA**](./aida) | Agent Integration & Development Architecture | 計画中 |
| [**AINI**](./aini) | Agent Interface & Navigation Intelligence | 計画中 |
| [**CCPM**](./ccpm) | Claude Code Profile Manager | 計画中 |

---

## Architecture

### Platform Independence

AIDA と AINI は特定のAIエージェントに依存しません。

```
                     ┌─────────────────┐
                     │  Agent Interface │
                     └────────┬────────┘
                              │
     ┌────────────────────────┼────────────────────────┐
     ▼                        ▼                        ▼
┌─────────┐             ┌─────────┐             ┌─────────┐
│ Claude  │             │ Gemini  │             │  Codex  │
│  Code   │             │   CLI   │             │   CLI   │
└─────────┘             └─────────┘             └─────────┘
```

### Communication

```
AINI ◄──── Redis Pub/Sub ────► AIDA
  │                              │
  └──────► kanban.md ◄───────────┘
           (永続化)
```

---

## Quick Links

### AIDA (Orchestration)
- [README](./aida/README.md)
- [Architecture](./aida/docs/architecture.md)
- [Agent Interface](./aida/docs/agent-interface.md)
- [Markdown Protocol](./aida/docs/markdown-protocol.md)
- [Redis Protocol](./aida/docs/redis-protocol.md)

### AINI (Visualization)
- [README](./aini/README.md)
- [Features](./aini/docs/features.md)

### CCPM (Profile Manager)
- [README](./ccpm/README.md)

---

## Tech Stack

| Component | Technology |
|-----------|------------|
| AIDA/CCPM | Go 1.22+, Cobra, Bubbletea |
| AINI | TypeScript, Svelte 5, Tauri |
| Communication | Redis, Markdown |

---

## License

MIT License
