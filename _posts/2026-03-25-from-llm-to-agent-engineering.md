---
title: 'From LLM to Agent: Why Token-In-Token-Out Is Not Enough'
date: 2026-03-25
tags:
  - llm
  - agent
  - rag
  - mcp
  - engineering
---

Large Language Models are impressive. They can write code, summarize papers, and hold conversations that feel remarkably human. But once you move past the demo and try to build something real, you quickly hit a wall: **an LLM, at its core, is just token-in, token-out.** You give it text, it gives you text back. That's it.

This post walks through the logical progression from a raw LLM to a full agent system — not as a glossary of buzzwords, but as a series of engineering problems and the solutions that emerged to address them.

## The Fundamental Limitation: Stateless Text Generation

<p align="center">
  <img src="/images/blog/llm-token-in-out.png" alt="LLM Token In Token Out" style="max-width:100%;">
</p>

An LLM has no memory across calls. It cannot browse the web. It cannot run code. It cannot read your database. Every request starts from scratch — the model reads a prompt, produces a completion, and forgets everything.

This is fine for answering "what is the capital of France?" It is not fine for "analyze our sales data from last quarter, identify trends, and update the dashboard."

The gap between what an LLM *can* do (generate text) and what we *want* it to do (accomplish tasks) is where all the interesting engineering lives.

## RAG: Giving the Model Knowledge It Doesn't Have

<p align="center">
  <img src="/images/blog/rag-pipeline.png" alt="RAG Pipeline" style="max-width:100%;">
</p>

The first problem people run into: **the model doesn't know about your data.** Its training has a cutoff date. It has never seen your internal docs, your codebase, or your Confluence pages.

The naive solution — fine-tuning — is expensive, slow, and goes stale. Retrieval-Augmented Generation (RAG) takes a different approach: instead of baking knowledge into the model's weights, you **retrieve relevant context at query time** and inject it into the prompt.

The pipeline looks like this:

1. **Index** your documents into a vector store (embeddings)
2. **Retrieve** the top-k most relevant chunks for a given query
3. **Augment** the prompt with those chunks
4. **Generate** a response grounded in real data

RAG solves the knowledge problem, but it doesn't solve the action problem. The model can now *talk about* your data — but it still can't *do* anything with it.

## Tool Use: From Talking to Doing

The next logical step: let the model call functions. Instead of only generating text, the model outputs structured tool calls — "search the database," "send an email," "create a JIRA ticket" — and a runtime executes them.

This is a significant shift. The model goes from being a text generator to being a **decision-maker** that can interact with external systems. But tool use alone introduces new problems:

- **Who defines the tools?** Someone has to write the function signatures, descriptions, and implementations.
- **How does the model discover them?** You can't stuff hundreds of tool definitions into every prompt.
- **How do tools from different systems talk to each other?** A Slack tool and a GitHub tool might be built by different teams with different interfaces.

## MCP: A Standard Interface for Tools

This is exactly the problem the Model Context Protocol (MCP) addresses. MCP provides a **standardized protocol** for connecting LLMs to external tools and data sources — think of it as a USB-C port for AI.

Before MCP, every integration was bespoke. Want to connect your model to GitHub? Write a custom adapter. Slack? Another adapter. Your internal API? Yet another one. Each with its own authentication, error handling, and schema format.

MCP defines a common interface:

- **Tools** — actions the model can take (e.g., `search_files`, `create_issue`)
- **Resources** — data the model can read (e.g., file contents, database rows)
- **Prompts** — reusable prompt templates

With MCP, a tool provider implements the protocol once, and any MCP-compatible agent can use it. The integration cost drops from O(M×N) to O(M+N).

## Skills: Reusable Task Patterns

Even with tools available, you often find the model repeating the same multi-step patterns. "Read the file, understand the context, make the edit, run the tests" — this sequence appears in almost every code modification task.

**Skills** are pre-defined task patterns that encode these workflows. Instead of hoping the model figures out the right sequence of tool calls every time, a skill provides a structured recipe:

- What context to gather first
- What tools to use and in what order
- What constraints to follow
- How to validate the result

Skills sit between the raw tool layer and the high-level user intent. They're not rigid scripts — the model still makes decisions within the skill — but they provide enough structure to make outcomes reliable and repeatable.

## The Agent: Putting It All Together

<p align="center">
  <img src="/images/blog/agent-loop.png" alt="Agent Loop" style="max-width:80%;">
</p>

Now we have all the pieces: an LLM that can reason, RAG for knowledge, tools for action, MCP for standardized integration, and skills for reliable patterns. An **agent** is the runtime that orchestrates all of this in a loop:

1. **Observe** — receive a task or user message
2. **Think** — reason about what to do next
3. **Act** — call a tool, retrieve context, or invoke a skill
4. **Reflect** — evaluate the result and decide whether to continue

This observe-think-act loop is what transforms a stateless text generator into something that can actually accomplish multi-step tasks. The agent maintains state across iterations, recovers from errors, and makes progress toward a goal.

## Harness Engineering: The Invisible Layer

Here's what most blog posts leave out: **the agent doesn't run in a vacuum.** There is a layer of engineering between the model and the user that makes everything work — the **harness**.

The harness is the system that:

- **Manages the conversation loop** — feeding context back to the model, compressing history when the context window fills up
- **Enforces permissions** — deciding which tool calls to auto-approve vs. prompt the user
- **Handles errors gracefully** — retrying failed tool calls, catching malformed outputs, preventing infinite loops
- **Provides hooks** — letting users inject custom behaviors at specific points (before commit, after file edit, etc.)
- **Controls cost and latency** — deciding when to use a more capable model vs. a faster one

This is where the real engineering complexity lives. The model is the brain, but the harness is the nervous system. A brilliant model with a poor harness produces unreliable results. A well-engineered harness can make even a less capable model useful.

Think of it this way: when you use an AI coding assistant, the quality of your experience depends less on whether the underlying model scores 92% vs. 95% on some benchmark, and more on whether the harness correctly manages file reads, prevents destructive actions, and maintains context across a long session.

## The Full Stack

<p align="center">
  <img src="/images/blog/full-stack.png" alt="Full Stack" style="max-width:90%;">
</p>

Zooming out, here is the logical stack from bottom to top:

| Layer | What It Does | Why It Exists |
|-------|-------------|---------------|
| **LLM** | Generates text, reasons about problems | Core intelligence |
| **RAG** | Retrieves relevant context | The model can't know everything |
| **Tools** | Executes actions in the real world | Text generation alone isn't enough |
| **MCP** | Standardizes tool interfaces | Bespoke integrations don't scale |
| **Skills** | Encodes reusable task patterns | Reliability over ad-hoc reasoning |
| **Agent** | Orchestrates the loop | Multi-step tasks need state |
| **Harness** | Manages the runtime | Production systems need guardrails |

Each layer exists because the layer below it is not sufficient on its own. That's the key insight — this isn't a collection of trendy acronyms. It's a logical progression of engineering solutions, each addressing a real limitation of what came before.

## What's Next

We are still early. Current agents are good at well-scoped tasks but struggle with truly open-ended, long-horizon work. The harness layer is where most of the innovation is happening now — better memory, smarter context management, more sophisticated planning.

But the direction is clear: we're moving from "ask the model a question" to "give the agent a goal." The gap between those two is filled with engineering, and understanding each layer is the first step to building systems that actually work.

---

*Last updated: March 25, 2026*
