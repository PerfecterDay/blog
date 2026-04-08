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