<div align="center">

<img src="./assets/pigs-mini-agent-banner.svg" width="100%" alt="pigs-mini-agent —— 用最小 Rust 实现理解 AI Agent 的核心循环" />

<br/>

<a href="./README.md"><img src="https://img.shields.io/badge/English-README.md-2563EB?style=for-the-badge" alt="English README"/></a>
<a href="https://www.rust-lang.org/"><img src="https://img.shields.io/badge/Rust_2021-000000?style=for-the-badge&logo=rust&logoColor=white" alt="Rust 2021"/></a>
<img src="https://img.shields.io/badge/version-0.1.0-7C3AED?style=for-the-badge" alt="version 0.1.0"/>
<img src="https://img.shields.io/badge/license-MIT-059669?style=for-the-badge" alt="MIT License"/>

<br/><br/>

**一个小型、自包含、教学导向的 Rust 通用 AI Agent 实现。**

</div>

---

## 这是什么？

pigs-mini-agent 想回答一个很具体的问题：

> **Claude Code、Cursor、Codex 这类 AI 编程 Agent，底层最核心的东西到底是什么？**

把复杂工程能力一层层剥掉以后，Agent 的核心其实很小：

~~~text
用户消息
   │
   ▼
对话历史
   │
   ▼
调用 LLM
   │
   ├── 只返回文本 ─────────────► 回复用户
   │
   └── 返回工具调用
           │
           ▼
        执行工具
           │
           ▼
      工具结果加入历史
           │
           └──────────────► 再次调用 LLM
~~~

真正负责推理和决策的是 **LLM**。

Agent 运行时主要提供的是：

**历史 + 工具 + 执行 + 循环**

这个项目的目标，就是把这条核心链路完整露出来，而不是藏在一个庞大的 Agent 框架后面。

---

## 为什么做这个项目？

<table>
<tr>
<td width="33%" valign="top">

### 🧠 看懂 Agent 循环
直接阅读一个完整但足够小的 Agent 控制流程。

</td>
<td width="33%" valign="top">

### 🛠️ 看懂工具调用
理解工具 schema、tool call、实际执行和 tool result 是怎样串起来的。

</td>
<td width="33%" valign="top">

### 🦀 顺便学习 Rust
在真实 Agent 示例里接触 async Rust、trait、enum、类型化错误和 HTTP API。

</td>
</tr>
</table>

项目刻意保持**自包含**：不需要在多个 crate 之间跳来跳去，就能跟完“用户输入 → LLM → 工具 → 再次 LLM → 最终回复”的完整路径。

---

## 核心架构

~~~text
┌─────────────────────────────────────────────────────────────┐
│                       Agent::chat()                         │
│                                                             │
│  用户输入 ─► messages ─► LlmClient::chat()                 │
│                               │                             │
│                               ▼                             │
│                    OpenAI-compatible API                    │
│                               │                             │
│                      ┌────────┴────────┐                    │
│                      ▼                 ▼                    │
│                    文本回复          tool_calls             │
│                      │                 │                    │
│                      │                 ▼                    │
│                      │           ToolRegistry               │
│                      │                 │                    │
│                      │        ┌────────┼────────┐           │
│                      │        ▼        ▼        ▼           │
│                      │      bash   read/write   edit        │
│                      │        │        │        │           │
│                      │        └────────┼────────┘           │
│                      │                 ▼                    │
│                      │             工具结果                 │
│                      │                 │                    │
│                      └──── 完成 ◄──────┴── 继续循环         │
└─────────────────────────────────────────────────────────────┘
~~~

这个循环是**有界的**：默认最多执行 **50 轮**，防止模型陷入不断调用工具的死循环。

---

## 内置工具

| 工具 | 功能 | 设计重点 |
|---|---|---|
| <code>bash</code> | 执行 Shell 命令 | 给 Agent 一个通用执行原语 |
| <code>read_file</code> | 读取文件并提供行级内容 | 让模型获取本地上下文 |
| <code>write_file</code> | 创建 / 覆盖文件并自动创建父目录 | 最小文件创建能力 |
| <code>edit_file</code> | 唯一匹配文本片段后进行替换 | 避免脆弱的行号编辑 |

### 为什么 <code>edit_file</code> 不用行号？

行号对于 LLM 来说并不是一个可靠的编辑接口：

- 模型可能数错；
- 前一次修改后，后面的行号可能已经发生变化；
- 错一行有时不会立即失败，而是静默修改了错误位置。

因此这里采用：

~~~text
old_string → new_string
~~~

<code>old_string</code> 必须在文件中**唯一匹配**。

如果不唯一，模型需要加入更多上下文再试。这样失败是显式的，成功也更容易验证。

---

## 快速开始

### 1. 设置 API Key

~~~bash
export OPENAI_API_KEY="sk-xxxx"
~~~

### 2. 运行交互式示例

~~~bash
cargo run --example chat
~~~

可选配置：

~~~bash
export OPENAI_BASE_URL="https://api.openai.com/v1"
export OPENAI_MODEL="gpt-4o"
~~~

内置 LLM 客户端使用 **OpenAI-compatible Chat Completions API**，因此也可以连接 DeepSeek、Qwen、Kimi、Ollama 等兼容接口。

例如：

~~~bash
# DeepSeek
export OPENAI_BASE_URL="https://api.deepseek.com/v1"
export OPENAI_MODEL="deepseek-chat"

# Qwen
export OPENAI_BASE_URL="https://dashscope.aliyuncs.com/compatible-mode/v1"
export OPENAI_MODEL="qwen-plus"

# Ollama
export OPENAI_BASE_URL="http://localhost:11434/v1"
export OPENAI_MODEL="llama3"
~~~

---

## 最小代码

~~~rust
use pigs_mini_agent::{create_default_tools, Agent, LlmClient};

#[tokio::main]
async fn main() -> pigs_mini_agent::Result<()> {
    let llm = LlmClient::from_env()?;
    let tools = create_default_tools();
    let mut agent = Agent::new(llm, tools);

    let response = agent
        .chat("创建一个 hello.txt，内容是 Hello, Agent!")
        .await?;

    println!("{response}");
    Ok(())
}
~~~

这已经足够构造一个可以调用工具完成任务的 Agent。

---

## LLM 客户端做了什么？

内置 <code>LlmClient</code> 刻意保持简单：

- 只使用 OpenAI-compatible **Chat Completions**；
- 使用非流式响应；
- 通过标准 <code>tools</code> / <code>tool_calls</code> 字段完成工具调用；
- HTTP 超时为 120 秒；
- 对 HTTP <code>429</code> 和服务端 <code>5xx</code> 错误进行重试；
- 使用指数退避；
- 可重试请求最多额外重试 3 次。

采用非流式是一个教学设计。

流式工具调用还需要处理 SSE 事件，以及被拆成多个 delta 的工具参数 JSON。那些代码对于生产系统很有价值，但会干扰“Agent 核心循环”这个主要教学目标。

---

## 推荐阅读顺序

项目结构按“数据结构 → 工具 → LLM → Agent 核心循环”的顺序组织：

| 顺序 | 文件 | 主要内容 |
|---|---|---|
| 1 | [<code>src/message.rs</code>](./src/message.rs) | 对话历史与消息数据模型 |
| 2 | [<code>src/tool.rs</code>](./src/tool.rs) | <code>Tool</code> trait 与 <code>ToolRegistry</code> |
| 3 | [<code>src/prompt.rs</code>](./src/prompt.rs) | 系统提示词如何描述 Agent 行为与工具 |
| 4 | [<code>src/llm.rs</code>](./src/llm.rs) | 如何构造请求、调用模型并解析回复 |
| 5 | [<code>src/tools/</code>](./src/tools) | 4 个工具的实际实现 |
| 6 | [<code>src/agent.rs</code>](./src/agent.rs) | **Agent 循环——整个项目的心脏** |
| 7 | [<code>src/error.rs</code>](./src/error.rs) | 类型化错误处理 |
| 8 | [<code>src/lib.rs</code>](./src/lib.rs) | crate 对外接口与模块结构 |

---

## 教学文档

仓库里有多层教学材料：

<table>
<tr>
<td width="33%" valign="top">

### 🌐 HTML 教程
[<code>docs/html/</code>](./docs/html)

浏览器可直接阅读的教学站点，采用 Terminal 风格视觉设计。

</td>
<td width="33%" valign="top">

### 📖 Markdown 章节
[<code>docs/00-overview.md</code>](./docs/00-overview.md) → [<code>docs/07-error.md</code>](./docs/07-error.md)

按模块逐章讲解，并配合源码拆解。

</td>
<td width="33%" valign="top">

### 🦀 Rust API 文档

~~~bash
cargo doc --open
~~~

直接从源码中的详细 <code>///</code> 注释生成。

</td>
</tr>
</table>

建议从 **[docs/00-overview.md](./docs/00-overview.md)** 开始。

---

## 关键设计选择

### 自包含，而不是依赖完整 Agent 框架

教学项目最重要的是能够追踪控制流，所以这里不依赖更大的 pigs 架构。

### 源码使用大量中文注释

重点不是只告诉你“这段代码做了什么”，而是尽量解释“为什么这样设计”。

### 非流式是刻意的

生产系统中流式很有用，但理解工具型 Agent 并不需要先处理 SSE 和碎片化 tool arguments。

### 明确限制循环轮数

默认 50 轮，避免错误工具调用模式导致无限执行。

---

## 参考与启发

这个实现参考过一些 Agent / AI 编程工具的架构思路：

| 项目 | 主要借鉴点 |
|---|---|
| CoreCoder | 极简 Agent 循环、工具设计、唯一匹配编辑、提示词结构 |
| claw-code / claw-analog | Rust Agent 控制流程模式 |
| Codex | 分层架构与消息模型设计 |
| pigs | Rust 类型设计与更完整 Agent 系统的架构经验 |

本项目的目标是教学和机制拆解，并不是其中任何一个项目的完整复刻。

---

## 技术栈

<div align="center">

<img src="https://img.shields.io/badge/Rust_2021-000000?style=flat-square&logo=rust&logoColor=white" />
<img src="https://img.shields.io/badge/Tokio-异步运行时-1F6FEB?style=flat-square" />
<img src="https://img.shields.io/badge/reqwest-HTTP_客户端-0EA5E9?style=flat-square" />
<img src="https://img.shields.io/badge/serde-JSON-F59E0B?style=flat-square" />
<img src="https://img.shields.io/badge/async--trait-异步工具-8B5CF6?style=flat-square" />
<img src="https://img.shields.io/badge/thiserror-错误处理-059669?style=flat-square" />

</div>

---

## 许可证

MIT，见 [LICENSE](./LICENSE)。

英文版见 **[README.md](./README.md)**。
