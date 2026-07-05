# Managed Agents — Memory Store

> **公测阶段。** Memory Store 在 `managed-agents-2026-04-01` beta header 下提供；SDK 会在所有 `client.beta.memory_stores.*` 调用中自动设置该 header。如果 `client.beta.memory_stores` 不存在，请升级到最新的 SDK 版本。

会话默认是临时的——会话结束后，Agent 学到的所有内容都会丢失。**Memory Store** 是一个工作区级别的小型文本文档集合，可跨会话持久化。当 Store 被附加到会话（通过 `resources[]`）时，它会作为文件系统目录挂载到容器中；Agent 使用普通的文件工具对其进行读写，系统提示中会自动注入一条说明，告知 Agent 该挂载的存在。

每次对 Memory 的变更都会生成一个不可变的 **Memory 版本**（`memver_...`），为你提供审计追踪和时间点回滚/编辑能力。

## 对象模型

| 对象 | ID 前缀 | 作用域 | 说明 |
| --- | --- | --- | --- |
| Memory store | `memstore_...` | 工作区 | 通过 `resources[]` 附加到会话 |
| Memory | `mem_...` | Store | 一个文本文件，通过 `path` 寻址（每个 ≤ 100KB——建议使用多个小文件） |
| Memory version | `memver_...` | Memory | 每次变更的不可变快照；`operation` ∈ `created` / `modified` / `deleted` |

## 创建 Store

`description` 会传递给 Agent，使其了解 Store 的内容——请为模型编写，而非为人类。

```python
store = client.beta.memory_stores.create(
    name="User Preferences",
    description="Per-user preferences and project context.",
)
print(store.id)  # memstore_01Hx...
```

其他 SDK：TypeScript `client.beta.memoryStores.create({...})`；Go `client.Beta.MemoryStores.New(ctx, ...)`。完整的各语言对照表请参阅 `shared/managed-agents-api-reference.md` → SDK 方法参考。

Store 支持 `retrieve` / `update` / `list`（支持 `include_archived`、`created_at_{gte,lte}` 过滤器）/ `delete` / **`archive`**。归档使 Store 变为只读——已有的会话附加继续有效，新会话无法引用它；不支持取消归档。

### 预填充内容（可选）

在任何会话运行之前预加载参考资料。`memories.create` 在指定 `path` 处创建 Memory；如果该路径已存在 Memory，调用返回 `409`（`memory_path_conflict_error`，包含 `conflicting_memory_id`）。Store ID 是第一个位置参数。

```python
client.beta.memory_stores.memories.create(
    store.id,
    path="/formatting_standards.md",
    content="All reports use GAAP formatting. Dates are ISO-8601...",
)
```

## 附加到会话

Memory Store 放在会话的 `resources[]` 数组中，与 `file` 和 `github_repository` 资源并列（参见 `shared/managed-agents-environments.md` → Resources）。Memory Store 仅在**创建会话时附加**——`sessions.resources.add()` 不接受 `memory_store`。

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[
        {
            "type": "memory_store",
            "memory_store_id": store.id,
            "access": "read_write",  # 或 "read_only"；默认为 "read_write"
            "instructions": "User preferences and project context. Check before starting any task.",
        }
    ],
)
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `type` | ✅ | `"memory_store"` |
| `memory_store_id` | ✅ | `memstore_...` |
| `access` | — | `"read_write"`（默认）或 `"read_only"`——在挂载的文件系统级别强制执行 |
| `instructions` | — | 针对该 Store 的会话级指导，补充 Store 的 `name`/`description`。≤ 4,096 字符。 |

**每个会话最多 8 个 Memory Store。** 当不同的 Memory 切片有不同的所有者或生命周期时，可附加多个——例如一个只读的共享参考 Store 加一个每用户读写的 Store，或者每个终端用户/团队/项目各一个 Store，共享同一个 Agent 配置。

### Agent 如何访问（FUSE 挂载）

每个附加的 Store 在会话容器中挂载到 `/mnt/memory/<store-name>/`。Agent 使用标准文件工具（`bash`、`read`、`write`、`edit`、`glob`、`grep`）与其交互——没有专用的 Memory 工具。`access: "read_only"` 在文件系统级别使挂载只读；`"read_write"` 允许 Agent 在其下创建、编辑和删除文件。每个挂载的简短描述（名称、路径、`instructions`、访问权限）会自动注入系统提示，使 Agent 知道 Store 的存在，无需你手动提及。

Agent 在挂载下进行的写入会持久化回 Store，并生成 Memory 版本，与主机端的 `memories.update` 调用效果一致。

## 直接管理 Memory（主机端）

使用以下操作进行审核工作流、修正错误的 Memory，或在带外方式中预填充 Store。

### 列表

返回 `Memory | MemoryPrefix` 条目——`MemoryPrefix`（`type: "memory_prefix"`，仅含 `path`）是层级列表中的类目录节点。使用 `path_prefix` 限定范围（包含尾部斜杠：`"/notes/"` 匹配 `/notes/a.md` 但不匹配 `/notes_backup/old.md`），使用 `depth` 限制树遍历深度。`order_by` / `order` 对结果排序。传入 `view="full"` 以在每个条目中包含 `content`；默认 `"basic"` 仅返回元数据。

```python
for m in client.beta.memory_stores.memories.list(store.id, path_prefix="/"):
    if m.type == "memory":
        print(f"{m.path}  ({m.content_size_bytes} bytes, sha={m.content_sha256[:8]})")
    else:  # "memory_prefix"
        print(f"{m.path}/")
```

### 读取

```python
mem = client.beta.memory_stores.memories.retrieve(memory_id, memory_store_id=store.id)
print(mem.content)
```

`retrieve` 默认为 `view="full"`（包含内容）；`view` 主要在列表端点上起作用。

### 创建 vs. 更新

| 操作 | 寻址方式 | 语义 |
| --- | --- | --- |
| `memories.create(store_id, path=..., content=...)` | **路径** | 在 `path` 处创建。如果路径已被占用，返回 `409`（`memory_path_conflict_error`，包含 `conflicting_memory_id`）。 |
| `memories.update(mem_id, memory_store_id=..., path=..., content=...)` | **`mem_...` ID** | 修改现有 Memory。可更改 `content`、`path`（重命名）或两者。重命名到已占用的路径返回同样的 `409 memory_path_conflict_error`。 |

```python
mem = client.beta.memory_stores.memories.create(
    store.id,
    path="/preferences/formatting.md",
    content="Always use tabs, not spaces.",
)

client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    path="/archive/2026_q1_formatting.md",  # 重命名
)
```

### 乐观并发控制（`update` 的前置条件）

`memories.update` 接受 `precondition`，使你可以执行 读取 → 修改 → 写回 而不会覆盖并发写入者的操作。唯一支持的类型是 `content_sha256`。不匹配时 API 返回 `409`（`memory_precondition_failed_error`）——重新读取并以最新状态重试。

```python
client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    content="CORRECTED: Always use 2-space indentation.",
    precondition={"type": "content_sha256", "content_sha256": mem.content_sha256},
)
```

### 删除

```python
client.beta.memory_stores.memories.delete(mem.id, memory_store_id=store.id)
```

传入 `expected_content_sha256` 实现条件删除。

## 审计与回滚——Memory 版本

每次变更都会创建一个不可变的 `memver_...` 快照。版本在父 Memory 的生命周期内累积；`memories.retrieve` 始终返回当前最新版本，版本端点提供历史记录。

| 触发操作 | 版本上的 `operation` 字段 |
| --- | --- |
| `memories.create` 在新路径 | `"created"` |
| `memories.update` 更改 `content`、`path` 或两者（或 Agent 端对挂载的写入） | `"modified"` |
| `memories.delete` | `"deleted"` |

每个版本还记录 `created_by`——一个 actor 对象，`type` ∈ `session_actor` / `api_actor` / `user_actor`——以及编辑后的 `redacted_at` + `redacted_by`。

### 列出版本

按时间倒序，分页返回。可按 `memory_id`、`operation`、`session_id`、`api_key_id` 或 `created_at_gte` / `created_at_lte` 过滤。传入 `view="full"` 以包含 `content`；默认仅返回元数据。

```python
for v in client.beta.memory_stores.memory_versions.list(store.id, memory_id=mem.id):
    print(f"{v.id}: {v.operation}")
```

### 检索版本

```python
version = client.beta.memory_stores.memory_versions.retrieve(
    version_id, memory_store_id=store.id
)
print(version.content)
```

### 编辑版本

从历史版本中清除内容，同时保留审计追踪（操作者 + 时间戳）。清除 `content`、`content_sha256`、`content_size_bytes` 和 `path`；其余内容保留。用于泄露的密钥、PII 或用户删除请求。

```python
client.beta.memory_stores.memory_versions.redact(version_id, memory_store_id=store.id)
```

## 端点参考

完整的 HTTP 方法/路径表请参阅 `shared/managed-agents-api-reference.md` → Memory Stores / Memories / Memory Versions。原始 HTTP 基础路径：

```
POST   /v1/memory_stores
POST   /v1/memory_stores/{memory_store_id}/archive
GET    /v1/memory_stores/{memory_store_id}/memories
PATCH  /v1/memory_stores/{memory_store_id}/memories/{memory_id}
GET    /v1/memory_stores/{memory_store_id}/memory_versions
POST   /v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact
```

cURL 示例和 CLI（`ant beta:memory-stores ...`）请参阅 `shared/live-sources.md` → Managed Agents 中的 Memory URL。
