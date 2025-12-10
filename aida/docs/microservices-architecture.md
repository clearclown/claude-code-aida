# AIDA Microservices Architecture

## Overview

AIDAをマイクロサービスアーキテクチャで構築し、gRPCで接続、Podmanでコンテナ管理する設計。

---

## アーキテクチャ図

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Host Machine                                  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Podman Pod: aida-pod                         │   │
│  │                                                                   │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │   │
│  │  │   gateway   │  │ orchestrator│  │   worker    │               │   │
│  │  │  :8080/REST │  │  :50051/gRPC│  │  :50052/gRPC│ × N          │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘               │   │
│  │         │                 │                │                      │   │
│  │         └─────────────────┼────────────────┘                      │   │
│  │                           │ gRPC                                  │   │
│  │  ┌─────────────┐  ┌──────┴──────┐  ┌─────────────┐               │   │
│  │  │  aggregator │  │    redis    │  │  file-store │               │   │
│  │  │  :50053/gRPC│  │    :6379    │  │  (volume)   │               │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘               │   │
│  │                                                                   │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │   aida-cli      │  ← TUI / CLI クライアント                           │
│  │  (bubbletea)    │                                                    │
│  └─────────────────┘                                                    │
│                                                                         │
│  ┌─────────────────┐                                                    │
│  │   aini-ui       │  ← Svelte + Tauri デスクトップUI                    │
│  │  (localhost:3000)                                                    │
│  └─────────────────┘                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## サービス構成

### 1. Gateway Service (Go)

**役割**: REST API → gRPC変換、認証、リクエストルーティング

```go
// cmd/gateway/main.go
package main

import (
    "github.com/gin-gonic/gin"
    pb "github.com/yourorg/aida/proto"
)

func main() {
    r := gin.Default()

    // REST endpoints
    r.POST("/api/tasks", createTask)
    r.GET("/api/tasks", listTasks)
    r.GET("/api/workers", listWorkers)
    r.POST("/api/orchestrate", startOrchestration)

    r.Run(":8080")
}
```

**ポート**: 8080 (REST)

---

### 2. Orchestrator Service (Go)

**役割**: タスク分割、Worker割り当て、進捗追跡

```protobuf
// proto/orchestrator.proto
syntax = "proto3";
package aida.orchestrator;

service Orchestrator {
    rpc CreateTask(CreateTaskRequest) returns (Task);
    rpc AssignTask(AssignTaskRequest) returns (Assignment);
    rpc GetProgress(ProgressRequest) returns (stream ProgressUpdate);
    rpc CompleteTask(CompleteTaskRequest) returns (TaskResult);
}

message Task {
    string id = 1;
    string description = 2;
    string priority = 3;
    string category = 4;
    repeated string depends_on = 5;
}
```

**ポート**: 50051 (gRPC)

---

### 3. Worker Service (Go)

**役割**: 個別タスク実行、エージェント呼び出し

```protobuf
// proto/worker.proto
syntax = "proto3";
package aida.worker;

service Worker {
    rpc Execute(ExecuteRequest) returns (stream ExecutionLog);
    rpc GetStatus(StatusRequest) returns (WorkerStatus);
    rpc Stop(StopRequest) returns (StopResponse);
}

message ExecuteRequest {
    string task_id = 1;
    string work_dir = 2;
    string branch = 3;
    string agent = 4; // "claude-code", "gemini-cli", etc.
    map<string, string> context = 5;
}
```

**ポート**: 50052 (gRPC) × N インスタンス

---

### 4. Aggregator Service (Go)

**役割**: 結果収集、マージ、レポート生成

```protobuf
// proto/aggregator.proto
syntax = "proto3";
package aida.aggregator;

service Aggregator {
    rpc CollectResult(CollectRequest) returns (CollectResponse);
    rpc MergeResults(MergeRequest) returns (MergeResponse);
    rpc GenerateReport(ReportRequest) returns (Report);
}
```

**ポート**: 50053 (gRPC)

---

## ディレクトリ構造

```
aida/
├── cmd/
│   ├── gateway/
│   │   └── main.go
│   ├── orchestrator/
│   │   └── main.go
│   ├── worker/
│   │   └── main.go
│   ├── aggregator/
│   │   └── main.go
│   └── aida-cli/
│       └── main.go          # TUI クライアント
├── internal/
│   ├── gateway/
│   ├── orchestrator/
│   ├── worker/
│   ├── aggregator/
│   └── common/
│       ├── config/
│       └── logging/
├── pkg/
│   └── agent/               # Agent Interface (公開API)
├── proto/
│   ├── orchestrator.proto
│   ├── worker.proto
│   ├── aggregator.proto
│   └── common.proto
├── adapters/
│   ├── claudecode/
│   ├── geminicli/
│   └── codexcli/
├── deploy/
│   ├── podman/
│   │   ├── pod.yaml         # Podman pod定義
│   │   └── quadlet/         # systemd統合
│   └── docker/
│       └── docker-compose.yaml
├── Containerfile.gateway
├── Containerfile.orchestrator
├── Containerfile.worker
├── Containerfile.aggregator
└── Makefile
```

---

## Podman設定

### Pod定義 (deploy/podman/pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: aida-pod
  labels:
    app: aida
spec:
  containers:
    - name: gateway
      image: localhost/aida-gateway:latest
      ports:
        - containerPort: 8080
          hostPort: 8080
      env:
        - name: ORCHESTRATOR_ADDR
          value: "localhost:50051"

    - name: orchestrator
      image: localhost/aida-orchestrator:latest
      ports:
        - containerPort: 50051
      env:
        - name: REDIS_ADDR
          value: "localhost:6379"
        - name: WORKER_POOL_SIZE
          value: "3"

    - name: worker-1
      image: localhost/aida-worker:latest
      env:
        - name: WORKER_ID
          value: "worker-1"
        - name: ORCHESTRATOR_ADDR
          value: "localhost:50051"
      volumeMounts:
        - name: workdir
          mountPath: /workspace

    - name: worker-2
      image: localhost/aida-worker:latest
      env:
        - name: WORKER_ID
          value: "worker-2"
        - name: ORCHESTRATOR_ADDR
          value: "localhost:50051"
      volumeMounts:
        - name: workdir
          mountPath: /workspace

    - name: aggregator
      image: localhost/aida-aggregator:latest
      ports:
        - containerPort: 50053
      env:
        - name: ORCHESTRATOR_ADDR
          value: "localhost:50051"

    - name: redis
      image: docker.io/library/redis:7-alpine
      ports:
        - containerPort: 6379

  volumes:
    - name: workdir
      hostPath:
        path: /path/to/your/project
        type: Directory
```

### 起動コマンド

```bash
# イメージビルド
make build-images

# Pod起動
podman play kube deploy/podman/pod.yaml

# 状態確認
podman pod ps
podman ps --pod

# ログ確認
podman logs aida-pod-orchestrator

# 停止
podman pod stop aida-pod
podman pod rm aida-pod
```

---

## Makefile

```makefile
.PHONY: proto build-images run stop

# Protocol Buffers生成
proto:
	protoc --go_out=. --go-grpc_out=. proto/*.proto

# 全イメージビルド
build-images:
	podman build -t localhost/aida-gateway:latest -f Containerfile.gateway .
	podman build -t localhost/aida-orchestrator:latest -f Containerfile.orchestrator .
	podman build -t localhost/aida-worker:latest -f Containerfile.worker .
	podman build -t localhost/aida-aggregator:latest -f Containerfile.aggregator .

# 開発用: 個別サービス起動
run-gateway:
	go run cmd/gateway/main.go

run-orchestrator:
	go run cmd/orchestrator/main.go

run-worker:
	WORKER_ID=worker-1 go run cmd/worker/main.go

# Pod起動
run:
	podman play kube deploy/podman/pod.yaml

# Pod停止
stop:
	podman pod stop aida-pod
	podman pod rm aida-pod

# CLIビルド
build-cli:
	go build -o bin/aida cmd/aida-cli/main.go

# テスト
test:
	go test ./...

# gRPC接続テスト
test-grpc:
	grpcurl -plaintext localhost:50051 list
```

---

## Containerfile例

### Containerfile.orchestrator

```dockerfile
# Build stage
FROM docker.io/library/golang:1.22-alpine AS builder

WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 go build -o /orchestrator cmd/orchestrator/main.go

# Runtime stage
FROM docker.io/library/alpine:3.19

RUN apk --no-cache add ca-certificates git

COPY --from=builder /orchestrator /orchestrator

EXPOSE 50051

CMD ["/orchestrator"]
```

---

## 通信フロー

### タスク実行フロー

```
1. Client (CLI/UI)
   │
   │ REST POST /api/orchestrate
   ▼
2. Gateway
   │
   │ gRPC CreateTask()
   ▼
3. Orchestrator
   │ - タスク分割
   │ - 依存関係解析
   │ - Worker選択
   │
   │ gRPC AssignTask()
   ▼
4. Worker (複数並列)
   │ - Git worktree作成
   │ - エージェント呼び出し
   │ - 進捗報告 (stream)
   │
   │ Redis Pub/Sub
   ▼
5. Aggregator
   │ - 結果収集
   │ - コンフリクト検出
   │ - マージ実行
   │
   │ gRPC GenerateReport()
   ▼
6. Gateway → Client
   (レスポンス返却)
```

---

## ローカル開発

### 前提条件

```bash
# Go 1.22+
go version

# Podman
podman --version

# Protocol Buffers
protoc --version

# grpcurl (テスト用)
grpcurl --version
```

### クイックスタート

```bash
# 1. リポジトリクローン
git clone https://github.com/yourorg/aida.git
cd aida

# 2. 依存関係インストール
go mod download

# 3. protoファイル生成
make proto

# 4. イメージビルド
make build-images

# 5. Pod起動
make run

# 6. 状態確認
podman pod ps
curl http://localhost:8080/api/health

# 7. CLIで操作
./bin/aida status
./bin/aida task create "認証機能を実装"
```

---

## サービス間通信

### gRPC vs REST

| 項目 | gRPC | REST |
|------|------|------|
| プロトコル | HTTP/2 | HTTP/1.1 |
| データ形式 | Protocol Buffers | JSON |
| ストリーミング | 双方向対応 | 制限あり |
| 用途 | サービス間 | 外部API |

**方針**:
- **内部通信**: gRPC（高速、型安全）
- **外部API**: REST via Gateway（互換性）

### Redis Pub/Sub

```
Channels:
- aida:events:task     # タスク状態変更
- aida:events:worker   # Worker状態変更
- aida:events:log      # ログストリーム
```

---

## スケーリング

### Worker水平スケール

```bash
# Worker追加
podman run -d --pod aida-pod \
  -e WORKER_ID=worker-3 \
  -e ORCHESTRATOR_ADDR=localhost:50051 \
  localhost/aida-worker:latest
```

### リソース制限

```yaml
# pod.yaml に追加
resources:
  limits:
    memory: "512Mi"
    cpu: "500m"
  requests:
    memory: "256Mi"
    cpu: "250m"
```

---

## 次のステップ

1. **Phase 1**: proto定義 → gRPC scaffolding
2. **Phase 2**: Orchestrator + Worker 基本実装
3. **Phase 3**: Podman統合 + ローカル動作確認
4. **Phase 4**: TUI (bubbletea) クライアント
5. **Phase 5**: AINI連携
