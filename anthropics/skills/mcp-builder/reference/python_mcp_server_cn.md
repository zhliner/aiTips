# Python MCP Server 实现指南

## 概述

本文档提供使用 MCP Python SDK 实现 MCP server 的 Python 特定最佳实践和示例。涵盖 server 设置、tool 注册模式、使用 Pydantic 进行输入验证、错误处理以及完整的可运行示例。

---

## 快速参考

### 关键导入
```python
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field, field_validator, ConfigDict
from typing import Optional, List, Dict, Any
from enum import Enum
import httpx
```

### Server 初始化
```python
mcp = FastMCP("service_mcp")
```

### Tool 注册模式
```python
@mcp.tool(name="tool_name", annotations={...})
async def tool_function(params: InputModel) -> str:
    # 实现
    pass
```

---

## MCP Python SDK 与 FastMCP

官方 MCP Python SDK 提供 FastMCP，一个用于构建 MCP server 的高层框架。它提供：
- 从函数签名和 docstring 自动生成 description 和 inputSchema
- Pydantic 模型集成用于输入验证
- 基于装饰器的 tool 注册，使用 `@mcp.tool`

**完整 SDK 文档，请使用 WebFetch 加载：**
`https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`

## Server 命名规范

Python MCP server 必须遵循以下命名模式：
- **格式**：`{service}_mcp`（小写加下划线）
- **示例**：`github_mcp`、`jira_mcp`、`stripe_mcp`

名称应当：
- 通用（不与特定功能绑定）
- 描述所集成的服务/API
- 易于从任务描述中推断
- 不包含版本号或日期

## Tool 实现

### Tool 命名

使用 snake_case 作为 tool 名称（例如 "search_users"、"create_project"、"get_channel_info"），名称应清晰且面向动作。

**避免命名冲突**：包含 service 上下文以防止重叠：
- 使用 "slack_send_message" 而非 "send_message"
- 使用 "github_create_issue" 而非 "create_issue"
- 使用 "asana_list_tasks" 而非 "list_tasks"

### 使用 FastMCP 的 Tool 结构

Tool 使用 `@mcp.tool` 装饰器定义，配合 Pydantic 模型进行输入验证：

```python
from pydantic import BaseModel, Field, ConfigDict
from mcp.server.fastmcp import FastMCP

# 初始化 MCP server
mcp = FastMCP("example_mcp")

# 定义用于输入验证的 Pydantic 模型
class ServiceToolInput(BaseModel):
    '''Service tool 操作的输入模型。'''
    model_config = ConfigDict(
        str_strip_whitespace=True,  # 自动去除字符串两端空白
        validate_assignment=True,    # 赋值时验证
        extra='forbid'              # 禁止额外字段
    )

    param1: str = Field(..., description="First parameter description (e.g., 'user123', 'project-abc')", min_length=1, max_length=100)
    param2: Optional[int] = Field(default=None, description="Optional integer parameter with constraints", ge=0, le=1000)
    tags: Optional[List[str]] = Field(default_factory=list, description="List of tags to apply", max_items=10)

@mcp.tool(
    name="service_tool_name",
    annotations={
        "title": "Human-Readable Tool Title",
        "readOnlyHint": True,     # Tool 不修改环境
        "destructiveHint": False,  # Tool 不执行破坏性操作
        "idempotentHint": True,    # 重复调用无额外效果
        "openWorldHint": False     # Tool 不与外部实体交互
    }
)
async def service_tool_name(params: ServiceToolInput) -> str:
    '''Tool 描述自动成为 'description' 字段。

    本 tool 对 service 执行特定操作。在处理之前，
    使用 ServiceToolInput Pydantic 模型验证所有输入。

    Args:
        params (ServiceToolInput): 验证后的输入参数，包含：
            - param1 (str): 第一个参数描述
            - param2 (Optional[int]): 带默认值的可选参数
            - tags (Optional[List[str]]): 标签列表

    Returns:
        str: JSON 格式的响应，包含操作结果
    '''
    # 在此实现
    pass
```

## Pydantic v2 关键特性

- 使用 `model_config` 代替嵌套的 `Config` 类
- 使用 `field_validator` 代替已弃用的 `validator`
- 使用 `model_dump()` 代替已弃用的 `dict()`
- 验证器需要 `@classmethod` 装饰器
- 验证器方法需要类型提示

```python
from pydantic import BaseModel, Field, field_validator, ConfigDict

class CreateUserInput(BaseModel):
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True
    )

    name: str = Field(..., description="User's full name", min_length=1, max_length=100)
    email: str = Field(..., description="User's email address", pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$')
    age: int = Field(..., description="User's age", ge=0, le=150)

    @field_validator('email')
    @classmethod
    def validate_email(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Email cannot be empty")
        return v.lower()
```

## 响应格式选项

支持多种输出格式以提高灵活性：

```python
from enum import Enum

class ResponseFormat(str, Enum):
    '''Tool 响应的输出格式。'''
    MARKDOWN = "markdown"
    JSON = "json"

class UserSearchInput(BaseModel):
    query: str = Field(..., description="Search query")
    response_format: ResponseFormat = Field(
        default=ResponseFormat.MARKDOWN,
        description="Output format: 'markdown' for human-readable or 'json' for machine-readable"
    )
```

**Markdown 格式**：
- 使用标题、列表和格式以提高清晰度
- 将时间戳转换为人类可读格式（例如 "2024-01-15 10:30:00 UTC" 而非 epoch）
- 显示名称并在括号中附带 ID（例如 "@john.doe (U123456)"）
- 省略冗长的元数据（例如只显示一个头像 URL，而非所有尺寸）
- 逻辑分组相关信息

**JSON 格式**：
- 返回完整的、适合程序化处理的结构化数据
- 包含所有可用字段和元数据
- 使用一致的字段名和类型

## 分页实现

对于列出资源的 tool：

```python
class ListInput(BaseModel):
    limit: Optional[int] = Field(default=20, description="Maximum results to return", ge=1, le=100)
    offset: Optional[int] = Field(default=0, description="Number of results to skip for pagination", ge=0)

async def list_items(params: ListInput) -> str:
    # 使用分页参数发起 API 请求
    data = await api_request(limit=params.limit, offset=params.offset)

    # 返回分页信息
    response = {
        "total": data["total"],
        "count": len(data["items"]),
        "offset": params.offset,
        "items": data["items"],
        "has_more": data["total"] > params.offset + len(data["items"]),
        "next_offset": params.offset + len(data["items"]) if data["total"] > params.offset + len(data["items"]) else None
    }
    return json.dumps(response, indent=2)
```

## 错误处理

提供清晰、可操作的错误消息：

```python
def _handle_api_error(e: Exception) -> str:
    '''所有 tool 的一致错误格式化。'''
    if isinstance(e, httpx.HTTPStatusError):
        if e.response.status_code == 404:
            return "Error: Resource not found. Please check the ID is correct."
        elif e.response.status_code == 403:
            return "Error: Permission denied. You don't have access to this resource."
        elif e.response.status_code == 429:
            return "Error: Rate limit exceeded. Please wait before making more requests."
        return f"Error: API request failed with status {e.response.status_code}"
    elif isinstance(e, httpx.TimeoutException):
        return "Error: Request timed out. Please try again."
    return f"Error: Unexpected error occurred: {type(e).__name__}"
```

## 共享工具

将通用功能提取为可复用的函数：

```python
# 共享 API 请求函数
async def _make_api_request(endpoint: str, method: str = "GET", **kwargs) -> dict:
    '''所有 API 调用的可复用函数。'''
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method,
            f"{API_BASE_URL}/{endpoint}",
            timeout=30.0,
            **kwargs
        )
        response.raise_for_status()
        return response.json()
```

## Async/Await 最佳实践

始终使用 async/await 处理网络请求和 I/O 操作：

```python
# 好的做法：异步网络请求
async def fetch_data(resource_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(f"{API_URL}/resource/{resource_id}")
        response.raise_for_status()
        return response.json()

# 不好的做法：同步请求
def fetch_data(resource_id: str) -> dict:
    response = requests.get(f"{API_URL}/resource/{resource_id}")  # 阻塞
    return response.json()
```

## 类型提示

全程使用类型提示：

```python
from typing import Optional, List, Dict, Any

async def get_user(user_id: str) -> Dict[str, Any]:
    data = await fetch_user(user_id)
    return {"id": data["id"], "name": data["name"]}
```

## Tool Docstring

每个 tool 必须有包含显式类型信息的全面 docstring：

```python
async def search_users(params: UserSearchInput) -> str:
    '''
    在 Example 系统中按名称、电子邮件或团队搜索用户。

    本 tool 在 Example 平台的所有用户资料中搜索，
    支持部分匹配和各种搜索过滤器。它不会
    创建或修改用户，仅搜索现有用户。

    Args:
        params (UserSearchInput): 验证后的输入参数，包含：
            - query (str): 用于匹配名称/电子邮件的搜索字符串（例如 "john"、"@example.com"、"team:marketing"）
            - limit (Optional[int]): 返回的最大结果数，范围 1-100（默认：20）
            - offset (Optional[int]): 分页跳过的结果数（默认：0）

    Returns:
        str: JSON 格式的字符串，包含搜索结果，schema 如下：

        成功响应：
        {
            "total": int,           # 找到的匹配总数
            "count": int,           # 本次响应中的结果数
            "offset": int,          # 当前分页偏移量
            "users": [
                {
                    "id": str,      # 用户 ID（例如 "U123456789"）
                    "name": str,    # 全名（例如 "John Doe"）
                    "email": str,   # 电子邮件地址（例如 "john@example.com"）
                    "team": str     # 团队名称（例如 "Marketing"）- 可选
                }
            ]
        }

        错误响应：
        "Error: <error message>" 或 "No users found matching '<query>'"

    Examples:
        - Use when: "Find all marketing team members" -> params with query="team:marketing"
        - Use when: "Search for John's account" -> params with query="john"
        - Don't use when: You need to create a user (use example_create_user instead)
        - Don't use when: You have a user ID and need full details (use example_get_user instead)

    Error Handling:
        - 输入验证错误由 Pydantic 模型处理
        - 请求过多时返回 "Error: Rate limit exceeded"（429 状态码）
        - API key 无效时返回 "Error: Invalid API authentication"（401 状态码）
        - 返回格式化的结果列表或 "No users found matching 'query'"
    '''
```

## 完整示例

以下是一个完整的 Python MCP server 示例：

```python
#!/usr/bin/env python3
'''
Example Service 的 MCP Server。

本 server 提供与 Example API 交互的 tool，包括用户搜索、
项目管理和数据导出功能。
'''

from typing import Optional, List, Dict, Any
from enum import Enum
import httpx
from pydantic import BaseModel, Field, field_validator, ConfigDict
from mcp.server.fastmcp import FastMCP

# 初始化 MCP server
mcp = FastMCP("example_mcp")

# 常量
API_BASE_URL = "https://api.example.com/v1"

# 枚举
class ResponseFormat(str, Enum):
    '''Tool 响应的输出格式。'''
    MARKDOWN = "markdown"
    JSON = "json"

# 用于输入验证的 Pydantic 模型
class UserSearchInput(BaseModel):
    '''用户搜索操作的输入模型。'''
    model_config = ConfigDict(
        str_strip_whitespace=True,
        validate_assignment=True
    )

    query: str = Field(..., description="Search string to match against names/emails", min_length=2, max_length=200)
    limit: Optional[int] = Field(default=20, description="Maximum results to return", ge=1, le=100)
    offset: Optional[int] = Field(default=0, description="Number of results to skip for pagination", ge=0)
    response_format: ResponseFormat = Field(default=ResponseFormat.MARKDOWN, description="Output format")

    @field_validator('query')
    @classmethod
    def validate_query(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Query cannot be empty or whitespace only")
        return v.strip()

# 共享工具函数
async def _make_api_request(endpoint: str, method: str = "GET", **kwargs) -> dict:
    '''所有 API 调用的可复用函数。'''
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method,
            f"{API_BASE_URL}/{endpoint}",
            timeout=30.0,
            **kwargs
        )
        response.raise_for_status()
        return response.json()

def _handle_api_error(e: Exception) -> str:
    '''所有 tool 的一致错误格式化。'''
    if isinstance(e, httpx.HTTPStatusError):
        if e.response.status_code == 404:
            return "Error: Resource not found. Please check the ID is correct."
        elif e.response.status_code == 403:
            return "Error: Permission denied. You don't have access to this resource."
        elif e.response.status_code == 429:
            return "Error: Rate limit exceeded. Please wait before making more requests."
        return f"Error: API request failed with status {e.response.status_code}"
    elif isinstance(e, httpx.TimeoutException):
        return "Error: Request timed out. Please try again."
    return f"Error: Unexpected error occurred: {type(e).__name__}"

# Tool 定义
@mcp.tool(
    name="example_search_users",
    annotations={
        "title": "Search Example Users",
        "readOnlyHint": True,
        "destructiveHint": False,
        "idempotentHint": True,
        "openWorldHint": True
    }
)
async def example_search_users(params: UserSearchInput) -> str:
    '''在 Example 系统中按名称、电子邮件或团队搜索用户。

    [完整 docstring 如上方所示]
    '''
    try:
        # 使用验证后的参数发起 API 请求
        data = await _make_api_request(
            "users/search",
            params={
                "q": params.query,
                "limit": params.limit,
                "offset": params.offset
            }
        )

        users = data.get("users", [])
        total = data.get("total", 0)

        if not users:
            return f"No users found matching '{params.query}'"

        # 根据请求格式格式化响应
        if params.response_format == ResponseFormat.MARKDOWN:
            lines = [f"# User Search Results: '{params.query}'", ""]
            lines.append(f"Found {total} users (showing {len(users)})")
            lines.append("")

            for user in users:
                lines.append(f"## {user['name']} ({user['id']})")
                lines.append(f"- **Email**: {user['email']}")
                if user.get('team'):
                    lines.append(f"- **Team**: {user['team']}")
                lines.append("")

            return "\n".join(lines)

        else:
            # 机器可读的 JSON 格式
            import json
            response = {
                "total": total,
                "count": len(users),
                "offset": params.offset,
                "users": users
            }
            return json.dumps(response, indent=2)

    except Exception as e:
        return _handle_api_error(e)

if __name__ == "__main__":
    mcp.run()
```

---

## 高级 FastMCP 功能

### Context 参数注入

FastMCP 可以自动向 tool 注入 `Context` 参数，用于日志记录、进度报告、resource 读取和用户交互等高级功能：

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("example_mcp")

@mcp.tool()
async def advanced_search(query: str, ctx: Context) -> str:
    '''具有上下文访问能力的高级 tool，用于日志记录和进度报告。'''

    # 为长时间操作报告进度
    await ctx.report_progress(0.25, "Starting search...")

    # 记录调试信息
    await ctx.log_info("Processing query", {"query": query, "timestamp": datetime.now()})

    # 执行搜索
    results = await search_api(query)
    await ctx.report_progress(0.75, "Formatting results...")

    # 访问 server 配置
    server_name = ctx.fastmcp.name

    return format_results(results)

@mcp.tool()
async def interactive_tool(resource_id: str, ctx: Context) -> str:
    '''可以向用户请求额外输入的 tool。'''

    # 在需要时请求敏感信息
    api_key = await ctx.elicit(
        prompt="Please provide your API key:",
        input_type="password"
    )

    # 使用提供的 key
    return await api_call(resource_id, api_key)
```

**Context 能力：**
- `ctx.report_progress(progress, message)` - 为长时间操作报告进度
- `ctx.log_info(message, data)` / `ctx.log_error()` / `ctx.log_debug()` - 日志记录
- `ctx.elicit(prompt, input_type)` - 向用户请求输入
- `ctx.fastmcp.name` - 访问 server 配置
- `ctx.read_resource(uri)` - 读取 MCP resource

### Resource 注册

将数据作为 resource 暴露，实现高效的、基于模板的访问：

```python
@mcp.resource("file://documents/{name}")
async def get_document(name: str) -> str:
    '''将文档作为 MCP resource 暴露。

    Resource 适用于不需要复杂参数的静态或半静态数据。
    它们使用 URI 模板实现灵活访问。
    '''
    document_path = f"./docs/{name}"
    with open(document_path, "r") as f:
        return f.read()

@mcp.resource("config://settings/{key}")
async def get_setting(key: str, ctx: Context) -> str:
    '''将配置作为带上下文的 resource 暴露。'''
    settings = await load_settings()
    return json.dumps(settings.get(key, {}))
```

**何时使用 Resource vs Tool：**
- **Resource**：用于带简单参数（URI 模板）的数据访问
- **Tool**：用于带验证和业务逻辑的复杂操作

### 结构化输出类型

FastMCP 支持字符串之外的多种返回类型：

```python
from typing import TypedDict
from dataclasses import dataclass
from pydantic import BaseModel

# 用于结构化返回的 TypedDict
class UserData(TypedDict):
    id: str
    name: str
    email: str

@mcp.tool()
async def get_user_typed(user_id: str) -> UserData:
    '''返回结构化数据 - FastMCP 处理序列化。'''
    return {"id": user_id, "name": "John Doe", "email": "john@example.com"}

# 用于复杂验证的 Pydantic 模型
class DetailedUser(BaseModel):
    id: str
    name: str
    email: str
    created_at: datetime
    metadata: Dict[str, Any]

@mcp.tool()
async def get_user_detailed(user_id: str) -> DetailedUser:
    '''返回 Pydantic 模型 - 自动生成 schema。'''
    user = await fetch_user(user_id)
    return DetailedUser(**user)
```

### 生命周期管理

初始化跨请求持久化的资源：

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def app_lifespan():
    '''管理 server 生命周期内的资源。'''
    # 初始化连接、加载配置等
    db = await connect_to_database()
    config = load_configuration()

    # 使所有 tool 可用
    yield {"db": db, "config": config}

    # 关闭时清理
    await db.close()

mcp = FastMCP("example_mcp", lifespan=app_lifespan)

@mcp.tool()
async def query_data(query: str, ctx: Context) -> str:
    '''通过上下文访问生命周期资源。'''
    db = ctx.request_context.lifespan_state["db"]
    results = await db.query(query)
    return format_results(results)
```

### 传输选项

FastMCP 支持两种主要传输机制：

```python
# stdio 传输（用于本地 tool）- 默认
if __name__ == "__main__":
    mcp.run()

# Streamable HTTP 传输（用于远程 server）
if __name__ == "__main__":
    mcp.run(transport="streamable_http", port=8000)
```

**传输方式选择：**
- **stdio**：命令行工具、本地集成、子进程执行
- **Streamable HTTP**：web 服务、远程访问、多 client

---

## 代码最佳实践

### 代码可组合性与可复用性

你的实现必须优先考虑可组合性和代码复用：

1. **提取通用功能**：
   - 为多个 tool 使用的操作创建可复用的辅助函数
   - 构建共享的 API client 用于 HTTP 请求，而非重复代码
   - 将错误处理逻辑集中在工具函数中
   - 将业务逻辑提取为可组合的专用函数
   - 提取共享的 markdown 或 JSON 字段选择和格式化功能

2. **避免重复**：
   - 永远不要在 tool 之间复制粘贴相似代码
   - 如果发现自己在写相似的逻辑两次，将其提取为函数
   - 分页、过滤、字段选择和格式化等通用操作应当共享
   - 身份验证/授权逻辑应当集中

### Python 特定最佳实践

1. **使用类型提示**：始终为函数参数和返回值添加类型注解
2. **Pydantic 模型**：为所有输入验证定义清晰的 Pydantic 模型
3. **避免手动验证**：让 Pydantic 通过约束处理输入验证
4. **合理的导入组织**：分组导入（标准库、第三方库、本地模块）
5. **错误处理**：使用特定的异常类型（httpx.HTTPStatusError，而非通用 Exception）
6. **异步上下文管理器**：对需要清理的资源使用 `async with`
7. **常量**：在模块级别以大写形式定义常量

## 质量检查清单

在完成 Python MCP server 实现之前，确保：

### 战略设计
- [ ] Tool 实现完整的工作流，而不仅仅是 API endpoint 的封装
- [ ] Tool 名称反映自然的任务划分
- [ ] 响应格式针对 agent 上下文效率进行优化
- [ ] 在适当的地方使用人类可读的标识符
- [ ] 错误消息引导 agent 正确使用

### 实现质量
- [ ] 聚焦实现：已实现最重要和最有价值的 tool
- [ ] 所有 tool 具有描述性名称和文档
- [ ] 相似操作的返回类型一致
- [ ] 所有外部调用实现了错误处理
- [ ] Server 名称遵循格式：`{service}_mcp`
- [ ] 所有网络操作使用 async/await
- [ ] 通用功能已提取为可复用函数
- [ ] 错误消息清晰、可操作且具有教育意义
- [ ] 输出经过适当验证和格式化

### Tool 配置
- [ ] 所有 tool 在装饰器中实现了 'name' 和 'annotations'
- [ ] 注解正确设置（readOnlyHint、destructiveHint、idempotentHint、openWorldHint）
- [ ] 所有 tool 使用 Pydantic BaseModel 进行输入验证并带有 Field() 定义
- [ ] 所有 Pydantic Field 具有显式类型和带约束的描述
- [ ] 所有 tool 具有包含显式输入/输出类型的全面 docstring
- [ ] Docstring 包含 dict/JSON 返回的完整 schema 结构
- [ ] Pydantic 模型处理输入验证（无需手动验证）

### 高级功能（如适用）
- [ ] 使用 Context 注入进行日志记录、进度报告或信息收集
- [ ] 为适当的数据 endpoint 注册了 resource
- [ ] 实现了生命周期管理用于持久连接
- [ ] 使用了结构化输出类型（TypedDict、Pydantic 模型）
- [ ] 配置了适当的传输方式（stdio 或 streamable HTTP）

### 代码质量
- [ ] 文件包含正确的导入，包括 Pydantic 导入
- [ ] 在适用场景下正确实现了分页
- [ ] 为可能的大型结果集提供了过滤选项
- [ ] 所有 async 函数使用 `async def` 正确定义
- [ ] HTTP client 使用遵循异步模式并带有正确的上下文管理器
- [ ] 全程使用了类型提示
- [ ] 常量在模块级别以大写形式定义

### 测试
- [ ] Server 可成功运行：`python your_server.py --help`
- [ ] 所有导入正确解析
- [ ] 示例 tool 调用按预期工作
- [ ] 错误场景得到优雅处理
