[← 返回 README](https://github.com/humanlayer/12-factor-agents/blob/main/README.md)

## 详细版本：我们是如何走到这里的

### 你不必听我的

无论你是 Agent 新手还是像我一样的老顽固，我都会试图说服你：抛弃你对 AI Agent 的大部分认知，退后一步，从第一性原理重新思考。（如果你没注意到几周前 OpenAI 的 responses 发布，剧透一下：把更多 Agent 逻辑推到 API 后面并不是正确方向）


## Agent 就是软件，以及软件的简史

让我们来聊聊我们是如何走到这里的

### 60 年前

我们将大量讨论有向图（DG）及其无环的朋友们——DAG。首先要指出的是……嗯……软件本身就是有向图。这就是为什么我们过去用流程图来表示程序。

![010-software-dag](https://github.com/humanlayer/12-factor-agents/blob/main/img/010-software-dag.png)

### 20 年前

大约 20 年前，我们开始看到 DAG 编排器变得流行起来。我们说的是经典工具，如 [Airflow](https://airflow.apache.org/)、[Prefect](https://www.prefect.io/)，以及一些前辈和一些较新的工具（如 [dagster](https://dagster.io/)、[inggest](https://www.inngest.com/)、[windmill](https://www.windmill.dev/)）。它们遵循相同的图模式，并额外提供了可观测性、模块化、重试、管理等能力。

![015-dag-orchestrators](https://github.com/humanlayer/12-factor-agents/blob/main/img/015-dag-orchestrators.png)

### 10-15 年前

当 ML 模型开始变得足够好用时，我们开始看到 DAG 中融入了 ML 模型。你可能会想到这样的步骤："将此列中的文本摘要为新列"或"按严重程度或情感对支持问题进行分类"。

![020-dags-with-ml](https://github.com/humanlayer/12-factor-agents/blob/main/img/020-dags-with-ml.png)

但归根结底，它大部分还是那些经典好用的确定性软件。

### Agent 的愿景

我并不是[第一个这么说的人](https://youtu.be/Dc99-zTMyMg?si=bcT0hIwWij2mR-40&t=73)，但我开始学习 Agent 时最大的收获是：你可以把 DAG 扔掉了。不再需要软件工程师编写每个步骤和边界条件的代码，你可以给 Agent 一个目标和一组转换：

![025-agent-dag](https://github.com/humanlayer/12-factor-agents/blob/main/img/025-agent-dag.png)

让 LLM 实时做出决策来确定路径

![026-agent-dag-lines](https://github.com/humanlayer/12-factor-agents/blob/main/img/026-agent-dag-lines.png)

这里的愿景是你写更少的软件，只需给 LLM 图的"边"，让它自己确定节点。你可以从错误中恢复，可以写更少的代码，而且你可能会发现 LLM 能找到问题的新颖解决方案。

### 作为循环的 Agent

换一种说法，你有一个由 3 个步骤组成的循环：

1. LLM 确定工作流中的下一步，输出结构化的 JSON（"工具调用"）
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

我们初始的上下文只是起始事件（可能是用户消息、可能是 cron 触发、可能是 webhook 等），然后我们要求 LLM 选择下一步（工具）或判断任务是否完成。

这是一个多步骤示例：

[![027-agent-loop-animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-animation.gif)](https://github.com/user-attachments/assets/3beb0966-fdb1-4c12-a47f-ed4e8240f8fd)

<details>
<summary><a href="https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-animation.gif">GIF 版本</a></summary>

![027-agent-loop-animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-animation.gif)

</details>

生成的"物化"DAG 看起来像这样：

![027-agent-loop-dag](https://github.com/humanlayer/12-factor-agents/blob/main/img/027-agent-loop-dag.png)

### 这种"循环直到解决"模式的问题

这种模式最大的问题：

- 当上下文窗口过长时，Agent 会迷失方向——它们会反复尝试同一个失败的方法，陷入死循环
- 字面上就这一个问题，但这就足以让这种方法举步维艰

即使你没有亲手编写过 Agent，你可能也在使用 Agent 编码工具时遇到过这个长上下文问题。它们过一段时间就会迷失方向，你需要开启新的对话。

我甚至可以提出一个我经常听到的观点，你可能也已经形成了自己的直觉：

> ### **即使模型支持越来越长的上下文窗口，使用小型、专注的提示和上下文总是能获得更好的结果**

与我交谈过的大多数构建者在意识到超过 10-20 轮对话就会变成 LLM 无法恢复的大混乱后，都**把"工具调用循环"的想法抛到了一边**。即使 Agent 在 90% 的情况下都是正确的，这也远未达到"足以交付给客户使用"的标准。你能想象一个在 10% 的页面加载时崩溃的 Web 应用吗？

**2025-06-09 更新** - 我非常喜欢 [@swyx](https://x.com/swyx/status/1932125643384455237) 的表述：

<a href="https://x.com/swyx/status/1932125643384455237"><img width="593" alt="Screenshot 2025-07-02 at 11 50 50 AM" src="https://github.com/user-attachments/assets/c7d94042-e4b9-4b87-87fd-55c7ff94bb3b" /></a>

### 真正有效的方法——微 Agent

我在实际中**确实**经常见到的一种做法是：采用 Agent 模式，将其融入更广泛的确定性 DAG 中。

![micro-agent-dag](https://github.com/humanlayer/12-factor-agents/blob/main/img/028-micro-agent-dag.png)

你可能会问——"这种情况下为什么还要用 Agent？"——我们很快就会讨论到，但基本上，让语言模型管理范围明确的任务集，可以轻松整合实时的人类反馈，将其转化为工作流步骤，而不会陷入上下文错误循环（[因子 1](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-01-natural-language-to-tool-calls.md)、[因子 3](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md)、[因子 7](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-07-contact-humans-with-tools.md)）。

> #### 让语言模型管理范围明确的任务集，可以轻松整合实时的人类反馈……而不会陷入上下文错误循环

### 一个真实的微 Agent

这里有一个例子，展示确定性代码如何运行一个负责处理部署中人类参与步骤的微 Agent。

![029-deploybot-high-level](https://github.com/humanlayer/12-factor-agents/blob/main/img/029-deploybot-high-level.png)

* **人类** 将 PR 合并到 GitHub main 分支
* **确定性代码** 部署到预发布环境
* **确定性代码** 对预发布环境运行端到端（e2e）测试
* **确定性代码** 交给 Agent 处理生产部署，初始上下文为："将 SHA 4af9ec0 部署到生产环境"
* **Agent** 调用 `deploy_frontend_to_prod(4af9ec0)`
* **确定性代码** 请求人类批准此操作
* **人类** 拒绝该操作并反馈"你能先部署后端吗？"
* **Agent** 调用 `deploy_backend_to_prod(4af9ec0)`
* **确定性代码** 请求人类批准此操作
* **人类** 批准该操作
* **确定性代码** 执行后端部署
* **Agent** 调用 `deploy_frontend_to_prod(4af9ec0)`
* **确定性代码** 请求人类批准此操作
* **人类** 批准该操作
* **确定性代码** 执行前端部署
* **Agent** 判断任务已成功完成，结束！
* **确定性代码** 对生产环境运行端到端测试
* **确定性代码** 任务完成，或交给回滚 Agent 审查失败情况并可能进行回滚

[![033-deploybot-animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/033-deploybot.gif)](https://github.com/user-attachments/assets/deb356e9-0198-45c2-9767-231cb569ae13)

<details>
<summary><a href="https://github.com/humanlayer/12-factor-agents/blob/main/img/033-deploybot.gif">GIF 版本</a></summary>

![033-deploybot-animation](https://github.com/humanlayer/12-factor-agents/blob/main/img/033-deploybot.gif)

</details>

这个例子基于一个真实的[开源 Agent，我们在 Humanlayer 用来管理部署](https://github.com/got-agents/agents/tree/main/deploybot-ts)——这是我上周与它的真实对话：

![035-deploybot-conversation](https://github.com/humanlayer/12-factor-agents/blob/main/img/035-deploybot-conversation.png)


我们没有给这个 Agent 大量的工具或任务。LLM 的主要价值在于解析人类的纯文本反馈并提出更新的行动方案。我们尽可能隔离任务和上下文，让 LLM 专注于一个小型的、5-10 步的工作流。

这里是另一个[更经典的支持/聊天机器人演示](https://x.com/chainlit_io/status/1858613325921480922)。

### 那么 Agent 到底是什么？

- **提示（Prompt）** - 告诉 LLM 如何行为，以及它有哪些可用的"工具"。提示的输出是一个 JSON 对象，描述工作流中的下一步（"工具调用"或"函数调用"）（[因子 2](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-02-own-your-prompts.md)）
- **switch 语句** - 根据 LLM 返回的 JSON，决定如何处理它（[因子 8](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md) 的一部分）
- **累积上下文** - 存储已发生的步骤列表及其结果（[因子 3](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md)）
- **for 循环** - 直到 LLM 发出某种"终止"工具调用（或纯文本响应）之前，将 switch 语句的结果添加到上下文窗口，并要求 LLM 选择下一步（[因子 8](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md)）

![040-4-components](https://github.com/humanlayer/12-factor-agents/blob/main/img/040-4-components.png)

在"deploybot"示例中，我们从掌控控制流和上下文累积中获得了几个好处：

- 在我们的 **switch 语句**和 **for 循环**中，我们可以劫持控制流来暂停等待人类输入或等待长时间运行的任务完成
- 我们可以轻松地序列化**上下文**窗口以实现暂停+恢复
- 在我们的**提示**中，我们可以优化如何向 LLM 传递指令和"到目前为止发生了什么"


[第二部分](https://github.com/humanlayer/12-factor-agents/blob/main/README.md#12-factor-agents)将**形式化这些模式**，使其可以应用于为任何软件项目添加令人印象深刻的 AI 功能，而无需完全投入传统的"AI Agent"实现/定义。


[因子 1 - 自然语言到工具调用 →](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-01-natural-language-to-tool-calls.md)
