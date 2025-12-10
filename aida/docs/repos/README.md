# Reference Repositories

参考にするOSSプロジェクトをクローンするディレクトリ。

## クローン方法

```bash
cd aida/docs/repos

# Markdown Protocol参考
git clone --depth 1 https://github.com/ioniks/MarkdownTaskManager.git

# マルチエージェントオーケストレーション参考
git clone --depth 1 https://github.com/BloopAI/vibe-kanban.git

# Go Git操作参考
git clone --depth 1 https://github.com/go-git/go-git.git

# TUI実装参考
git clone --depth 1 https://github.com/charmbracelet/bubbletea.git
```

## 参照ポイント

| Project | 参照すべきファイル |
|---------|-------------------|
| MarkdownTaskManager | `AI_WORKFLOW.md`, `CLAUDE.md.exemple` |
| vibe-kanban | バックエンド構造、worktree管理 |
| go-git | `_examples/` ディレクトリ |
| bubbletea | `examples/`, `tutorials/` |

詳細は `oss-adoption-plan.md` を参照。
