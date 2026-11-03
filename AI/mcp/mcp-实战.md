## 编写一个mcp server
{docsify-updated}

使用了 gitlab mcp server，但是报错：
`Your enabled tools are consuming most of the model's context window. Try disabling some MCP servers or tools to free up space.`
原因是 gitlab mcp server 提供的 tools 太多，导致上下文窗口被填满。所以需要编写一个 mcp server，只允许 `merge_request` 相关的 tools。

```
mkdir mcp-proxy
cd mcp-proxy

npm init -y

npm install @modelcontextprotocol/sdk
```

编写一个 mcp server，只允许 merge_request 相关的 tools：
```

import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

import {
  ListToolsRequestSchema,
  CallToolRequestSchema
} from "@modelcontextprotocol/sdk/types.js";

// 1. Tool 白名单
const ALLOWED_TOOLS = "merge_request";

// 2. 创建上游 MCP Client
const upstreamClient = new Client(
  {
    name: "mcp-proxy-client",
    version: "1.0.0"
  },
  {
    capabilities: {}
  }
);

// 假设原始 MCP Server 使用 stdio
const upstreamTransport = new StdioClientTransport({
  command: "npx",
  args: ["-y", "@zereight/mcp-gitlab@latest"],
  env: {
    GITLAB_PERSONAL_ACCESS_TOKEN: "xxx",
    GITLAB_API_URL: "xxx"
  }
});

// 3. 连接原始 MCP Server
await upstreamClient.connect(upstreamTransport);

// 4. 获取原始 Tool 列表
const { tools } = await upstreamClient.listTools();

// 5. 按白名单过滤
const filteredTools = tools.filter(tool =>
  // console.error(
  // ` tools: ${tool.name}}`
  // )
  tool.name.includes("merge_request")
);

console.error(
  `Original tools: ${tools.length}, Enabled tools: ${filteredTools.length}`
);

// 6. 创建下游 MCP Server
const proxyServer = new Server(
  {
    name: "mcp-whitelist-proxy",
    version: "1.0.0"
  },
  {
    capabilities: {
      tools: {
        listChanged: false
      }
    }
  }
);

// 7. 拦截 tools/list
proxyServer.setRequestHandler(
  ListToolsRequestSchema,
  async () => ({
    tools: filteredTools
  })
);

// 8. 拦截 tools/call
proxyServer.setRequestHandler(
  CallToolRequestSchema,
  async (request) => {
    const { name, arguments: args } = request.params;

    // 再次校验白名单，避免绕过 tools/list
    if (!ALLOWED_TOOLS.has(name)) {
      throw new Error(`Tool not allowed: ${name}`);
    }

    // 确认上游确实存在该 Tool
    if (!filteredTools.some(tool => tool.name === name)) {
      throw new Error(`Unknown tool: ${name}`);
    }

    // 原样转发调用参数
    return await upstreamClient.callTool({
      name,
      arguments: args ?? {}
    });
  }
);

// 9. 启动下游 stdio Server
const downstreamTransport = new StdioServerTransport();

await proxyServer.connect(downstreamTransport);

// 10. 进程退出时清理上游连接
async function shutdown() {
  await proxyServer.close();
  await upstreamClient.close();
  process.exit(0);
}

process.on("SIGINT", shutdown);
process.on("SIGTERM", shutdown);

console.error("MCP Proxy started");
```

然后执行：
```
npm pkg set type=module
node index.js
```

debug : 
```
npx -y @modelcontextprotocol/inspector node /Users/gtja/workspace/blog/AI/mcp-rpoxy/mcp-proxy.js
```
