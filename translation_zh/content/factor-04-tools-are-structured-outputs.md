[← 返回 README](https://github.com/humanlayer/12-factor-agents/blob/main/README.md)

### 4. 工具就是结构化输出

工具不需要很复杂。从本质上讲，它们只是 LLM 输出的结构化数据，用于触发确定性代码。

![140-tools-are-just-structured-outputs](https://github.com/humanlayer/12-factor-agents/blob/main/img/140-tools-are-just-structured-outputs.png)

例如，假设你有两个工具 `CreateIssue` 和 `SearchIssues`。让 LLM "使用多个工具之一"，本质上就是让它输出我们可以解析为代表这些工具的对象的 JSON。

```python

class Issue:
  title: str
  description: str
  team_id: str
  assignee_id: str

class CreateIssue:
  intent: "create_issue"
  issue: Issue

class SearchIssues:
  intent: "search_issues"
  query: str
  what_youre_looking_for: str
```

这个模式很简单：
1. LLM 输出结构化 JSON
3. 确定性代码执行相应的操作（如调用外部 API）
4. 结果被捕获并反馈到上下文中

这在 LLM 的决策和你的应用程序操作之间创建了清晰的分离。LLM 决定做什么，但你的代码控制如何执行。仅仅因为 LLM "调用了一个工具"，并不意味着你必须每次都以相同的方式执行特定的对应函数。

如果你还记得我们上面的 switch 语句

```python
if nextStep.intent == 'create_payment_link':
    stripe.paymentlinks.create(nextStep.parameters)
    return # 或者你想要的任何操作，见下文
elif nextStep.intent == 'wait_for_a_while': 
    # 做一些 monadic 之类的事情
else: #... 模型没有调用我们已知的工具
    # 执行其他操作
```

**注意**：关于"纯提示词"、"工具调用"和"JSON 模式"的优劣以及各自的性能权衡，已经有很多讨论。我们很快会链接一些相关资源，但在这里不再赘述。参见 [Prompting vs JSON Mode vs Function Calling vs Constrained Generation vs SAP](https://www.boundaryml.com/blog/schema-aligned-parsing)、[When should I use function calling, structured outputs, or JSON mode?](https://www.vellum.ai/blog/when-should-i-use-function-calling-structured-outputs-or-json-mode#:~:text=We%20don%27t%20recommend%20using%20JSON,always%20use%20Structured%20Outputs%20instead) 和 [OpenAI JSON vs Function Calling](https://docs.llamaindex.ai/en/stable/examples/llm/openai_json_vs_function_calling/)。

"下一步"可能不像"运行一个纯函数并返回结果"那样原子化。当你将"工具调用"视为模型输出描述确定性代码应该做什么的 JSON 时，你就解锁了大量的灵活性。将此与[因素 8 掌控你的控制流](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-08-own-your-control-flow.md)结合起来。

[← 掌控你的上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-03-own-your-context-window.md) | [统一执行状态 →](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-05-unify-execution-state.md)
