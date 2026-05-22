# Spring AI Demo 分析
{docsify-updated}

> https://github.com/tzolov/playground-flight-booking

## 安装与环境配置
1. `export OPENAI_API_KEY=....`
2. `docker run -d --name chroma -p 8000:8000 chromadb/chroma`
3. 创建 chroma collection: 
   ```
    HOST=http://localhost:8000

    # 1) 创建 tenant
    curl -sf -X POST "$HOST/api/v2/tenants" \
    -H 'Content-Type: application/json' \
    -d '{"name":"SpringAiTenant"}' && echo " tenant OK"

    # 2) 在该 tenant 下创建 database
    curl -sf -X POST "$HOST/api/v2/tenants/SpringAiTenant/databases" \
    -H 'Content-Type: application/json' \
    -d '{"name":"SpringAiDatabase"}' && echo " database OK"

    # 3) 在该 database 下创建 collection
    curl -sf -X POST "$HOST/api/v2/tenants/SpringAiTenant/databases/SpringAiDatabase/collections" \
    -H 'Content-Type: application/json' \
    -d '{"name":"SpringAiCollection"}' && echo " collection OK"

    # 4) 校验
    echo "---- collections ----"
    curl -s "$HOST/api/v2/tenants/SpringAiTenant/databases/SpringAiDatabase/collections" | jq
   ```

## RAG：

何时	在 LLM 调用之前,每一轮对话都会执行(每次 chat())
触发条件	只要 QuestionAnswerAdvisor 在 advisor 链里就无条件执行 —— LLM 不参与决策
检索 query	直接取当前 user message 的纯文本(chatClientRequest.prompt().getUserMessage().getText())
检索过程	VectorStore.similaritySearch() → Spring AI 内部先调 EmbeddingModel.embed(query) 把问题向量化 → 再走 Chroma 的 /api/v2/.../query 接口拿 top-K
如何注入 prompt	用内置模板把原始问题和检索到的文档一起拼成一段新的 user message,替换原文


每次用户输入(比如"取消订单 101"):

向量化 → 调用智谱 https://open.bigmodel.cn/api/paas/v4/embeddings(配置里指定的 embedding-3),把"取消订单 101"转成 2048 维向量。
相似检索 → 在 Chroma 的 SpringAiCollection 里查 top-4(SearchRequest 默认 K=4)与之最接近的文档片段。
结果:很可能命中 terms-of-service.txt 里的"3. Cancelling Bookings - $75 economy / $50 premium / $25 business"这条。
改写 prompt:把这段条款连同原始问题拼成一段新 user message。
后续 LLM 看到的问题不再是"取消订单 101",而是"原问题 + 取消政策条款" —— 这就是模型能精准说"取消会有 $75 手续费,需要您确认"的原因。


## Tool Calling：
Get booking details
```
OpenAiChatModel
if (this.toolExecutionEligibilityPredicate.isToolExecutionRequired(prompt.getOptions(), response)) {
    var toolExecutionResult = this.toolCallingManager.executeToolCalls(prompt, response);
    if (toolExecutionResult.returnDirect()) {
        return ChatResponse.builder()...build();
    } else {
        // Send the tool execution result back to the model.
        return this.internalCall(
            new Prompt(toolExecutionResult.conversationHistory(), prompt.getOptions()),
            response);
    }
}
return response;
```

何时	每次 LLM 调用返回之后立刻检查
触发条件	isToolExecutionRequired —— 实际判断 response.getResult().getOutput().hasToolCalls(),即 LLM 返回里是否带了 tool_calls 字段
谁决定调谁	LLM 决定。Spring AI 在第一次请求时已经把所有 @Tool 方法的 JSON Schema 一并发过去了
执行体	ToolCallingManager.executeToolCalls() 反射调用对应的 @Tool 方法
关键回环	调用完后递归 internalCall(),把工具结果作为新的 message 接到对话历史末尾,再请求一次 LLM