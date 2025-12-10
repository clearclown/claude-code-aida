# CLAUDE.md - Claude Code Profile Manager Instructions

このドキュメントはClaude Codeがこのプロジェクトで作業する際のガイドラインです。

---

## ⚠️ Platform Dependency Warning

**このプロジェクトは Claude Code に依存しています。**

Claude Codeが廃止された場合、このプロジェクトは不要になります。
汎用的な機能は [AIDA](../aida) / [AINI](../aini) に実装してください。

---

## Project Identity

**CCPM** (Claude Code Profile Manager)
- **役割**: Claude Code専用のプロファイル管理
- **依存**: Claude Code CLI
- **寿命**: Claude Code と共に

### 関連プロジェクト
| Project | 役割 | 依存関係 |
|---------|------|----------|
| **AIDA** | オーケストレーション | 永続（エージェント非依存） |
| **AINI** | 状態可視化UI | 永続（エージェント非依存） |
| **CCPM** (this) | Claude Code設定管理 | Claude Code依存（廃止可能） |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                          CCPM                                    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                      CLI / TUI                            │   │
│  │  cobra (CLI) | bubbletea (TUI)                           │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                   Profile Manager                         │   │
│  │  Load | Merge | Apply | Backup | Restore                 │   │
│  └──────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Config Writers                         │   │
│  │  MCP | Settings | Hooks | Skills                         │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Claude Code Config Files                       │
│                                                                  │
│  ~/.mcp.json                                                     │
│  ~/.claude/settings.json                                         │
│  ~/.claude/settings.local.json                                   │
│  ~/.claude/commands/                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Tech Stack

| Component | Technology | 理由 |
|-----------|------------|------|
| Language | Go 1.22+ | 単一バイナリ配布 |
| CLI | cobra + viper | 標準的 |
| TUI | bubbletea + lipgloss | リッチUI |
| Config | YAML + JSON | Claude Code互換 |

---

## Directory Structure

```
ccpm/
├── cmd/
│   └── ccpm/
│       └── main.go
├── internal/
│   ├── profile/
│   │   ├── profile.go      # Profile構造体
│   │   ├── loader.go       # 読み込み
│   │   ├── merger.go       # マージロジック
│   │   └── applier.go      # 適用
│   ├── layer/
│   │   ├── layer.go        # Layer構造体
│   │   └── registry.go     # 組み込みレイヤー
│   ├── config/
│   │   ├── mcp.go          # mcp.json操作
│   │   ├── settings.go     # settings.json操作
│   │   └── hooks.go        # hooks操作
│   ├── backup/
│   │   ├── backup.go       # バックアップ
│   │   └── restore.go      # リストア
│   └── tui/
│       ├── app.go
│       └── views/
├── pkg/
│   └── profile/            # 公開API（AIDA連携用）
│       └── api.go
├── configs/
│   └── builtin-layers/     # 組み込みレイヤー
│       ├── terraform/
│       ├── docker/
│       └── aws/
├── go.mod
├── README.md
├── CLAUDE.md
└── LICENSE
```

---

## Core Types

### Profile

```go
// internal/profile/profile.go
package profile

type Profile struct {
    Name        string            `yaml:"name"`
    Description string            `yaml:"description,omitempty"`
    Layers      []string          `yaml:"layers,omitempty"`
    MCP         *MCPConfig        `yaml:"mcp,omitempty"`
    Settings    *SettingsConfig   `yaml:"settings,omitempty"`
    Hooks       *HooksConfig      `yaml:"hooks,omitempty"`
    Permissions *PermissionsConfig `yaml:"permissions,omitempty"`
}

type MCPConfig struct {
    MCPServers map[string]MCPServer `yaml:"mcpServers,omitempty"`
    Enabled    []string             `yaml:"enabledMcpjsonServers,omitempty"`
}

type MCPServer struct {
    Command string            `json:"command"`
    Args    []string          `json:"args,omitempty"`
    Env     map[string]string `json:"env,omitempty"`
}
```

### Layer

```go
// internal/layer/layer.go
package layer

type Layer struct {
    Name        string     `yaml:"name"`
    Description string     `yaml:"description,omitempty"`
    MCP         *MCPConfig `yaml:"mcp,omitempty"`
    Settings    *Settings  `yaml:"settings,omitempty"`
    Hooks       *Hooks     `yaml:"hooks,omitempty"`
}
```

---

## Merge Algorithm

```go
// internal/profile/merger.go

func Merge(base, overlay *Profile) *Profile {
    result := base.DeepCopy()

    // MCPServers: サーバー名でディープマージ
    for name, server := range overlay.MCP.MCPServers {
        if existing, ok := result.MCP.MCPServers[name]; ok {
            result.MCP.MCPServers[name] = mergeServer(existing, server)
        } else {
            result.MCP.MCPServers[name] = server
        }
    }

    // EnabledServers: 配列を結合（重複排除）
    result.MCP.Enabled = unique(append(
        result.MCP.Enabled,
        overlay.MCP.Enabled...,
    ))

    // Hooks: 配列に追加
    result.Hooks.PreToolUse = append(
        result.Hooks.PreToolUse,
        overlay.Hooks.PreToolUse...,
    )

    // Permissions: 結合
    result.Permissions.Allow = unique(append(
        result.Permissions.Allow,
        overlay.Permissions.Allow...,
    ))

    return result
}
```

---

## Claude Code Config Paths

```go
// internal/config/paths.go
package config

import (
    "os"
    "path/filepath"
)

func MCPConfigPath() string {
    return filepath.Join(os.Getenv("HOME"), ".mcp.json")
}

func SettingsPath() string {
    return filepath.Join(os.Getenv("HOME"), ".claude", "settings.json")
}

func SettingsLocalPath() string {
    return filepath.Join(os.Getenv("HOME"), ".claude", "settings.local.json")
}

func CommandsDir() string {
    return filepath.Join(os.Getenv("HOME"), ".claude", "commands")
}
```

---

## Implementation Priority

### Phase 1: Core (v0.1)
1. Profile構造体定義
2. 読み込み・保存
3. `ccpm profile list`
4. `ccpm profile use`

### Phase 2: Merge & Layer (v0.2)
1. マージロジック
2. Layer管理
3. `ccpm layer add/remove`
4. 組み込みレイヤー

### Phase 3: Safety (v0.3)
1. バックアップ
2. リストア
3. Dry-run
4. アトミック書き込み

### Phase 4: TUI (v0.4)
1. bubbletea TUI
2. プロファイル選択
3. レイヤートグル
4. プレビュー

---

## Coding Standards

### Error Handling

```go
// Good: 詳細なエラー
func LoadProfile(name string) (*Profile, error) {
    path := profilePath(name)
    data, err := os.ReadFile(path)
    if err != nil {
        if os.IsNotExist(err) {
            return nil, fmt.Errorf("profile %q not found: %w", name, ErrNotFound)
        }
        return nil, fmt.Errorf("failed to read profile %q: %w", name, err)
    }
    // ...
}
```

### Atomic Write

```go
// Good: アトミック書き込み
func WriteConfig(path string, data []byte) error {
    // 1. 一時ファイルに書き込み
    tmp := path + ".tmp"
    if err := os.WriteFile(tmp, data, 0600); err != nil {
        return fmt.Errorf("failed to write temp file: %w", err)
    }

    // 2. リネーム（アトミック）
    if err := os.Rename(tmp, path); err != nil {
        os.Remove(tmp) // クリーンアップ
        return fmt.Errorf("failed to rename: %w", err)
    }

    return nil
}
```

### Backup Before Write

```go
func ApplyProfile(profile *Profile) error {
    // 1. 現在の設定をバックアップ
    if err := backup.Create("pre-apply"); err != nil {
        return fmt.Errorf("failed to create backup: %w", err)
    }

    // 2. 設定を適用
    if err := writeConfigs(profile); err != nil {
        // ロールバック
        backup.Restore("pre-apply")
        return fmt.Errorf("failed to apply, rolled back: %w", err)
    }

    return nil
}
```

---

## AIDA Integration API

```go
// pkg/profile/api.go
package profile

// Load はワークディレクトリのプロファイルをロードする
func Load(workDir string) (*Profile, error)

// Apply は現在のプロファイルをClaude Code設定に適用する
func Apply(profile *Profile) error

// Current は現在適用中のプロファイルを返す
func Current() (*Profile, error)

// Backup は現在の設定をバックアップする
func Backup(name string) error

// Restore はバックアップから復元する
func Restore(name string) error
```

---

## Testing

```go
// internal/profile/merger_test.go
func TestMerge_MCPServers(t *testing.T) {
    base := &Profile{
        MCP: &MCPConfig{
            MCPServers: map[string]MCPServer{
                "server1": {Command: "cmd1"},
            },
        },
    }

    overlay := &Profile{
        MCP: &MCPConfig{
            MCPServers: map[string]MCPServer{
                "server2": {Command: "cmd2"},
            },
        },
    }

    result := Merge(base, overlay)

    assert.Len(t, result.MCP.MCPServers, 2)
    assert.Equal(t, "cmd1", result.MCP.MCPServers["server1"].Command)
    assert.Equal(t, "cmd2", result.MCP.MCPServers["server2"].Command)
}
```

---

## Notes for Claude

- **段階的実装**: Phase 1から順に
- **安全第一**: バックアップ、アトミック書き込みを必ず実装
- **Claude Code設定を壊さない**: 慎重にテスト
- **AIDA連携を意識**: `pkg/profile` は公開API
- **日本語コメント可**: コード内コメントは日本語でも可
