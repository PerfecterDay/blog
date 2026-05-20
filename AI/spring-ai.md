# Spring AI
{docsify-updated}

> https://docs.spring.io/spring-ai/reference/index.html

> 原文:[AI Concepts — Spring AI Reference 1.1.6](https://docs.spring.io/spring-ai/reference/concepts.html)
> 本文档为人工翻译,仅供学习参考。如有歧义,请以英文原文为准。
> 版权属于原作者(Broadcom Inc. 及其关联公司),依照 Apache License 2.0 发布。

本节介绍 Spring AI 所使用的核心概念,建议仔细阅读,以理解 Spring AI 实现背后的理念。

## 模型(Models)

AI 模型是用于处理和生成信息的算法,通常模仿人类的认知功能。通过从大规模数据集中学习模式与洞察,这些模型能够进行预测,生成文本、图像或其他输出,从而在各行各业增强应用能力。

AI 模型种类繁多,每一种都适合特定的使用场景。虽然 ChatGPT 及其生成式 AI 能力通过文本输入与输出吸引了大众,但市场上还有许多模型和厂商提供了多种多样的输入与输出形式。在 ChatGPT 之前,Midjourney、Stable Diffusion 等文生图模型同样让无数人着迷。

<center><img src="pics/spring-ai-concepts-model-types.jpg" alt=""></center>

Spring AI 目前支持以 **语言、图像、音频** 作为输入与输出的模型。上表最后一行(输入文本、输出数字)更常被称为 **文本嵌入(text embedding)**,代表 AI 模型内部使用的数据结构。Spring AI 对 embeddings 提供了支持,以实现更高级的用例。

GPT 这类模型的特别之处在于它们经过 **预训练**,这也是 GPT 中 "P" 的由来 —— Chat Generative **Pre-trained** Transformer。预训练让 AI 成为通用的开发者工具,使用者无需具备深厚的机器学习或模型训练背景即可上手。

## 提示词(Prompts)

Prompt 是引导 AI 模型生成特定输出的、基于语言的输入。对熟悉 ChatGPT 的人来说,prompt 看起来似乎只是输入到对话框、再发送到 API 的一段文字,但实际含义远不止于此。在许多 AI 模型中,prompt 的文本并不只是简单的字符串。

ChatGPT 的 API 在一次 prompt 中包含多段文本输入,每段都被分配一个 **角色(role)**:

- **system role**:告诉模型如何表现、设定本次交互的上下文。
- **user role**:通常是用户的实际输入。

设计有效的 prompt 既是艺术,也是科学。ChatGPT 是为"人类对话"而设计的,这与用 SQL 这样的方式提问截然不同 —— 你必须像与另一个人交谈一样与 AI 模型沟通。

这种交互方式如此重要,以至于 **"提示工程(Prompt Engineering)"** 已经发展为一门独立的学科。提升 prompt 效果的技巧层出不穷,在 prompt 上投入时间往往能显著改善生成结果。

分享 prompt 已成为社区常态,这方面也有活跃的学术研究。一个能说明编写有效 prompt 多么反直觉的例子是:一篇近期论文发现,最有效的 prompt 之一以 **"深呼吸,一步一步地处理这个问题(Take a deep breath and work on this step by step)"** 开头 —— 这也提示我们,语言在与 AI 沟通时为何如此重要。即便是 ChatGPT 3.5 这种较早版本,我们都还没有完全摸清如何最大化其能力,更不用说不断推出的新版本。

### 提示模板(Prompt Templates)

构造有效的 prompt 涉及两件事:**确立请求的上下文**,以及 **将请求中的部分内容替换为针对用户输入的具体值**。

这一过程使用传统的、基于文本的模板引擎来创建与管理 prompt。Spring AI 选用了开源库 **StringTemplate** 来实现这个目的。

例如,一个简单的提示模板可能是:

```
Tell me a {adjective} joke about {content}.
```

在 Spring AI 中,prompt template 类似 Spring MVC 架构中的 **"View"**。开发者提供一个模型对象(通常是 `java.util.Map`),用来填充模板里的占位符。"渲染"后的字符串即作为发送给 AI 模型的 prompt 内容。

发送给模型的 prompt 在具体数据格式上差异很大。最初只是简单的字符串,如今则演化为包含多条消息的结构,每条消息里的字符串都对应模型所识别的不同角色。

## 嵌入(Embeddings)

Embeddings 是文本、图像或视频的 **数值化表示**,用于捕捉输入之间的关联。

其机制是把文本、图像、视频转换为浮点数数组,即**向量(vector)**。这些向量被设计用来表达原始内容的"含义"。embedding 数组的长度称为向量的 **维度(dimensionality)**。

通过计算两段文本对应向量之间的数值距离,应用可以判断生成这些向量的对象之间的相似度。

<center><img src="pics/spring-ai-embeddings.jpg" alt=""></center>

作为探索 AI 的 Java 开发者,你不必深入理解这些向量表示背后的复杂数学理论或具体实现。对其在 AI 系统中所扮演的角色与作用有一个基本认知即可 —— 尤其是在把 AI 能力集成进自己应用的场景下。

Embeddings 在 **检索增强生成(RAG)** 这类实用场景中尤其重要。它们能将数据表示为 **语义空间** 中的点,类似欧氏几何里的二维平面,只是维度更高。就像平面上的点可以根据坐标"接近"或"远离",在语义空间中,点的距离反映了语义上的相似度。讨论相似话题的句子在多维空间中位置更靠近,这一性质有助于文本分类、语义搜索甚至商品推荐 —— AI 可以根据点在语义空间中的"位置"识别并归类相关概念。

你可以把这个语义空间想象成一个向量。

## 词元(Tokens)

Tokens 是 AI 模型工作的基本单元。**输入时**,模型把单词转换成 tokens;**输出时**,再把 tokens 转换回单词。

在英文中,1 个 token 大致相当于一个单词的 75%。作为参考:莎士比亚全集约 90 万词,大约对应 120 万 tokens。

<center><img src="pics/spring-ai-concepts-tokens.png" alt=""></center>

也许更重要的一点是:**Tokens = 钱**。对于托管型 AI 模型服务,你的费用由消耗的 token 数决定,**输入和输出都计入总数**。

此外,模型存在 **token 上限**,限制单次 API 调用中可处理的文本量,这个上限通常被称为 **"上下文窗口(context window)"**。超出上限的文本不会被模型处理。

例如,ChatGPT-3 的 token 上限为 4K;GPT-4 提供 8K、16K、32K 等不同选项;Anthropic 的 Claude 模型支持 100K tokens;Meta 近期的研究甚至推出了 100 万 tokens 上下文的模型。

如果想用 GPT-4 总结莎士比亚全集,你必须设计相应的软件工程方案,把数据切片并以适配上下文窗口的方式呈现给模型。Spring AI 项目正是用来帮你解决此类问题的。

## 结构化输出(Structured Output)

AI 模型的输出按传统是以 `java.lang.String` 返回的 —— 即使你要求模型返回 JSON,它也只是一段恰好是合法 JSON 的字符串,而不是 JSON 数据结构。仅在 prompt 里要求"返回 JSON"也并不能 100% 保证结果格式正确。

这一难点催生了一个专门方向:精心设计 prompt 以产生预期的输出格式,然后把得到的简单字符串转换为可用于应用集成的数据结构。

<center><img src="pics/structured-output-architecture.jpg" alt=""></center>

结构化输出转换需要精心打磨的 prompt,通常还需要多次与模型交互才能得到符合预期的格式。

## 把你的数据与 API 接入 AI 模型

如何让 AI 模型获得它训练时没见过的信息?

注意:GPT 3.5/4.0 的训练数据仅截止到 2021 年 9 月。对超出该时间点的问题,模型会回答"不知道"。一个有趣的冷知识是,这个训练数据集约 650GB。

让 AI 模型接入你自己的数据,有三种主流技术:

1. **微调(Fine Tuning)**:经典的机器学习手段,通过修改模型内部权重来定制模型。然而,微调对机器学习专家而言依旧具有挑战性,对 GPT 这类大型模型来说也极其耗费资源;此外,部分模型并不提供这个选项。
2. **提示填充(Prompt Stuffing)**:一种更实用的替代方法,将你自己的数据直接嵌入到提交给模型的 prompt 中。由于模型的 token 上限,需要相应技术把"相关数据"塞进上下文窗口。这种做法俗称 **"塞 prompt"**,也就是 **检索增强生成(RAG-Retrieval Augmented Generation)**。Spring AI 提供了基于这一思路的实现。
   <center><img src="pics/spring-ai-prompt-stuffing.jpg" alt=""></center>
3. **工具调用(Tool Calling)**:通过注册"工具"(用户定义的服务),把大语言模型与外部系统的 API 连接起来。Spring AI 极大简化了为支持工具调用所需编写的代码。

### 检索增强生成(RAG)

为了把相关数据正确地嵌入到 prompt 以获得准确的模型响应,衍生出了 **检索增强生成(Retrieval Augmented Generation, RAG)** 这一技术。

它的工作模式类似批处理:任务从你的文档中读取非结构化数据,进行转换,然后写入 **向量数据库**。本质上是一条 **ETL(Extract / Transform / Load)** 流水线。向量数据库被用于 RAG 流程中的 **检索** 环节。

把非结构化数据导入向量数据库的过程中,最重要的转换之一是 **把原始文档切分成更小的片段**。这一切分过程涉及两个关键步骤:

1. **保持语义边界进行切分**。例如对包含段落和表格的文档,要避免在段落或表格中间切开;对代码文件,要避免在方法实现的中间切开。
2. **进一步把片段切到 AI 模型 token 上限的一小部分**。

RAG 的下一阶段是处理用户输入:当 AI 模型需要回答一个用户问题时,该问题与所有"相似"的文档片段会一并放入发送给模型的 prompt 中。这正是要使用向量数据库的原因 —— 它非常擅长查找相似内容。

<center><img src="pics/spring-ai-rag.jpg" alt=""></center>

- **ETL Pipeline**:进一步说明了如何编排"从数据源抽取数据、以结构化方式存入向量数据库"的过程,确保数据以最适合检索的形式传给 AI 模型。
- **ChatClient — RAG**:说明了如何在应用中使用 `QuestionAnswerAdvisor` 启用 RAG 能力。

### 工具调用(Tool Calling)

大语言模型(LLM)在训练完成之后即被"冻结",导致知识停留在某一时间点,且模型本身无法访问或修改外部数据。

**工具调用(Tool Calling)** 机制解决了这一不足:它允许你把自己的服务注册为"工具",把 LLM 与外部系统的 API 连接起来。这些系统可以为 LLM 提供实时数据,或代表 LLM 执行数据处理动作。

Spring AI 极大简化了支持工具调用所需的代码,并替你处理工具调用的会话过程。你只需提供一个被 `@Tool` 注解标注的方法,然后在 prompt options 中引用它,模型就能使用该工具;同一次 prompt 中也可以定义并引用多个工具。

工作流程如下:

<center><img src="pics/tool-calling-01.jpg" alt=""></center>

1. 想让某个工具对模型可用时,把它的定义包含在 chat 请求里。每个工具定义包含 **名称、描述、输入参数 schema** 三部分。
2. 当模型决定调用某个工具时,它会返回一个响应,包含工具名以及按 schema 构造的输入参数。
3. 应用根据工具名定位并执行该工具,使用模型给出的参数。
4. 工具调用的结果由应用进行处理。
5. 应用把工具调用结果发回模型。
6. 模型把工具调用结果作为补充上下文,生成最终回答。

更多细节请参考 **Tool Calling** 官方文档,以了解如何在不同 AI 模型上使用该特性。

## 评估 AI 响应(Evaluating AI Responses)

对 AI 系统针对用户请求所产生的输出进行有效评估,对保证最终应用的准确性与实用性至关重要。一些新兴技术正是 **利用预训练模型自身** 来完成评估。

评估过程会分析所生成的回答是否符合用户意图、与查询的上下文是否一致。常用指标包括 **相关性(relevance)**、**连贯性(coherence)**、**事实正确性(factual correctness)** 等。

一种做法是把用户请求与模型生成的响应一并提交给模型,询问该响应是否与所提供的数据一致。

此外,还可以把存储在向量数据库中的信息作为补充数据加入评估过程,以辅助判断响应的相关性。

Spring AI 提供了 **Evaluator API**,目前包含一些用于评估模型响应的基础策略。更多内容见 **Evaluation Testing** 文档。

---

**翻译说明**:本文翻译自 Spring AI 官方文档的 "AI Concepts" 章节,版权属于原作者。中文版仅用于学习参考,术语首次出现时附英文原词以便对照。如发现翻译错误或理解偏差,请以 [英文原文](https://docs.spring.io/spring-ai/reference/concepts.html) 为准。
