# Redis Protocol Specification

## Overview

AIDA/AINI間のリアルタイム通信にRedisを使用します。

**役割分担**:
- **Markdown Protocol**: 永続的な状態保存（Git管理可能）
- **Redis Protocol**: リアルタイム通知・一時的な状態共有

```
┌─────────┐                    ┌─────────┐
│  AINI   │                    │  AIDA   │
│  (UI)   │                    │ (Engine)│
└────┬────┘                    └────┬────┘
     │                              │
     │    ┌──────────────────┐      │
     └───►│      Redis       │◄─────┘
          │                  │
          │  • Pub/Sub       │
          │  • State Cache   │
          │  • Command Queue │
          └────────┬─────────┘
                   │
                   ▼
          ┌──────────────────┐
          │   kanban.md      │  ← 永続ストレージ
          │   (Git管理)       │
          └──────────────────┘
```

---

## Redis Configuration

### Connection

```yaml
# ~/.aida/config.yaml
redis:
  host: localhost
  port: 6379
  password: ""           # 空の場合は認証なし
  db: 0
  prefix: "aida:"        # キープレフィックス

  # 接続プール
  pool:
    max_connections: 10
    min_idle: 2
    timeout: 5s
```

### Key Naming Convention

```
aida:{project}:{resource}:{id}

例:
aida:my-app:task:task-001
aida:my-app:worker:worker-1
aida:my-app:state:current
```

---

## Data Structures

### 1. Task State (Hash)

```redis
HSET aida:my-app:task:task-001
  id          "task-001"
  description "認証機能の実装"
  status      "in_progress"
  worker      "worker-1"
  agent       "claude-code"
  progress    "60"
  started_at  "2025-12-10T10:05:00Z"
  updated_at  "2025-12-10T10:50:00Z"
```

### 2. Worker State (Hash)

```redis
HSET aida:my-app:worker:worker-1
  id          "worker-1"
  agent       "claude-code"
  status      "running"
  task_id     "task-001"
  pid         "12345"
  started_at  "2025-12-10T10:05:00Z"
  heartbeat   "2025-12-10T10:50:30Z"
```

### 3. Task Queue (List)

```redis
# 待機タスクキュー
LPUSH aida:my-app:queue:pending "task-003"
LPUSH aida:my-app:queue:pending "task-004"

# 取り出し（Worker側）
BRPOP aida:my-app:queue:pending 0
```

### 4. Active Workers (Set)

```redis
SADD aida:my-app:workers:active "worker-1" "worker-2"
SMEMBERS aida:my-app:workers:active
```

### 5. Project State (Hash)

```redis
HSET aida:my-app:state:current
  status          "running"
  started_at      "2025-12-10T10:00:00Z"
  total_tasks     "6"
  completed_tasks "1"
  active_workers  "2"
```

---

## Pub/Sub Channels

### Channel Naming

```
aida:{project}:events:{type}

例:
aida:my-app:events:task
aida:my-app:events:worker
aida:my-app:events:log
aida:my-app:events:command
```

### Event Types

#### Task Events

```json
// Channel: aida:my-app:events:task
{
  "type": "task.created",
  "timestamp": "2025-12-10T10:00:00Z",
  "data": {
    "id": "task-001",
    "description": "認証機能の実装"
  }
}

{
  "type": "task.assigned",
  "timestamp": "2025-12-10T10:05:00Z",
  "data": {
    "id": "task-001",
    "worker": "worker-1",
    "agent": "claude-code"
  }
}

{
  "type": "task.progress",
  "timestamp": "2025-12-10T10:30:00Z",
  "data": {
    "id": "task-001",
    "progress": 40
  }
}

{
  "type": "task.completed",
  "timestamp": "2025-12-10T11:00:00Z",
  "data": {
    "id": "task-001",
    "duration": "55m"
  }
}

{
  "type": "task.failed",
  "timestamp": "2025-12-10T11:00:00Z",
  "data": {
    "id": "task-001",
    "error": "Timeout exceeded"
  }
}
```

#### Worker Events

```json
// Channel: aida:my-app:events:worker
{
  "type": "worker.spawned",
  "timestamp": "2025-12-10T10:05:00Z",
  "data": {
    "id": "worker-1",
    "agent": "claude-code",
    "pid": 12345
  }
}

{
  "type": "worker.status",
  "timestamp": "2025-12-10T10:05:30Z",
  "data": {
    "id": "worker-1",
    "status": "running",
    "task_id": "task-001"
  }
}

{
  "type": "worker.heartbeat",
  "timestamp": "2025-12-10T10:50:30Z",
  "data": {
    "id": "worker-1",
    "cpu": 45,
    "memory": 128
  }
}

{
  "type": "worker.stopped",
  "timestamp": "2025-12-10T11:00:00Z",
  "data": {
    "id": "worker-1",
    "reason": "task_completed"
  }
}
```

#### Log Events

```json
// Channel: aida:my-app:events:log
{
  "type": "log.entry",
  "timestamp": "2025-12-10T10:30:00Z",
  "data": {
    "worker": "worker-1",
    "level": "info",
    "message": "Created src/auth/login.go"
  }
}

{
  "type": "log.entry",
  "timestamp": "2025-12-10T10:45:00Z",
  "data": {
    "worker": "worker-1",
    "level": "warn",
    "message": "Dependency conflict detected"
  }
}
```

#### Command Events (AINI → AIDA)

```json
// Channel: aida:my-app:events:command
{
  "type": "command.start_orchestration",
  "timestamp": "2025-12-10T10:00:00Z",
  "data": {
    "config": {}
  }
}

{
  "type": "command.spawn_worker",
  "timestamp": "2025-12-10T10:05:00Z",
  "data": {
    "count": 3,
    "agent": "claude-code"
  }
}

{
  "type": "command.stop_worker",
  "timestamp": "2025-12-10T11:00:00Z",
  "data": {
    "worker_id": "worker-1"
  }
}

{
  "type": "command.assign_task",
  "timestamp": "2025-12-10T10:05:00Z",
  "data": {
    "task_id": "task-001",
    "worker_id": "worker-1"
  }
}
```

---

## Go Implementation

### Client

```go
// pkg/redis/client.go
package redis

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

type Client struct {
    rdb     *redis.Client
    prefix  string
    project string
}

type Config struct {
    Host     string
    Port     int
    Password string
    DB       int
    Prefix   string
}

func NewClient(cfg Config, project string) (*Client, error) {
    rdb := redis.NewClient(&redis.Options{
        Addr:     fmt.Sprintf("%s:%d", cfg.Host, cfg.Port),
        Password: cfg.Password,
        DB:       cfg.DB,
    })

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()

    if err := rdb.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("redis connection failed: %w", err)
    }

    return &Client{
        rdb:     rdb,
        prefix:  cfg.Prefix,
        project: project,
    }, nil
}

func (c *Client) key(parts ...string) string {
    return fmt.Sprintf("%s%s:%s", c.prefix, c.project,
        strings.Join(parts, ":"))
}

func (c *Client) Close() error {
    return c.rdb.Close()
}
```

### Task Operations

```go
// pkg/redis/task.go
package redis

import (
    "context"
    "encoding/json"
)

type Task struct {
    ID          string `redis:"id"`
    Description string `redis:"description"`
    Status      string `redis:"status"`
    Worker      string `redis:"worker"`
    Agent       string `redis:"agent"`
    Progress    int    `redis:"progress"`
    StartedAt   string `redis:"started_at"`
    UpdatedAt   string `redis:"updated_at"`
}

func (c *Client) SetTask(ctx context.Context, task *Task) error {
    key := c.key("task", task.ID)
    return c.rdb.HSet(ctx, key, task).Err()
}

func (c *Client) GetTask(ctx context.Context, id string) (*Task, error) {
    key := c.key("task", id)
    var task Task
    if err := c.rdb.HGetAll(ctx, key).Scan(&task); err != nil {
        return nil, err
    }
    return &task, nil
}

func (c *Client) UpdateTaskProgress(ctx context.Context, id string, progress int) error {
    key := c.key("task", id)
    return c.rdb.HSet(ctx, key,
        "progress", progress,
        "updated_at", time.Now().Format(time.RFC3339),
    ).Err()
}
```

### Pub/Sub

```go
// pkg/redis/pubsub.go
package redis

import (
    "context"
    "encoding/json"
)

type Event struct {
    Type      string         `json:"type"`
    Timestamp string         `json:"timestamp"`
    Data      map[string]any `json:"data"`
}

// Publish はイベントを発行する
func (c *Client) Publish(ctx context.Context, eventType string, event *Event) error {
    channel := c.key("events", eventType)
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }
    return c.rdb.Publish(ctx, channel, data).Err()
}

// Subscribe はイベントを購読する
func (c *Client) Subscribe(ctx context.Context, eventTypes ...string) (<-chan *Event, error) {
    channels := make([]string, len(eventTypes))
    for i, t := range eventTypes {
        channels[i] = c.key("events", t)
    }

    pubsub := c.rdb.Subscribe(ctx, channels...)
    ch := make(chan *Event, 100)

    go func() {
        defer close(ch)
        for msg := range pubsub.Channel() {
            var event Event
            if err := json.Unmarshal([]byte(msg.Payload), &event); err != nil {
                continue
            }
            select {
            case ch <- &event:
            case <-ctx.Done():
                return
            }
        }
    }()

    return ch, nil
}

// PublishTaskEvent はタスクイベントを発行する
func (c *Client) PublishTaskEvent(ctx context.Context, eventType string, taskID string, data map[string]any) error {
    if data == nil {
        data = make(map[string]any)
    }
    data["id"] = taskID

    return c.Publish(ctx, "task", &Event{
        Type:      eventType,
        Timestamp: time.Now().Format(time.RFC3339),
        Data:      data,
    })
}

// PublishWorkerEvent はワーカーイベントを発行する
func (c *Client) PublishWorkerEvent(ctx context.Context, eventType string, workerID string, data map[string]any) error {
    if data == nil {
        data = make(map[string]any)
    }
    data["id"] = workerID

    return c.Publish(ctx, "worker", &Event{
        Type:      eventType,
        Timestamp: time.Now().Format(time.RFC3339),
        Data:      data,
    })
}
```

---

## TypeScript Implementation (AINI)

### Client

```typescript
// lib/redis/client.ts
import { createClient, RedisClientType } from 'redis';

export interface RedisConfig {
  host: string;
  port: number;
  password?: string;
  db?: number;
  prefix?: string;
}

export class RedisClient {
  private client: RedisClientType;
  private subscriber: RedisClientType;
  private prefix: string;
  private project: string;

  constructor(config: RedisConfig, project: string) {
    const url = `redis://${config.host}:${config.port}/${config.db ?? 0}`;

    this.client = createClient({ url, password: config.password });
    this.subscriber = this.client.duplicate();
    this.prefix = config.prefix ?? 'aida:';
    this.project = project;
  }

  async connect(): Promise<void> {
    await this.client.connect();
    await this.subscriber.connect();
  }

  async disconnect(): Promise<void> {
    await this.client.quit();
    await this.subscriber.quit();
  }

  private key(...parts: string[]): string {
    return `${this.prefix}${this.project}:${parts.join(':')}`;
  }
}
```

### Subscription

```typescript
// lib/redis/subscription.ts
import type { RedisClient } from './client';

export interface Event {
  type: string;
  timestamp: string;
  data: Record<string, any>;
}

export type EventHandler = (event: Event) => void;

export class EventSubscriber {
  private handlers: Map<string, EventHandler[]> = new Map();

  constructor(private redis: RedisClient) {}

  async subscribe(eventTypes: string[]): Promise<void> {
    for (const type of eventTypes) {
      const channel = this.redis.key('events', type);

      await this.redis.subscriber.subscribe(channel, (message) => {
        const event: Event = JSON.parse(message);
        const handlers = this.handlers.get(type) ?? [];
        handlers.forEach(handler => handler(event));
      });
    }
  }

  on(eventType: string, handler: EventHandler): void {
    const handlers = this.handlers.get(eventType) ?? [];
    handlers.push(handler);
    this.handlers.set(eventType, handlers);
  }

  off(eventType: string, handler: EventHandler): void {
    const handlers = this.handlers.get(eventType) ?? [];
    const index = handlers.indexOf(handler);
    if (index !== -1) {
      handlers.splice(index, 1);
    }
  }
}
```

### Svelte Store Integration

```typescript
// stores/redis.ts
import { writable, derived } from 'svelte/store';
import { RedisClient } from '$lib/redis/client';
import { EventSubscriber } from '$lib/redis/subscription';

// Connection state
export const connected = writable(false);
export const error = writable<string | null>(null);

// Task state from Redis
export const tasks = writable<Map<string, Task>>(new Map());
export const workers = writable<Map<string, Worker>>(new Map());

let redis: RedisClient | null = null;
let subscriber: EventSubscriber | null = null;

export async function connect(config: RedisConfig, project: string) {
  try {
    redis = new RedisClient(config, project);
    await redis.connect();

    subscriber = new EventSubscriber(redis);
    await subscriber.subscribe(['task', 'worker', 'log']);

    // Event handlers
    subscriber.on('task', (event) => {
      tasks.update(map => {
        const task = map.get(event.data.id) ?? {};
        map.set(event.data.id, { ...task, ...event.data });
        return new Map(map);
      });
    });

    subscriber.on('worker', (event) => {
      workers.update(map => {
        const worker = map.get(event.data.id) ?? {};
        map.set(event.data.id, { ...worker, ...event.data });
        return new Map(map);
      });
    });

    connected.set(true);
  } catch (e) {
    error.set(e.message);
    connected.set(false);
  }
}

export async function disconnect() {
  if (redis) {
    await redis.disconnect();
    redis = null;
    subscriber = null;
    connected.set(false);
  }
}
```

---

## Heartbeat & Health Check

### Worker Heartbeat (AIDA側)

```go
func (w *Worker) startHeartbeat(ctx context.Context) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            w.redis.PublishWorkerEvent(ctx, "worker.heartbeat", w.ID, map[string]any{
                "cpu":    getCPUUsage(),
                "memory": getMemoryUsage(),
            })

            // Heartbeat timestamp も更新
            w.redis.rdb.HSet(ctx, w.redis.key("worker", w.ID),
                "heartbeat", time.Now().Format(time.RFC3339))
        }
    }
}
```

### Health Check (AINI側)

```typescript
// Workerが30秒以上heartbeatを送ってこなければdead判定
const HEARTBEAT_TIMEOUT = 30 * 1000;

function checkWorkerHealth(worker: Worker): 'healthy' | 'warning' | 'dead' {
  const lastHeartbeat = new Date(worker.heartbeat).getTime();
  const now = Date.now();
  const diff = now - lastHeartbeat;

  if (diff < 15000) return 'healthy';
  if (diff < HEARTBEAT_TIMEOUT) return 'warning';
  return 'dead';
}
```

---

## Fallback Strategy

Redisが利用できない場合のフォールバック:

```go
// pkg/redis/client.go

func (c *Client) PublishWithFallback(ctx context.Context, event *Event) error {
    err := c.Publish(ctx, event.Type, event)
    if err != nil {
        // Redis失敗時はMarkdownに直接書き込み
        log.Warn("Redis publish failed, falling back to markdown", "error", err)
        return c.writeToMarkdown(event)
    }
    return nil
}
```

```
Redis可用時:
  AIDA ──Redis──► AINI  (リアルタイム)
         │
         └──► kanban.md  (永続化)

Redis不可用時:
  AIDA ──► kanban.md ◄── AINI  (ファイル監視)
```

---

## Docker Compose Example

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes

volumes:
  redis_data:
```

---

## Security Considerations

1. **認証**: 本番環境では `requirepass` を設定
2. **TLS**: 必要に応じて TLS 接続を使用
3. **ネットワーク**: Redis を外部に公開しない
4. **キー有効期限**: 一時データには TTL を設定

```go
// 一時データには TTL を設定
func (c *Client) SetTaskWithTTL(ctx context.Context, task *Task, ttl time.Duration) error {
    key := c.key("task", task.ID)
    pipe := c.rdb.Pipeline()
    pipe.HSet(ctx, key, task)
    pipe.Expire(ctx, key, ttl)
    _, err := pipe.Exec(ctx)
    return err
}
```
