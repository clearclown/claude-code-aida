# Claude Code Profile Manager

**Claude Code専用 プロファイル管理ツール**

[![Go Version](https://img.shields.io/badge/Go-1.22+-00ADD8?style=flat&logo=go)](https://go.dev/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

---

## ⚠️ Platform Dependency Notice

このプロジェクトは **Claude Code に依存** しています。
Claude Codeが廃止された場合、このプロジェクトも役割を終えます。

汎用的なオーケストレーション・可視化には [AIDA](../aida) / [AINI](../aini) を使用してください。

---

## Overview

Claude Code Profile Managerは、プロジェクトごとのClaude Code設定を管理するツールです。

**管理対象**:
- MCP設定 (`~/.mcp.json`)
- Claude設定 (`~/.claude/settings.json`, `settings.local.json`)
- Skills
- Hooks
- CLAUDE.md

```
┌─────────────────────────────────────────────────────────────────┐
│                   Profile Manager                                │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐          │
│  │    Base     │ +  │   Layers    │ +  │   Project   │ = Active │
│  │   Config    │    │             │    │   Config    │   Config │
│  └─────────────┘    └─────────────┘    └─────────────┘          │
│                                                                  │
│         base/            layers/           projects/             │
│       mcp.json      terraform/mcp.json   my-app/mcp.json        │
│       hooks.yaml    docker/mcp.json                              │
│                     testing/mcp.json                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Related Projects

| Project | Description | Platform |
|---------|-------------|----------|
| [AIDA](../aida) | オーケストレーション | 汎用（永続） |
| [AINI](../aini) | 状態可視化UI | 汎用（永続） |
| **CCPM** (this) | Claude Code設定管理 | Claude Code専用 |

---

## Installation

```bash
# Go 1.22+ required
go install github.com/ablaze/ccpm/cmd/ccpm@latest

# Or build from source
git clone https://github.com/ablaze/ccpm.git
cd ccpm
go build -o ccpm ./cmd/ccpm
```

---

## Quick Start

```bash
# 1. 初期化
ccpm init

# 2. プロファイル作成
ccpm profile create my-project

# 3. レイヤー追加
ccpm layer add terraform
ccpm layer add docker

# 4. プロファイル適用
ccpm profile use my-project

# 5. 状態確認
ccpm status
```

---

## Commands

### Profile Management

| Command | Description |
|---------|-------------|
| `ccpm profile list` | プロファイル一覧 |
| `ccpm profile create <name>` | 新規プロファイル作成 |
| `ccpm profile use <name>` | プロファイル適用 |
| `ccpm profile edit <name>` | プロファイル編集 |
| `ccpm profile delete <name>` | プロファイル削除 |
| `ccpm profile export` | 現在の設定をエクスポート |

### Layer Management

| Command | Description |
|---------|-------------|
| `ccpm layer list` | 利用可能なレイヤー一覧 |
| `ccpm layer add <name>` | レイヤーを追加 |
| `ccpm layer remove <name>` | レイヤーを削除 |
| `ccpm layer create <name>` | 新規レイヤー作成 |

### Status & Info

| Command | Description |
|---------|-------------|
| `ccpm status` | 現在の状態表示 |
| `ccpm diff` | 適用中と保存済みの差分表示 |
| `ccpm backup` | 現在の設定をバックアップ |
| `ccpm restore` | バックアップから復元 |

---

## Configuration Structure

### Directory Layout

```
~/.ccpm/
├── config.yaml           # グローバル設定
├── base/                 # ベース設定（全プロジェクト共通）
│   ├── mcp.json
│   ├── settings.json
│   ├── hooks.yaml
│   └── skills/
├── layers/               # 機能レイヤー（選択式）
│   ├── terraform/
│   │   └── mcp.json
│   ├── docker/
│   │   └── mcp.json
│   ├── testing/
│   │   └── mcp.json
│   └── aws/
│       └── mcp.json
├── profiles/             # プロファイル定義
│   ├── web-dev.yaml
│   ├── data-science.yaml
│   └── infrastructure.yaml
└── backups/              # バックアップ
```

### Profile Definition

```yaml
# ~/.ccpm/profiles/web-dev.yaml
name: web-dev
description: Web development profile

# 使用するレイヤー
layers:
  - docker
  - testing

# MCP設定の上書き
mcp:
  mcpServers:
    my-custom-server:
      command: node
      args: ["/path/to/server.js"]

# Hooks
hooks:
  pre-tool-use:
    - command: echo "Running pre-tool hook"

# 有効にするMCPサーバー
enabledMcpServers:
  - context7
  - playwright

# パーミッション
permissions:
  allow:
    - "Bash(npm:*)"
    - "Bash(git:*)"
```

---

## Merge Strategy

設定は以下の順序でマージされます（後が優先）:

```
1. base/           # ベース設定
       ↓
2. layers/*        # 選択されたレイヤー（順序で適用）
       ↓
3. profiles/*.yaml # プロファイル固有設定
       ↓
4. .ccpm/          # プロジェクトディレクトリ固有（あれば）
```

### Merge Rules

| 項目 | ルール |
|------|--------|
| `mcpServers` | サーバー名でディープマージ |
| `enabledMcpjsonServers` | 配列を結合（ユニーク） |
| `hooks` | 配列に追加 |
| `permissions.allow` | 配列を結合（ユニーク） |
| `skills` | ディレクトリをシンボリックリンク |

---

## Built-in Layers

| Layer | Description | MCP Servers |
|-------|-------------|-------------|
| `terraform` | Terraform/IaC開発 | terraform-mcp |
| `docker` | コンテナ開発 | podman-mcp |
| `testing` | テスト自動化 | - |
| `aws` | AWS開発 | aws-api, terraform |
| `gcp` | GCP開発 | gcloud |
| `database` | DB操作 | postgres-mcp |
| `frontend` | フロントエンド開発 | playwright, browser-automation |
| `mobile` | モバイル開発 | mobile-mcp, dart-flutter |

---

## Integration with AIDA

Claude Code Profile ManagerはAIDAの `adapters/claudecode` 内部で使用されます:

```go
// aida/adapters/claudecode/adapter.go
import "github.com/ablaze/ccpm/pkg/profile"

func (a *ClaudeCodeAgent) Setup(task agent.Task) error {
    // プロジェクトに応じたプロファイルをロード
    prof, err := profile.Load(task.WorkDir)
    if err != nil {
        return err
    }

    // プロファイルを適用
    return prof.Apply()
}
```

---

## TUI Mode

```bash
ccpm  # TUIモードで起動
```

```
┌─────────────────────────────────────────────────────────────────┐
│ Claude Code Profile Manager                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│ Current Profile: web-dev                                         │
│ Active Layers: docker, testing                                   │
│                                                                  │
│ ┌─ Profiles ──────────────────────────────────────────────────┐ │
│ │ > web-dev         [active]                                  │ │
│ │   data-science                                              │ │
│ │   infrastructure                                            │ │
│ │   mobile-dev                                                │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ ┌─ Available Layers ──────────────────────────────────────────┐ │
│ │ [✓] docker                                                  │ │
│ │ [✓] testing                                                 │ │
│ │ [ ] terraform                                               │ │
│ │ [ ] aws                                                     │ │
│ │ [ ] database                                                │ │
│ └─────────────────────────────────────────────────────────────┘ │
│                                                                  │
│ [Enter] Apply  [Space] Toggle Layer  [e] Edit  [n] New  [q] Quit │
└─────────────────────────────────────────────────────────────────┘
```

---

## Safety Features

1. **バックアップ自動作成**: 設定変更前に自動バックアップ
2. **アトミック書き込み**: 一時ファイル経由で安全に書き込み
3. **Dry-run モード**: `--dry-run` で変更内容を事前確認
4. **ロールバック**: `ccpm restore` で直前の状態に復元

---

## License

MIT License - see [LICENSE](LICENSE) for details.
