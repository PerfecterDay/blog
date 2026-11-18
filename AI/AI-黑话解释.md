```
你输入一句话（Prompt）        ←（被切成 tokens）
        ↓
系统拼接：
    - 当前问题（tokens）
    - 历史对话 Context（tokens）
    - 用户长期信息 Memory（tokens）
    - Agent 调用外部数据 MCP 提供（tokens）
        ↓
👉 以上全部拼成一个 “token 序列”
        ↓
一起喂给 LLM（按 token 处理）
        ↓
LLM 按 token 逐个生成回答（输出 tokens）
        ↓
如果是 Agent：
    工具调用结果 → 再转成 tokens → 再喂回 LLM
```

## Prompt Engineering
举个例子，你要求 Codex：将项目中的 OpenFeign 迁移到 RestClient。

Prompt Engineering 告诉它：
- 必须保持业务行为不变。
- 使用 Spring Boot 4。
- 必须保留 Resilience4j。
- 修改后必须运行 Maven 测试。
- 最后汇报 Git Diff。

Context Engineering 为它提供：
- 当前 OpenFeign 接口。
- Feign 配置。
- Resilience4j 配置。
- 相关 Service 调用代码。
- 项目依赖版本。
- 现有单元测试。
两者配合，才能得到更好的结果。


System Prompt + User Prompt

## 上下文
在 AI 编程 Agent 中，上下文（Context）就是 AI 为了理解当前任务而能够获取和使用的所有相关信息。

你可以把它理解成：一个 Java 开发人员接手一个陌生项目时，需要先了解的背景资料。


一个假设的上下文窗口： 100,000 Tokens 。假设项目有 100 万行代码，不可能每次都把所有代码塞进模型的上下文窗口。因此，Coding Agent 必须解决一个问题：
从庞大的代码库中，挑选出当前任务真正需要的信息。

这就是 Augment Context Engine 试图解决的问题。


| 对比维度 | Prompt Engineering | Context Engineering |
|---|---|---|
| 核心问题 | 怎么向模型提要求 | 给模型提供什么信息 |
| 主要工作 | 设计指令 | 检索、筛选、组织上下文 |
| 关注重点 | 指令质量 | 信息质量和相关性 |
| 常见手段 | 角色、规则、示例、输出格式 | RAG、代码索引、记忆、工具调用 |
| 典型产物 | Prompt | 模型可用的上下文 |

Prompt Engineering 解决的是如何正确地告诉 AI 做什么；Context Engineering 解决的是如何让 AI 知道完成任务所需要的信息；Agent Engineering 则解决的是如何让 AI 自主完成整个任务闭环。


## agent
Agent 最终调用大模型 API，本质上还是一次普通的 LLM API 请求。

但与直接调用 Chat Completion 不同的是：Agent 会在多次 API 调用之间维护对话状态、收集上下文、调用工具，并把工具执行结果再次提交给大模型。

所以，一个 Agent 解决问题，通常不是一次 API 调用，而是一个循环：
1. 收到用户任务。
2. 组织 Prompt 和 Context。
3. 调用 LLM。
4. LLM 决定直接回答，还是调用工具。
5. Agent 执行工具。
6. 把工具结果追加到上下文。
7. 再次调用 LLM。
8. 重复以上过程，直到任务完成。