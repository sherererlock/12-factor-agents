# 12-Factor Agents - 构建可靠 LLM 应用的原则

<div align="center">
<a href="https://www.apache.org/licenses/LICENSE-2.0">
        <img src="https://img.shields.io/badge/Code-Apache%202.0-blue.svg" alt="Code License: Apache 2.0"></a>
<a href="https://creativecommons.org/licenses/by-sa/4.0/">
        <img src="https://img.shields.io/badge/Content-CC%20BY--SA%204.0-lightgrey.svg" alt="Content License: CC BY-SA 4.0"></a>
<a href="https://humanlayer.dev/discord">
    <img src="https://img.shields.io/badge/chat-discord-5865F2" alt="Discord Server"></a>
<a href="https://www.youtube.com/watch?v=8kMaTybvDUw">
    <img src="https://img.shields.io/badge/aidotengineer-conf_talk_(17m)-white" alt="YouTube
Deep Dive"></a>
<a href="https://www.youtube.com/watch?v=yxJDyQ8v6P0">
    <img src="https://img.shields.io/badge/youtube-deep_dive-crimson" alt="YouTube
Deep Dive"></a>
    
</div>

<p></p>

*秉承 [12 Factor Apps](https://12factor.net/) 的精神*。*本项目源码公开在 https://github.com/humanlayer/12-factor-agents，欢迎您的反馈和贡献。让我们一起探索！*

> [!TIP]
> 错过了 AI Engineer World's Fair？[在这里观看演讲](https://www.youtube.com/watch?v=8kMaTybvDUw)
>
> 寻找上下文工程相关内容？[直接跳转到 Factor 3](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md)
>
> 想为 `npx/uvx create-12-factor-agent` 做贡献？查看[讨论帖](https://github.com/humanlayer/12-factor-agents/discussions/61)


<img referrerpolicy="no-referrer-when-downgrade" src="https://static.scarf.sh/a.png?x-pxid=2acad99a-c2d9-48df-86f5-9ca8061b7bf9" />

<a href="#visual-nav"><img width="907" alt="Screenshot 2025-04-03 at 2 49 07 PM" src="https://github.com/user-attachments/assets/23286ad8-7bef-4902-b371-88ff6a22e998" /></a>


你好，我是 Dex。我一直在[研究](https://youtu.be/8bIHcttkOTE) [AI Agent](https://theouterloop.substack.com)，已经[有很长一段时间](https://humanlayer.dev)了。


**我尝试过市面上所有的 Agent 框架**，从即插即用的 crew/langchain 系列，到"极简主义"的 smolagents，再到"生产级"的 langraph、griptape 等等。

**我与许多非常优秀的创始人交流过**，无论是在 YC 内外，他们都在用 AI 构建令人印象深刻的产品。他们中的大多数人都是自己搭建技术栈。我在面向客户的生产级 Agent 中很少看到框架的身影。

**令我惊讶的是**，市面上大多数自称"AI Agent"的产品并没有那么具有自主性。它们大多是确定性代码，只在恰到好处的地方穿插 LLM 调用，让体验变得真正神奇。

Agent，至少优秀的那些，并不遵循["给你一个 prompt，给你一堆工具，循环执行直到达成目标"](https://www.anthropic.com/engineering/building-effective-agents#agents)这种模式。相反，它们主要由普通软件构成。

所以，我开始思考这个问题：

> ### **我们可以用哪些原则来构建足以交付给生产环境客户的 LLM 驱动软件？**

欢迎来到 12-factor agents。正如自 Daley 以来的每一任芝加哥市长都在城市各大机场反复宣传的那样，我们很高兴你的到来。

*特别感谢 [@iantbutler01](https://github.com/iantbutler01)、[@tnm](https://github.com/tnm)、[@hellovai](https://www.github.com/hellovai)、[@stantonk](https://www.github.com/stantonk)、[@balanceiskey](https://www.github.com/balanceiskey)、[@AdjectiveAllison](https://www.github.com/AdjectiveAllison)、[@pfbyjy](https://www.github.com/pfbyjy)、[@a-churchill](https://www.github.com/a-churchill)，以及 SF MLOps 社区对本指南的早期反馈。*

## 简要概述：12 个因素

即使 LLM [继续呈指数级增长](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-10-small-focused-agents.md#what-if-llms-get-smarter)，仍有一些核心工程技术能使 LLM 驱动的软件更可靠、更易扩展、更易维护。

- [我们如何走到这里：软件简史](https://github.com/humanlayer/12-factor-agents/blob/main/content/brief-history-of-software.md)
- [Factor 1：自然语言转工具调用](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-01-natural-language-to-tool-calls.md)
- [Factor 2：掌控你的 Prompt](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-02-own-your-prompts.md)
- [Factor 3：掌控你的上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md)
- [Factor 4：工具就是结构化输出](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-04-tools-are-structured-outputs.md)
- [Factor 5：统一执行状态与业务状态](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-05-unify-execution-state.md)
- [Factor 6：通过简单 API 实现启动/暂停/恢复](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-06-launch-pause-resume.md)
- [Factor 7：通过工具调用联系人类](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-07-contact-humans-with-tools.md)
- [Factor 8：掌控你的控制流](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md)
- [Factor 9：将错误压缩进上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md)
- [Factor 10：小型、专注的 Agent](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-10-small-focused-agents.md)
- [Factor 11：从任何地方触发，在用户所在之处与他们会合](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-11-trigger-from-anywhere.md)
- [Factor 12：让你的 Agent 成为无状态归约器](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-12-stateless-reducer.md)

### 可视化导航

|    |    |    |
|----|----|-----|
|[![factor 1](https://github.com/humanlayer/12-factor-agents/blob/main/img/110-natural-language-tool-calls.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-01-natural-language-to-tool-calls.md) | [![factor 2](https://github.com/humanlayer/12-factor-agents/blob/main/img/120-own-your-prompts.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-02-own-your-prompts.md) | [![factor 3](https://github.com/humanlayer/12-factor-agents/blob/main/img/130-own-your-context-building.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md) |
|[![factor 4](https://github.com/humanlayer/12-factor-agents/blob/main/img/140-tools-are-just-structured-outputs.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-04-tools-are-structured-outputs.md) | [![factor 5](https://github.com/humanlayer/12-factor-agents/blob/main/img/150-unify-state.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-05-unify-execution-state.md) | [![factor 6](https://github.com/humanlayer/12-factor-agents/blob/main/img/160-pause-resume-with-simple-apis.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-06-launch-pause-resume.md) |
| [![factor 7](https://github.com/humanlayer/12-factor-agents/blob/main/img/170-contact-humans-with-tools.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-07-contact-humans-with-tools.md) | [![factor 8](https://github.com/humanlayer/12-factor-agents/blob/main/img/180-control-flow.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md) | [![factor 9](https://github.com/humanlayer/12-factor-agents/blob/main/img/190-factor-9-errors-static.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md) |
| [![factor 10](https://github.com/humanlayer/12-factor-agents/blob/main/img/1a0-small-focused-agents.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-10-small-focused-agents.md) | [![factor 11](https://github.com/humanlayer/12-factor-agents/blob/main/img/1b0-trigger-from-anywhere.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-11-trigger-from-anywhere.md) | [![factor 12](https://github.com/humanlayer/12-factor-agents/blob/main/img/1c0-stateless-reducer.png)](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-12-stateless-reducer.md) |

## 我们如何走到这里

想深入了解我的 Agent 之旅以及是什么引领我们走到这里，请查看[软件简史](https://github.com/humanlayer/12-factor-agents/blob/main/content/brief-history-of-software.md)——以下是简要概述：

### Agent 的愿景

我们将大量讨论有向图（DG）及其无环的朋友们——DAG。首先我想指出的是……嗯……软件本身就是有向图。我们过去用流程图来表示程序是有原因的。

![010-software-dag](https://github.com/humanlayer/12-factor-agents/blob/main/img/010-software-dag.png)

### 从代码到 DAG

大约 20 年前，我们开始看到 DAG 编排器变得流行。我们说的是经典产品如 [Airflow](https://airflow.apache.org/)、[Prefect](https://www.prefect.io/)，以及一些后来者如 [dagster](https://dagster.io/)、[inngest](https://www.inngest.com/)、[windmill](https://www.windmill.dev/)。它们遵循相同的图模式，并额外提供了可观测性、模块化、重试、管理等能力。

![015-dag-orchestrators](https://github.com/humanlayer/12-factor-agents/blob/main/img/015-dag-orchestrators.png)

### Agent 的愿景

我不是[第一个这么说的人](https://youtu.be/Dc99-zTMyMg?si=bcT0hIwWij2mR-40&t=73)，但当我开始学习 Agent 时，最大的收获是你可以把 DAG 扔掉了。不再需要软件工程师编写每个步骤和边界情况，你可以给 Agent 一个目标和一组状态转换：

![025-agent-dag](https://github.com/humanlayer/12-factor-agents/blob/main/img/025-agent-dag.png)

然后让 LLM 实时做出决策来确定路径

![026-agent-dag-lines](https://github.com/humanlayer/12-factor-agents/blob/main/img/026-agent-dag-lines.png)

这里的愿景是你写更少的代码，只需给 LLM 图的"边"，让它自己确定节点。你可以从错误中恢复，可以写更少的代码，而且你可能会发现 LLM 能找到解决问题的新方法。


### Agent 即循环

正如我们稍后将看到的，事实证明这并不完全奏效。

让我们再深入一层——Agent 由一个包含 3 个步骤的循环组成：

1. LLM 确定工作流中的下一步，输出结构化 JSON（"工具调用"）
2. 确定性代码执行工具调用
3. 结果被追加到上下文窗口
4. 重复，直到下一步被确定为"完成"

```python
initial_event = {"message": "..."}
context = [initial_event]
while True:
  next_step = await llm.determine_next_step(context)
  context.append(next_step)

  if (next_step.intent === "done"):
    return next_step.final_answer

  result = await execute_step(next_step)
  context.append(result)
```

我们的初始上下文只是起始事件（可能是用户消息、定时任务触发、webhook 等），然后我们让 LLM 选择下一步（工具）或确定任务已完成。

这是一个多步骤示例：

[![027-agent-loop-animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-animation.gif)](https://github.com/user-attachments/assets/3beb0966-fdb1-4c12-a47f-ed4e8240f8fd)

<details>
<summary><a href="https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-animation.gif">GIF 版本</a></summary>

![027-agent-loop-animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-animation.gif)

</details>

## 为什么是 12-factor agents？

归根结底，这种方法的效果并不如我们所期望的那样好。

在构建 HumanLayer 的过程中，我与至少 100 位 SaaS 构建者（主要是技术创始人）交流过，他们都希望让现有产品更具自主性。这段旅程通常是这样的：

1. 决定要构建一个 Agent
2. 产品设计、UX 规划、确定要解决的问题
3. 希望快速推进，于是选用某个 $FRAMEWORK 并*开始构建*
4. 达到 70-80% 的质量标准
5. 意识到 80% 对大多数面向客户的功能来说还不够好
6. 意识到突破 80% 需要逆向工程框架、prompt、流程等
7. 从头开始

<details>
<summary>随机免责声明</**免责声明**：我不确定说这些话的最佳时机，但这里似乎和任何地方一样合适：**这绝不是在批评市面上的众多框架，或在这些框架上工作的非常聪明的人**。他们成就了令人难以置信的事情，并加速了 AI 生态系统的发展。

我希望这篇文章的一个成果是，Agent 框架构建者能够从我和其他人的经历中学习，让框架变得更好。

特别是对于那些希望快速推进但需要深度控制的构建者。

**免责声明 2**：我不会讨论 MCP。我相信你能看出它在哪里适用。

**免责声明 3**：我主要使用 TypeScript，[有各种原因](https://www.linkedin.com/posts/dexterihorthy_llms-typescript-aiagents-activity-7290858296679313408-Lh9e?utm_source=share&utm_medium=member_desktop&rcm=ACoAAA4oHTkByAiD-wZjnGsMBUL_JT6nyyhOh30)，但所有这些内容在 Python 或任何你喜欢的语言中都适用。


无论如何，回到正题...

</details>

### 优秀 LLM 应用的设计模式

在研究了数百个 AI 库并与数十位创始人合作之后，我的直觉是：

1. 有一些核心要素能让 Agent 变得优秀
2. 全力投入一个框架并进行本质上是绿地项目的重写可能会适得其反
3. 有一些核心原则能让 Agent 变得优秀，如果你引入一个框架，你会获得其中的大部分/全部
4. 但是，我所见过的让构建者将高质量 AI 软件交付给客户的最快方式，是从 Agent 构建中提取小型、模块化的概念，并将它们融入现有产品
5. 这些来自 Agent 的模块化概念可以由大多数熟练的软件工程师定义和应用，即使他们没有 AI 背景

> #### 我所见过的让构建者将优质 AI 软件交付给客户的最快方式，是从 Agent 构建中提取小型、模块化的概念，并将它们融入现有产品


## 12 个因素（再次列出）


- [我们如何走到这里：软件简史](https://github.com/humanlayer/12-factor-agents/blob/main/content/brief-history-of-software.md)
- [Factor 1：自然语言转工具调用](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-01-natural-language-to-tool-calls.md)
- [Factor 2：掌控你的 Prompt](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-02-own-your-prompts.md)
- [Factor 3：掌控你的上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md)
- [Factor 4：工具就是结构化输出](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-04-tools-are-structured-outputs.md)
- [Factor 5：统一执行状态与业务状态](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-05-unify-execution-state.md)
- [Factor 6：通过简单 API 实现启动/暂停/恢复](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-06-launch-pause-resume.md)
- [Factor 7：通过工具调用联系人类](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-07-contact-humans-with-tools.md)
- [Factor 8：掌控你的控制流](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md)
- [Factor 9：将错误压缩进上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md)
- [Factor 10：小型、专注的 Agent](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-10-small-focused-agents.md)
- [Factor 11：从任何地方触发，在用户所在之处与他们会合](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-11-trigger-from-anywhere.md)
- [Factor 12：让你的 Agent 成为无状态归约器](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-12-stateless-reducer.md)

## 荣誉提及 / 其他建议

- [Factor 13：预取所有可能需要的上下文](https://github.com/humanlayer/12-factor-agents/blob/main/content/appendix-13-pre-fetch.md)

## 相关资源

- 在[这里](https://github.com/humanlayer/12-factor-agents)为本指南做贡献
- [我在 2025 年 3 月的 Tool Use 播客中讨论了其中的很多内容](https://youtu.be/8bIHcttkOTE)
- 我在 [The Outer Loop](https://theouterloop.substack.com) 上写了一些相关内容
- 我与 [@hellovai](https://github.com/hellovai) 一起做关于[最大化 LLM 性能的网络研讨会](https://github.com/hellovai/ai-that-works/tree/main)
- 我们在 [got-agents/agents](https://github.com/got-agents/agents) 下使用这种方法构建开源 Agent
- 我们忽略了自己所有的建议，构建了一个[在 Kubernetes 中运行分布式 Agent 的框架](https://github.com/humanlayer/kubechain)
- 本指南中的其他链接：
  - [12 Factor Apps](https://12factor.net)
  - [Building Effective Agents (Anthropic)](https://www.anthropic.com/engineering/building-effective-agents#agents)
  - [Prompts are Functions](https://thedataexchange.media/baml-revolution-in-ai-engineering/ )
  - [Library patterns: Why frameworks are evil](https://tomasp.net/blog/2015/library-frameworks/)
  - [The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction)
  - [Mailcrew Agent](https://github.com/dexhorthy/mailcrew)
  - [Mailcrew Demo Video](https://www.youtube.com/watch?v=f_cKnoPC_Oo)
  - [Chainlit Demo](https://x.com/chainlit_io/status/1858613325921480922)
  - [TypeScript for LLMs](https://www.linkedin.com/posts/dexterihorthy_llms-typescript-aiagents-activity-7290858296679313408-Lh9e)
  - [Schema Aligned Parsing](https://www.boundaryml.com/blog/schema-aligned-parsing)
  - [Function Calling vs Structured Outputs vs JSON Mode](https://www.vellum.ai/blog/when-should-i-use-function-calling-structured-outputs-or-json-mode)
  - [BAML on GitHub](https://github.com/boundaryml/baml)
  - [OpenAI JSON vs Function Calling](https://docs.llamaindex.ai/en/stable/examples/llm/openai_json_vs_function_calling/)
  - [Outer Loop Agents](https://theouterloop.substack.com/p/openais-realtime-api-is-a-step-towards)
  - [Airflow](https://airflow.apache.org/)
  - [Prefect](https://www.prefect.io/)
  - [Dagster](https://dagster.io/)
  - [Inngest](https://www.inngest.com/)
  - [Windmill](https://www.windmill.dev/)
  - [The AI Agent Index (MIT)](https://aiagentindex.mit.edu/)
  - [NotebookLM on Finding Model Capability Boundaries](https://open.substack.com/pub/swyx/p/notebooklm?selection=08e1187c-cfee-4c63-93c9-71216640a5f8)

## 贡献者

感谢所有为 12-factor agents 做出贡献的人！

[<img src="https://avatars.githubusercontent.com/u/3730605?v=4&s=80" width="80px" alt="dexhorthy" />](https://github.com/dexhorthy) [<img src="https://avatars.githubusercontent.com/u/50557586?v=4&s=80" width="80px" alt="Sypherd" />](https://github.com/Sypherd) [<img src="https://avatars.githubusercontent.com/u/66259401?v=4&s=80" width="80px" alt="tofaramususa" />](https://github.com/tofaramususa) [<img src="https://avatars.githubusercontent.com/u/18105223?v=4&s=80" width="80px" alt="a-churchill" />](https://github.com/a-churchill) [<img src="https://avatars.githubusercontent.com/u/4084885?v=4&s=80" width="80px" alt="Elijas" />](https://github.com/Elijas) [<img src="https://avatars.githubusercontent.com/u/39267118?v=4&s=80" width="80px" alt="hugolmn" />](https://github.com/hugolmn) [<img src="https://avatars.githubusercontent.com/u/1882972?v=4&s=80" width="80px" alt="jeremypeters" />](https://github.com/jeremypeters)

[<img src="https://avatars.githubusercontent.com/u/380402?v=4&s=80" width="80px" alt="kndl" />](https://github.com/kndl) [<img src="https://avatars.githubusercontent.com/u/16674643?v=4&s=80" width="80px" alt="maciejkos" />](https://github.com/maciejkos) [<img src="https://avatars.githubusercontent.com/u/85041180?v=4&s=80" width="80px" alt="pfbyjy" />](https://github.com/pfbyjy) [<img src="https://avatars.githubusercontent.com/u/36044389?v=4&s=80" width="80px" alt="0xRaduan" />](https://github.com/0xRaduan) [<img src="https://avatars.githubusercontent.com/u/7169731?v=4&s=80" width="80px" alt="zyuanlim" />](https://github.com/zyuanlim) [<img src="https://avatars.githubusercontent.com/u/15862501?v=4&s=80" width="80px" alt="lombardo-chcg" />](https://github.com/lombardo-chcg) [<img src="https://avatars.githubusercontent.com/u/160066852?v=4&s=80" width="80px" alt="sahanatvessel" />](https://github.com/sahanatvessel)
 
## 许可证

所有内容和图片均采用 <a href="https://creativecommons.org/licenses/by-sa/4.0/">CC BY-SA 4.0 许可证</a> 授权

代码采用 <a href="https://www.apache.org/licenses/LICENSE-2.0">Apache 2.0 许可证</a> 授权

