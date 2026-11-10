---
title: "从 LLM 到 Agent"
date: 2026-11-10
permalink: /blogs/from-llm-to-agent/
lang: zh-CN
series: inference
series_order: 6
read_time: true
read_time_minutes: 5
excerpt: "拿一次改代码的任务，把 RAG、工具调用、MCP 和 Agent 循环串起来。"
tags: [llm, agent, rag, mcp]
---

上一篇讲到，Agent 会一遍遍带着旧上下文调用模型。那它到底在一遍遍算什么？拿一个我平时会交给 coding agent 的活来说：“找出仓库里哪些测试还在依赖真实时钟，把最容易偶发失败的那个改掉，跑一下测试。”

如果我把这句话直接发给一个没有工具的模型，它大概也能写出一份像模像样的方案。问题是，它连仓库里有哪些文件都不知道，更别说改完没有、测试过没过。要把这活儿做完，得先给它看东西，再给它办事的办法。

## 先让模型看见仓库

最直接的办法是把文件贴给模型。文件多了，就先找出相关内容，再把找到的东西放进上下文。比如先搜 `time.sleep` 和 `datetime.now`，找到几个候选文件，再读它们附近的代码和测试说明。

**RAG（Retrieval-Augmented Generation，检索增强生成）**也是这个思路：先检索，再用检索结果辅助生成。最常见的例子是把文档切块、建索引，回答问题时找出相关段落。向量检索擅长找意思相近的内容，像函数名这种精确匹配，用关键词搜索往往更直接。RAG 里的检索不必是向量检索，也不一定要由模型发起。[^rag]

比如我问的是“为什么 CI 偶尔超时”，用语义检索找历史故障记录可能有用。已经知道要找 `time.sleep`，直接搜代码就行。两种办法都能把原来不在 prompt 里的材料交给模型，但找到了相关材料，也只意味着它有了判断的依据。

## 模型说要搜，谁去搜？

假设我们给模型一个 `search_files` 工具，告诉它工具能做什么、参数怎么填。模型判断需要搜索，就返回一条结构化请求，意思是“在测试目录里搜 `time.sleep`”。

接下来，由**应用里的运行时**检查参数和权限，再真的去搜。搜完，把文件名、行号和命中内容放回上下文，再调用一次模型。模型看到结果后，才知道下一步该读哪个文件。改文件和跑测试也一样：模型提出请求，运行时执行，结果再回到模型这儿。[^tools]

这一步有个我觉得挺容易混的地方：模型说“测试过了”，和工具真的返回“测试通过”，是两回事。第二种情况下还有一个问题：跑的是改过的那条测试，还是随便跑了一条？这些事最终得看运行记录。

<figure class="inference-figure">
{% include inference/agent-tool-loop.svg %}
<figcaption>图里模型先提出搜索请求，再读回传的结果。收到结果后，它可以继续操作，也可以给出最终回复。</figcaption>
</figure>

## 那 MCP 管哪一段？

先说，没有 MCP 也能做上面这件事。应用自己实现一个 `search_files`，用模型 API 的工具调用把它接上，已经够用。麻烦是在工具多了之后：本地文件、GitHub、工单系统各有接口，每个应用都要重新接一遍。

**MCP（Model Context Protocol）**约定的是应用和外部能力之间怎么对接。拿搜索工具举例：[^mcp]

- **Host** 是你正在用的应用。它负责和模型打交道，管理会话，并决定给模型哪些工具。
- **Client** 在 Host 里面，负责和某个 MCP Server 通信。它可以用 `tools/list` 发现可用工具，再用 `tools/call` 发出请求。
- **Server** 提供这些能力。文件搜索可以由 Server 自己执行，GitHub 工具也可以由 Server 转调 GitHub API。

模型返回的工具请求和 Client 发出的 MCP 消息，是两段不同的通信。Host 可以把几个 Server 的工具和它自己的工具一起交给模型，也可以按任务只暴露其中一部分。MCP 还定义了 **Resources** 和 **Prompts**，分别用于提供可读的数据和可复用的提示模板。至于账号能访问哪些仓库、某个操作要不要审批，仍得由实际的权限和执行环境来决定。

## 从调用一次工具，到把活做完

找到了几处真实时钟的用法之后，模型可能先读最可疑的一个测试，发现它通过等 100 毫秒来猜后台任务已经结束。于是它改成等待任务明确完成，再跑这个测试。结果报错：本机少了依赖。接下来是装依赖、换仓库规定的测试入口，还是直接报告验证受阻，要看环境和权限。

这就是我理解的 **Agent 循环**。运行时拿着任务和已有结果反复调用模型，让它决定接下来做什么。每一步的工具结果会改变下一步的决定。这个循环也得有退出条件：任务完成了，遇到了需要人处理的阻塞，或者已经用完预算。

回过头看上一篇的 KV Cache，就知道为什么 Agent 会有那么多重复的输入。读文件、改文件、跑测试，每一步之后都得让模型继续看之前的任务和刚刚拿到的结果。前面的上下文经常能复用，后面则一直在追加。

现在能跑这个循环了。但如果模型只改完代码就回一句“搞定”，或者修了半天忘了用户最初只让改一个测试，工具再多也没用。下一篇聊这部分。

[^rag]: Lewis 等，[Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)，介绍检索与生成相结合的做法。
[^tools]: OpenAI，[Function calling](https://platform.openai.com/docs/guides/function-calling)，工具请求、应用执行与结果回传。
[^mcp]: Model Context Protocol，[Architecture overview](https://modelcontextprotocol.io/docs/learn/architecture)，Host、Client、Server 及协议原语。
