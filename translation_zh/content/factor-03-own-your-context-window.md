[← 返回 README](../README.md)

### 3. 掌控你的上下文窗口

你不一定需要使用标准的消息格式来向 LLM 传递上下文。

> #### 在任何时刻，你对智能体中 LLM 的输入都是"到目前为止发生了什么，下一步是什么"

<!-- todo syntax highlighting -->
<!-- ![130-own-your-context-building](../../img/130-own-your-context-building.png) -->

一切都是上下文工程。[LLM 是无状态函数](https://thedataexchange.media/baml-revolution-in-ai-engineering/)，它将输入转换为输出。要获得最佳输出，你需要提供最佳输入。

创建优质上下文意味着：

- 你给模型的提示词和指令
- 你检索的任何文档或外部数据（例如 RAG）
- 任何过去的状态、工具调用结果或其他历史记录
- 来自相关但独立的历史/对话的任何过去消息或事件（记忆）
- 关于输出何种结构化数据的指令

![image](https://github.com/user-attachments/assets/0f1f193f-8e94-4044-a276-576bd7764fd0)


### 关于上下文工程

本指南的核心是关于如何从当今的模型中获取最大价值。特别没有提及的是：

- 更改模型参数，如 temperature、top_p、frequency_penalty、presence_penalty 等
- 训练你自己的补全模型或嵌入模型
- 微调现有模型

同样，我不知道什么是将上下文传递给 LLM 的最佳方式，但我知道你希望能够灵活地尝试一切。

#### 标准与自定义上下文格式

大多数 LLM 客户端使用类似这样的标准消息格式：

```yaml
[
  {
    "role": "system",
    "content": "You are a helpful assistant..."
  },
  {
    "role": "user",
    "content": "Can you deploy the backend?"
  },
  {
    "role": "assistant",
    "content": null,
    "tool_calls": [
      {
        "id": "1",
        "name": "list_git_tags",
        "arguments": "{}"
      }
    ]
  },
  {
    "role": "tool",
    "name": "list_git_tags",
    "content": "{\"tags\": [{\"name\": \"v1.2.3\", \"commit\": \"abc123\", \"date\": \"2024-03-15T10:00:00Z\"}, {\"name\": \"v1.2.2\", \"commit\": \"def456\", \"date\": \"2024-03-14T15:30:00Z\"}, {\"name\": \"v1.2.1\", \"commit\": \"abe033d\", \"date\": \"2024-03-13T09:15:00Z\"}]}",
    "tool_call_id": "1"
  }
]
```

虽然这对大多数用例来说效果很好，但如果你真想从当今的 LLM 中获得最大收益，你需要以最高效利用 token 和注意力的方式来将上下文传递给 LLM。

作为标准消息格式的替代方案，你可以构建针对你的用例优化的自定义上下文格式。例如，你可以使用自定义对象，并根据需要将它们打包/展开到一个或多个 user、system、assistant 或 tool 消息中。

下面是将整个上下文窗口放入单个 user 消息的示例：
```yaml

[
  {
    "role": "system",
    "content": "You are a helpful assistant..."
  },
  {
    "role": "user",
    "content": |
            Here's everything that happened so far:
        
        <slack_message>
            From: @alex
            Channel: #deployments
            Text: Can you deploy the backend?
        </slack_message>
        
        <list_git_tags>
            intent: "list_git_tags"
        </list_git_tags>
        
        <list_git_tags_result>
            tags:
              - name: "v1.2.3"
                commit: "abc123"
                date: "2024-03-15T10:00:00Z"
              - name: "v1.2.2"
                commit: "def456"
                date: "2024-03-14T15:30:00Z"
              - name: "v1.2.1"
                commit: "ghi789"
                date: "2024-03-13T09:15:00Z"
        </list_git_tags_result>
        
        what's the next step?
    }
]
```

模型可能会通过你提供的工具 schema 推断出你在问它 `what's the next step`，但将其融入你的提示词模板中也无妨。

### 代码示例

我们可以用类似这样的方式来构建：

```python

class Thread:
  events: List[Event]

class Event:
  # 可以只用字符串，也可以显式声明 - 随你选择
  type: Literal["list_git_tags", "deploy_backend", "deploy_frontend", "request_more_information", "done_for_now", "list_git_tags_result", "deploy_backend_result", "deploy_frontend_result", "request_more_information_result", "done_for_now_result", "error"]
  data: ListGitTags | DeployBackend | DeployFrontend | RequestMoreInformation |  
        ListGitTagsResult | DeployBackendResult | DeployFrontendResult | RequestMoreInformationResult | string

def event_to_prompt(event: Event) -> str:
    data = event.data if isinstance(event.data, str) \
           else stringifyToYaml(event.data)

    return f"<{event.type}>\n{data}\n</{event.type}>"


def thread_to_prompt(thread: Thread) -> str:
  return '\n\n'.join(event_to_prompt(event) for event in thread.events)
```

#### 上下文窗口示例

使用这种方法，上下文窗口可能如下所示：

**初始 Slack 请求：**
```xml
<slack_message>
    From: @alex
    Channel: #deployments
    Text: Can you deploy the latest backend to production?
</slack_message>
```

**列出 Git 标签后：**
```xml
<slack_message>
    From: @alex
    Channel: #deployments
    Text: Can you deploy the latest backend to production?
    Thread: []
</slack_message>

<list_git_tags>
    intent: "list_git_tags"
</list_git_tags>

<list_git_tags_result>
    tags:
      - name: "v1.2.3"
        commit: "abc123"
        date: "2024-03-15T10:00:00Z"
      - name: "v1.2.2"
        commit: "def456"
        date: "2024-03-14T15:30:00Z"
      - name: "v1.2.1"
        commit: "ghi789"
        date: "2024-03-13T09:15:00Z"
</list_git_tags_result>
```

**错误和恢复后：**
```xml
<slack_message>
    From: @alex
    Channel: #deployments
    Text: Can you deploy the latest backend to production?
    Thread: []
</slack_message>

<deploy_backend>
    intent: "deploy_backend"
    tag: "v1.2.3"
    environment: "production"
</deploy_backend>

<error>
    error running deploy_backend: Failed to connect to deployment service
</error>

<request_more_information>
    intent: "request_more_information_from_human"
    question: "I had trouble connecting to the deployment service, can you provide more details and/or check on the status of the service?"
</request_more_information>

<human_response>
    data:
      response: "I'm not sure what's going on, can you check on the status of the latest workflow?"
</human_response>
```

从这里开始，你的下一步可能是：

```python
nextStep = await determine_next_step(thread_to_prompt(thread))
```

```python
{
  "intent": "get_workflow_status",
  "workflow_name": "tag_push_prod.yaml",
}
```

XML 风格的格式只是一个示例 —— 关键在于你可以构建适合你应用程序的自定义格式。如果你能够灵活地尝试不同的上下文结构以及存储内容与传递给 LLM 的内容之间的平衡，你将获得更好的质量。

掌控上下文窗口的主要优势：

1. **信息密度**：以最大化 LLM 理解的方式来组织信息
2. **错误处理**：以有助于 LLM 恢复的格式包含错误信息。考虑在错误和失败的调用解决后，将其从上下文窗口中隐藏。
3. **安全性**：控制传递给 LLM 的信息，过滤掉敏感数据
4. **灵活性**：随着你了解什么最适合你的用例，随时调整格式
5. **Token 效率**：针对 token 效率和 LLM 理解来优化上下文格式

上下文包括：提示词、指令、RAG 文档、历史记录、工具调用、记忆

记住：上下文窗口是你与 LLM 的主要接口。掌控你组织和呈现信息的方式，可以显著提升你的智能体性能。

示例 - 信息密度 - 同样的信息，更少的 token：

![Loom Screenshot 2025-04-22 at 09 00 56](https://github.com/user-attachments/assets/5cf041c6-72da-4943-be8a-99c73162b12a)


### 别只听我说

在 12-factor agents 发布大约两个月后，上下文工程开始成为一个相当流行的术语。

<a href="https://x.com/karpathy/status/1937902205765607626"><img width="378" alt="Screenshot 2025-06-25 at 4 11 45 PM" src="https://github.com/user-attachments/assets/97e6e667-c35f-4855-8233-af40f05d6bce" /></a> <a href="https://x.com/tobi/status/1935533422589399127"><img width="378" alt="Screenshot 2025-06-25 at 4 12 59 PM" src="https://github.com/user-attachments/assets/7e6f5738-0d38-4910-82d1-7f5785b82b99" /></a>

2025 年 7 月，[@lenadroid](https://x.com/lenadroid) 还制作了一份相当不错的[上下文工程速查表](https://x.com/lenadroid/status/1943685060785524824)。

<a href="https://x.com/lenadroid/status/1943685060785524824"><img width="256" alt="image" src="https://github.com/user-attachments/assets/cac88aa3-8faf-440b-9736-cab95a9de477" /></a>



这里反复出现的主题是：我不知道什么是最佳方法，但我知道你希望能够灵活地尝试一切。


[← 掌控你的提示词](factor-02-own-your-prompts.md) | [工具就是结构化输出 →](factor-04-tools-are-structured-outputs.md)
