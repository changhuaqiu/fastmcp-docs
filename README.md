# FastMCP 开发指南

> Model Context Protocol (MCP) FastMCP 框架完整开发文档

## 简介

本项目提供 FastMCP 框架的完整开发指南，帮助开发者快速构建 MCP 服务器。

## 文档

- [FastMCP 完整开发指南](docs/fastmcp-guide.md)

## 目录

1. **简介** - FastMCP 框架概述和主要特性
2. **快速开始** - 安装、创建第一个服务器
3. **核心概念** - MCP 三大要素、FastMCP 初始化
4. **工具开发** - 定义各种类型的工具
5. **资源开发** - 静态和动态资源
6. **提示词开发** - 创建交互式提示词
7. **上下文使用** - Context 对象的高级用法
8. **结构化输出** - Pydantic 模型、TypedDict 等
9. **服务器运行** - 多种传输协议、生命周期管理
10. **客户端集成** - Claude Desktop、MCP Inspector 等
11. **完整示例** - 完整的项目结构示例
12. **常见问题** - 常见问题解答
13. **参考资料** - 相关文档链接

## 什么是 FastMCP？

FastMCP 是 MCP Python SDK 提供的高层框架，用于快速构建 MCP 服务器。它通过装饰器语法简化了工具、资源和提示词的定义。

### 主要特性

| 特性 | 说明 |
|------|------|
| 装饰器语法 | 使用 `@mcp.tool()` 等装饰器快速定义功能 |
| 类型注解支持 | 自动生成 JSON Schema |
| 结构化输出 | 支持返回 Pydantic 模型 |
| 上下文注入 | 自动注入 Context 对象 |
| 生命周期管理 | 支持 startup/shutdown 钩子 |
| 多传输协议 | 支持 stdio、SSE、Streamable HTTP |

## 快速开始

```bash
# 安装依赖
pip install "mcp[cli]"

# 创建服务器
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Demo", json_response=True)

@mcp.tool()
def add(a: int, b: int) -> int:
    """加法运算"""
    return a + b

if __name__ == "__main__":
    mcp.run()
```

## 相关资源

- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP 官方文档](https://modelcontextprotocol.io)

## 许可证

MIT License
