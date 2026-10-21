# MCP
{docsify-updated}

> https://modelcontextprotocol.io/docs/2026-07-28/getting-started/intro


## 什么是 MCP
MCP（模型上下文协议）是一项用于将**人工智能应用**与**外部系统**连接的开源标准。  
借助 MCP，Claude 或 ChatGPT 等人工智能应用可以连接到数据源（例如本地文件、数据库）、工具（例如搜索引擎、计算器）和工作流（例如专用提示词），从而使其能够访问关键信息并执行任务。
可以将 MCP 视为人工智能应用程序的 USB-C 接口。正如 USB-C 为电子设备提供了标准化的连接方式，MCP 也为人工智能应用程序与外部系统之间的连接提供了标准化的方式。

<center><img src="mcp-simple-diagram.avif" width=""></center>

### MCP 能提供哪些功能
+ 智能助手可以访问您的 Google 日历和 Notion，充当更个性化的 AI 助手。
+ Claude Code 可以基于 Figma 设计生成完整的 Web 应用。
+ 企业级聊天机器人可以连接组织内的多个数据库，让用户能够通过聊天分析数据。
+ AI 模型可以在 Blender 上创建 3D 设计，并通过 3D 打印机将其打印出来。

## 核心概念

### 参与者
MCP 架构中的主要参与者包括：
+ MCP HOST ：负责协调和管理一个或多个 MCP 客户端的人工智能应用程序
+ MCP Client ：与 MCP 服务器保持连接，并从 MCP 服务器获取上下文供 MCP 主机使用的组件
+ MCP Server ：向 MCP 客户端提供上下文的程序

例如：Visual Studio Code 充当 MCP 主机。当 Visual Studio Code 与 MCP 服务器（例如 Sentry MCP 服务器）建立连接时，Visual Studio Code 运行时会实例化一个 MCP 客户端对象，用于维护与 Sentry MCP 服务器的连接。当 Visual Studio Code 随后连接到另一个 MCP 服务器（例如本地文件系统服务器）时，Visual Studio Code 运行时会实例化另一个 MCP 客户端对象来维护该连接。

<center><img src="pics/mcp-prticipants.png" alt=""></center>

请注意，MCP 服务器指的是提供上下文数据的程序，无论其在何处运行。MCP 服务器可以在本地或远程执行。例如，当 Claude Desktop 启动文件系统服务器时，由于该服务器使用 STDIO 传输协议，因此会在同一台机器上本地运行。这通常被称为“本地”MCP 服务器。官方的 Sentry MCP 服务器运行在 Sentry 平台上，并使用 Streamable HTTP 传输协议。这通常被称为“远程”MCP 服务器。

注意：完全可以实现一套通用 MCP Client，支持连接多个不同的 MCP Server。Agent 的核心工作则是在这些 Server 暴露的工具中进行发现、选择和调度，而不是为每个 Server 重写一套通信客户端。这也是为什么很多 Agent 只需要执行 MCP server 就可以的原因。同时，MCP Client 的协议逻辑可以复用，但不同 Server 的连接方式可能不同：
- stdio：启动本地子进程，通过标准输入输出通信。
- Streamable HTTP：通过 HTTP 端点通信，可涉及会话和认证机制。
因此，成熟的 MCP Client 通常会支持多种 Transport。即使工具调用 API 相同，底层连接和生命周期管理仍可能不同。

### 原语
MCP 原语是 MCP 中的最重要的概念。它们定义了客户端和服务器之间可以相互提供的内容。这些原语规定了可与人工智能应用程序共享的上下文信息类型，以及可执行的操作范围。  
MCP 定义了三种服务器可以公开的核心基本操作：
+ Tools ：AI 应用程序可调用以执行操作的可执行函数（例如，文件操作、API 调用、数据库查询）
+ Resources ：为 AI 应用程序提供上下文信息的数据源（例如，文件内容、数据库记录、API 响应）
+ Prompts ：有助于构建与语言模型交互结构的可复用模板（例如，系统提示词、少量示例）

每种原语都关联有用于发现（`*/list`）、检索（`*/get`）以及在某些情况下执行（`tools/call`）的方法。MCP 客户端将使用 `*/list` 方法来发现可用的原语。例如，客户端可以先列出所有可用的工具（`tools/list`），然后执行它们。这种设计使得列表能够动态更新。

MCP 还定义了客户端可以暴露的原语。这些原语使 MCP 服务器开发者能够构建更丰富的交互功能：
+ Elicitation ： 允许服务器向用户请求更多信息。当服务器开发者希望从用户处获取更多信息，或要求用户对某项操作进行确认时，此功能非常有用。服务器可通过 `elicitation/create` 方法请求用户输入。

除了服务器和客户端基本操作外，该协议还支持可选扩展。例如，“任务”（Tasks）扩展允许服务器为长期运行的请求返回一个持久化句柄，以便客户端可以轮询状态并在稍后获取结果。

### 通知
该协议支持实时通知，以实现服务器与客户端之间的动态更新。例如，当服务器的可用工具发生变化时（例如新增功能或现有工具被修改），服务器可以发送工具更新通知，以告知已连接的客户端这些变化。通知以 `JSON-RPC 2.0` 通知消息的形式发送（不期待收到响应）。变更通知采用“主动订阅”模式：客户端会打开一个长期存在的 `subscriptions/listen` 流，并指定其希望接收的通知类型，服务器则通过该流发送匹配的通知。

### inspector
```
npx @modelcontextprotocol/inspector //启动 inspector 服务,然后在界面上选择要inspect 的 mcp server

npx -y @modelcontextprotocol/inspector npx -y @modelcontextprotocol/server-memory // 启动 inspector 服务并连接到 memory server
```