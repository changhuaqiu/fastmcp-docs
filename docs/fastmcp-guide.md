# FastMCP 完整开发指南

> 基于 MCP Python SDK v1.x 的官方 FastMCP 框架

---

## 目录

1. [简介](#1-简介)
2. [快速开始](#2-快速开始)
3. [核心概念](#3-核心概念)
4. [工具开发](#4-工具开发)
5. [资源开发](#5-资源开发)
6. [提示词开发](#6-提示词开发)
7. [上下文使用](#7-上下文使用)
8. [结构化输出](#8-结构化输出)
9. [服务器运行](#9-服务器运行)
10. [客户端集成](#10-客户端集成)
11. [完整示例](#11-完整示例)
12. [常见问题](#12-常见问题)
13. [参考资料](#13-参考资料)

---

## 1. 简介

**FastMCP** 是 MCP Python SDK 提供的高层框架，用于快速构建 MCP 服务器。它简化了工具、资源和提示词的定义，自动处理协议细节。

### 主要特性

| 特性 | 说明 |
|------|------|
| 装饰器语法 | 使用 `@mcp.tool()` 等装饰器快速定义功能 |
| 类型注解支持 | 自动生成 JSON Schema |
| 结构化输出 | 支持返回 Pydantic 模型 |
| 上下文注入 | 自动注入 Context 对象 |
| 生命周期管理 | 支持 startup/shutdown 钩子 |
| 多传输协议 | 支持 stdio、SSE、Streamable HTTP |

---

## 2. 快速开始

### 2.1 安装依赖

```bash
# 推荐使用 uv
uv init mcp-server-demo
cd mcp-server-demo
uv add "mcp[cli]"

# 或使用 pip
pip install "mcp[cli]"
```

### 2.2 最简单的服务器

```python
from mcp.server.fastmcp import FastMCP

# 创建 MCP 服务器
mcp = FastMCP("Demo Server", json_response=True)

# 定义工具
@mcp.tool()
def add(a: int, b: int) -> int:
    """加法运算"""
    return a + b

# 运行服务器
if __name__ == "__main__":
    mcp.run()
```

### 2.3 测试服务器

```bash
# 方法一：直接运行
python server.py

# 方法二：使用 MCP Inspector
uv run mcp dev server.py

# 方法三：Claude Desktop 集成
uv run mcp install server.py --name "my-server"
```

---

## 3. 核心概念

### 3.1 MCP 三大要素

| 要素 | 控制方 | 用途 | 类比 |
|------|---------|------|------|
| **Tools（工具）** | 模型 | 执行操作、产生副作用 | POST 请求 |
| **Resources（资源）** | 应用 | 提供上下文数据 | GET 请求 |
| **Prompts（提示词）** | 用户 | 交互模板 | 命令/菜单 |

### 3.2 FastMCP 初始化参数

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(
    name="My Server",           # 服务器名称
    json_response=True,           # 使用 JSON 响应（推荐）
    stateless_http=False,        # 是否使用无状态 HTTP 模式
    streamable_http_path="/mcp",  # Streamable HTTP 路径
    instructions="服务器说明文字",  # 向客户端说明服务器用途
    website_url="https://example.com",  # 网站地址
    icons=[...]                 # UI 图标
    lifespan=lifespan_func,      # 生命周期管理函数
    token_verifier=...,          # Token 验证器（OAuth）
    auth=...,                  # 认证配置
)
```

---

## 4. 工具开发

### 4.1 基础工具定义

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Calculator", json_response=True)

@mcp.tool()
def add(a: int, b: int) -> int:
    """加法运算

    Args:
        a: 第一个数字
        b: 第二个数字
    """
    return a + b

@mcp.tool()
def greet(name: str = "World", style: str = "friendly") -> str:
    """向某人问好

    Args:
        name: 名字
        style: 风格 (friendly, formal, casual)
    """
    styles = {
        "friendly": f"你好，{name}！",
        "formal": f"您好，{name}。",
        "casual": f"嘿，{name}"
    }
    return styles.get(style, styles["friendly"])
```

### 4.2 异步工具

```python
import asyncio
import httpx

@mcp.tool()
async def fetch_url(url: str) -> str:
    """获取网页内容"""
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.text[:500]  # 限制长度
```

### 4.3 工具元数据

```python
from mcp.server.fastmcp import FastMCP, Icon

icon = Icon(src="icon.png", mimeType="image/png", sizes="64x64")

@mcp.tool(
    name="calculate",           # 自定义工具名（默认使用函数名）
    description="执行计算",    # 自定义描述
    title="计算器",           # 显示标题
    icons=[icon]               # 工具图标
)
def add_numbers(a: float, b: float) -> float:
    """加法"""
    return a + b
```

### 4.4 错误处理

```python
@mcp.tool()
def divide(a: float, b: float) -> float:
    """除法运算"""
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b

# FastMCP 会自动捕获异常并返回给客户端
```

---

## 5. 资源开发

### 5.1 静态资源

```python
@mcp.resource("config://settings")
def get_settings() -> str:
    """获取服务器配置"""
    return """{
  "theme": "dark",
  "language": "zh-CN",
  "debug": false
}"""

@mcp.resource("file://documents/{name}")
def read_document(name: str) -> str:
    """读取文档内容"""
    return f"文档 {name} 的内容..."
```

### 5.2 动态资源

```python
from pydantic import BaseModel, Field

class WeatherInfo(BaseModel):
    temperature: float = Field(description="温度")
    humidity: float = Field(description="湿度")
    condition: str = Field(description="天气状况")

@mcp.resource("weather://{city}")
async def get_weather_resource(city: str) -> WeatherInfo:
    """获取城市天气"""
    import random
    return WeatherInfo(
        temperature=random.uniform(15, 30),
        humidity=random.uniform(30, 80),
        condition=random.choice(["晴天", "多云", "阴天"])
    )
```

### 5.3 资源元数据

```python
from pydantic import AnyUrl

@mcp.resource(
    uri="data://users/{id}",  # URI 模板
    name="用户数据",          # 显示名称
    description="获取用户详细信息",
    title="用户档案",          # 显示标题
    mimeType="application/json"  # MIME 类型
)
def get_user(id: str) -> str:
    return f"{{\"id\": \"{id}\", \"name\": \"Alice\"}}"
```

---

## 6. 提示词开发

### 6.1 简单提示词

```python
@mcp.prompt()
def greet_user(name: str, style: str = "friendly") -> str:
    """生成问候提示词"""
    styles = {
        "friendly": "请用温暖友好的语调问候",
        "formal": "请用正式专业的语调问候",
        "casual": "请用轻松随意的语调问候"
    }
    return f"{styles.get(style, styles['friendly'])} 名为 {name} 的人。"
```

### 6.2 多消息提示词

```python
from mcp.server.fastmcp.prompts import base

@mcp.prompt(title="代码审查")
def review_code(code: str) -> list[base.Message]:
    """代码审查提示词"""
    return [
        base.UserMessage("请审查以下代码："),
        base.UserMessage(code),
        base.AssistantMessage("我会帮你检查代码质量。请告诉我你主要关注什么方面？")
    ]
```

### 6.3 提示词元数据

```python
from mcp.types import PromptArgument

@mcp.prompt(
    name="generate-email",      # 提示词名称
    description="生成邮件模板",   # 描述
    title="邮件生成器",         # 标题
    arguments=[                 # 参数定义
        PromptArgument(
            name="recipient",
            description="收件人",
            required=True
        ),
        PromptArgument(
            name="tone",
            description="语调",
            required=False
        )
    ]
)
def email_prompt(recipient: str, tone: str = "professional") -> str:
    return f"给 {recipient} 写一封 {tone} 的邮件。"
```

---

## 7. 上下文使用

### 7.1 注入 Context

```python
from mcp.server.fastmcp import Context, FastMCP
from mcp.server.session import ServerSession

mcp = FastMCP("Context Demo")

@mcp.tool()
async def process_data(
    data: str,
    ctx: Context[ServerSession, None]  # 自动注入 Context
) -> str:
    """使用上下文的工具"""
    # 发送日志
    await ctx.debug(f"处理数据: {data}")
    await ctx.info("开始处理")
    await ctx.warning("这是一个实验性功能")

    # 返回结果
    return f"已处理: {data}"
```

### 7.2 进度报告

```python
@mcp.tool()
async def long_running_task(
    task_name: str,
    steps: int = 5,
    ctx: Context[ServerSession, None] = None
) -> str:
    """执行长耗时任务"""
    await ctx.info(f"开始任务: {task_name}")

    for i in range(steps):
        progress = (i + 1) / steps
        await ctx.report_progress(
            progress=progress,
            total=1.0,
            message=f"步骤 {i + 1}/{steps}"
        )
        await asyncio.sleep(0.5)  # 模拟耗时

    return f"任务 '{task_name}' 完成"
```

### 7.3 资源读取

```python
from pydantic import AnyUrl

@mcp.tool()
async def summarize_config(ctx: Context[ServerSession, None]) -> str:
    """总结配置"""
    # 读取资源
    content = await ctx.read_resource(AnyUrl("config://settings"))
    return f"配置摘要: {content.contents[0].text[:100]}"
```

### 7.4 用户交互（Elicitation）

```python
from pydantic import BaseModel, Field

class Preferences(BaseModel):
    confirm: bool = Field(description="确认操作")
    notes: str = Field(default="", description="备注")

@mcp.tool()
async def confirm_action(
    action: str,
    ctx: Context[ServerSession, None]
) -> str:
    """需要用户确认的操作"""
    result = await ctx.elicit(
        message=f"确认要执行 {action} 吗？",
        schema=Preferences
    )

    if result.action == "accept" and result.data.confirm:
        return f"已执行: {action}，备注: {result.data.notes}"
    return "操作已取消"
```

### 7.5 访问服务器配置

```python
@mcp.tool()
def server_info(ctx: Context) -> dict:
    """获取服务器信息"""
    return {
        "name": ctx.fastmcp.name,
        "instructions": ctx.fastmcp.instructions,
        "debug_mode": ctx.fastmcp.settings.debug
    }
```

---

## 8. 结构化输出

### 8.1 使用 Pydantic 模型

```python
from pydantic import BaseModel, Field

class WeatherData(BaseModel):
    """天气数据结构"""
    temperature: float = Field(description="温度（摄氏度）")
    humidity: float = Field(description="湿度百分比")
    condition: str = Field(description="天气状况")
    wind_speed: float = Field(description="风速（km/h）")

@mcp.tool()
def get_weather(city: str) -> WeatherData:
    """获取城市天气 - 返回结构化数据"""
    return WeatherData(
        temperature=22.5,
        humidity=65.0,
        condition="晴天",
        wind_speed=5.2
    )
```

### 8.2 使用 TypedDict

```python
from typing import TypedDict

class LocationInfo(TypedDict):
    latitude: float
    longitude: float
    name: str

@mcp.tool()
def get_location(address: str) -> LocationInfo:
    """获取位置坐标"""
    return LocationInfo(
        latitude=51.5074,
        longitude=-0.1278,
        name="伦敦, 英国"
    )
```

### 8.3 返回字典

```python
@mcp.tool()
def get_statistics(data_type: str) -> dict[str, float]:
    """获取统计数据"""
    return {
        "mean": 42.5,
        "median": 40.0,
        "std_dev": 5.2
    }
```

### 8.4 列表返回

```python
@mcp.tool()
def list_cities() -> list[str]:
    """获取城市列表"""
    return ["北京", "上海", "深圳", "杭州"]
```

### 8.5 自定义 CallToolResult

```python
from mcp.types import CallToolResult, TextContent

@mcp.tool()
def advanced_tool() -> CallToolResult:
    """完全控制响应"""
    return CallToolResult(
        content=[TextContent(type="text", text="模型可见的内容")],
        structuredContent={"result": "success", "data": {"key": "value"}},
        _meta={"hidden": "仅客户端可见的元数据"}
    )
```

---

## 9. 服务器运行

### 9.1 Stdio 传输（默认）

```python
if __name__ == "__main__":
    mcp.run()  # 默认使用 stdio
```

### 9.2 Streamable HTTP 传输（推荐生产环境）

```python
if __name__ == "__main__":
    mcp.run(transport="streamable-http")
    # 服务器将运行在 http://localhost:8000/mcp
```

### 9.3 SSE 传输

```python
if __name__ == "__main__":
    mcp.run(transport="sse")
```

### 9.4 自定义主机和端口

```python
if __name__ == "__main__":
    mcp.settings.host = "0.0.0.0"
    mcp.settings.port = 3000
    mcp.run(transport="streamable-http")
```

### 9.5 生命周期管理

```python
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from dataclasses import dataclass

@dataclass
class AppContext:
    db: Database
    cache: Cache

@asynccontextmanager
async def lifespan(server: FastMCP) -> AsyncIterator[AppContext]:
    """应用生命周期管理"""
    # 启动时初始化
    db = await Database.connect()
    cache = Cache()
    try:
        yield AppContext(db=db, cache=cache)
    finally:
        # 关闭时清理
        await db.disconnect()
        await cache.close()

# 传递 lifespan 到服务器
mcp = FastMCP("My App", lifespan=lifespan)

# 在工具中访问生命周期资源
@mcp.tool()
def query_database(
    query: str,
    ctx: Context[ServerSession, AppContext]
) -> str:
    """查询数据库"""
    db = ctx.request_context.lifespan_context.db
    return db.execute(query)
```

---

## 10. 客户端集成

### 10.1 Claude Desktop 集成

```json
{
  "mcpServers": {
    "my-server": {
      "command": "python",
      "args": ["/path/to/server.py"],
      "cwd": "/path/to/project",
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```

配置文件位置：
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`

### 10.2 使用 MCP CLI 安装

```bash
# 安装到 Claude Desktop
uv run mcp install server.py --name "my-server"

# 传递环境变量
uv run mcp install server.py -v API_KEY=abc123

# 从 .env 文件读取
uv run mcp install server.py -f .env
```

### 10.3 MCP Inspector 测试

```bash
# 启动服务器
python server.py

# 另一个终端启动 Inspector
npx -y @modelcontextprotocol/inspector
```

在 Inspector UI 中连接到你的服务器。

### 10.4 自定义客户端连接

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(
    command="python",
    args=["server.py"]
)

async def main():
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 列出工具
            tools = await session.list_tools()
            print(f"工具: {[t.name for t in tools.tools]}")

            # 调用工具
            result = await session.call_tool("add", {"a": 5, "b": 3})
            print(f"结果: {result}")

asyncio.run(main())
```

---

## 11. 完整示例

### 项目结构

```
mcp-server/
├── pyproject.toml
├── .env
├── server.py          # 主入口
├── tools/
│   ├── __init__.py
│   ├── calculator.py   # 计算器工具
│   └── weather.py     # 天气工具
└── services/
    ├── __init__.py
    └── data_service.py # 业务服务
```

### pyproject.toml

```toml
[project]
name = "my-mcp-server"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = [
    "mcp>=1.0.0",
    "pydantic>=2.0.0",
    "httpx>=0.27.0",
    "python-dotenv>=1.0.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### .env

```env
API_KEY=your_api_key
DEBUG=False
```

### server.py

```python
#!/usr/bin/env python3
"""MCP Server 主入口"""

import os
from dotenv import load_dotenv

from mcp.server.fastmcp import FastMCP

load_dotenv()

# 创建服务器
mcp = FastMCP(
    name="My MCP Server",
    json_response=True,
    instructions="提供计算器和天气查询功能"
)

# 导入工具模块
from tools import calculator, weather

# 注册资源
@mcp.resource("config://info")
def get_config() -> str:
    """服务器配置信息"""
    return f"""
    服务器: My MCP Server
    版本: 0.1.0
    API Key: {os.getenv('API_KEY', 'Not set')}
    """

if __name__ == "__main__":
    mcp.run()
```

### tools/calculator.py

```python
"""计算器工具模块"""

from server import mcp

@mcp.tool()
def add(a: int, b: int) -> int:
    """加法运算"""
    return a + b

@mcp.tool()
def subtract(a: int, b: int) -> int:
    """减法运算"""
    return a - b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """乘法运算"""
    return a * b

@mcp.tool()
def divide(a: float, b: float) -> float:
    """除法运算"""
    if b == 0:
        raise ValueError("除数不能为零")
    return a / b
```

### tools/weather.py

```python
"""天气查询工具模块"""

from pydantic import BaseModel, Field
from server import mcp

class WeatherData(BaseModel):
    temperature: float = Field(description="温度（摄氏度）")
    humidity: float = Field(description="湿度百分比")
    condition: str = Field(description="天气状况")

@mcp.tool()
def get_weather(city: str) -> WeatherData:
    """获取指定城市的天气信息"""
    # 模拟数据，实际应用中应调用天气 API
    import random
    return WeatherData(
        temperature=round(random.uniform(15, 30), 1),
        humidity=round(random.uniform(30, 80), 1),
        condition=random.choice(["晴天", "多云", "阴天", "小雨"])
    )

@mcp.tool()
def get_forecast(city: str, days: int = 3) -> list[dict]:
    """获取未来几天的天气预报"""
    if days < 1 or days > 7:
        raise ValueError("预报天数必须在 1-7 之间")

    import random
    forecast = []
    for i in range(1, days + 1):
        forecast.append({
            "day": f"第{i}天",
            "temperature": round(random.uniform(15, 30), 1),
            "condition": random.choice(["晴天", "多云", "阴天", "小雨"])
        })
    return forecast
```

### services/data_service.py

```python
"""业务服务层"""

class DataService:
    """模拟数据服务"""

    def __init__(self):
        self._users = {
            "1": {"name": "Alice", "age": 30},
            "2": {"name": "Bob", "age": 25},
        }

    def get_user(self, user_id: str) -> dict | None:
        """获取用户信息"""
        return self._users.get(user_id)

    def list_users(self) -> list[dict]:
        """列出所有用户"""
        return list(self._users.values())

# 单例实例
data_service = DataService()
```

---

## 12. 常见问题

### Q1: 工具返回了复杂对象，但客户端只看到文本？

A: 确保返回类型有正确的类型注解。FastMCP 会根据注解自动生成 Schema。

### Q2: 如何调试工具调用？

A: 使用 `ctx.debug()` 发送调试日志，或在 MCP Inspector 中查看详细信息。

### Q3: 服务器启动后看不到工具？

A: 检查：
1. 工具函数是否正确导入
2. 装饰器 `@mcp.tool()` 是否正确应用
3. 查看服务器日志是否有错误

### Q4: 如何处理认证？

A: 使用 `token_verifier` 参数和 `AuthSettings` 配置 OAuth 2.1 认证。

### Q5: Streamable HTTP 与 stdio 有什么区别？

A:
- **stdio**: 适合本地开发和桌面客户端集成
- **Streamable HTTP**: 适合生产环境、多节点部署、Web 客户端

### Q6: 如何处理大文件上传？

A: 对于大文件，建议使用分块上传或通过资源 URI 引用，而不是直接传递文件内容。

---

## 13. 参考资料

- [MCP Python SDK GitHub](https://github.com/modelcontextprotocol/python-sdk)
- [MCP 官方文档](https://modelcontextprotocol.io)
- [FastMCP API 参考](https://github.com/modelcontextprotocol/python-sdk/tree/main/src/mcp/server/fastmcp)
- [MCP 规范](https://spec.modelcontextprotocol.io/)
- [Claude Desktop 集成指南](https://modelcontextprotocol.io/docs/develop/connect-local-servers)
