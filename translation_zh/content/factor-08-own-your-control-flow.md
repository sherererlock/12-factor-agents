[← 返回 README](../README.md)

### 8. 掌控你的控制流

如果你掌控了自己的控制流，就可以做很多有趣的事情。

![180-control-flow](../../img/180-control-flow.png)


构建适合你特定用例的自定义控制结构。具体来说，某些类型的工具调用可能是跳出循环并等待人类或其他长时间运行任务（如训练管道）响应的理由。你可能还希望加入以下自定义实现：

- 工具调用结果的摘要或缓存
- 对结构化输出进行 LLM 评判
- 上下文窗口压缩或其他[内存管理](factor-03-own-your-context-window.md)
- 日志记录、追踪和指标
- 客户端速率限制
- 持久化休眠 / 暂停 / "等待事件"


下面的示例展示了三种可能的控制流模式：


- request_clarification：模型请求更多信息，跳出循环并等待人类响应
- fetch_git_tags：模型请求 git 标签列表，获取标签，追加到上下文窗口，然后直接传回模型
- deploy_backend：模型请求部署后端，这是一个高风险操作，因此跳出循环并等待人类批准

```python
def handle_next_step(thread: Thread):

  while True:
    next_step = await determine_next_step(thread_to_prompt(thread))
    
    # inlined for clarity - in reality you could put 
    # this in a method, use exceptions for control flow, or whatever you want
    if next_step.intent == 'request_clarification':
      thread.events.append({
        type: 'request_clarification',
          data: nextStep,
        })

      await send_message_to_human(next_step)
      await db.save_thread(thread)
      # async step - break the loop, we'll get a webhook later
      break
    elif next_step.intent == 'fetch_open_issues':
      thread.events.append({
        type: 'fetch_open_issues',
        data: next_step,
      })

      issues = await linear_client.issues()

      thread.events.append({
        type: 'fetch_open_issues_result',
        data: issues,
      })
      # sync step - pass the new context to the LLM to determine the NEXT next step
      continue
    elif next_step.intent == 'create_issue':
      thread.events.append({
        type: 'create_issue',
        data: next_step,
      })

      await request_human_approval(next_step)
      await db.save_thread(thread)
      # async step - break the loop, we'll get a webhook later
      break
```

这种模式允许你根据需要中断和恢复代理的流程，创建更自然的对话和工作流。

**示例** - 我对目前所有 AI 框架的第一功能需求是：我们需要能够中断一个正在工作的代理并稍后恢复，尤其是在工具**选择**和工具**调用**之间的时刻。

如果没有这种可恢复性/粒度级别，就无法在工具调用运行之前进行审查/批准，这意味着你被迫选择以下之一：

1. 在等待长时间运行的任务完成时将任务暂停在内存中（想想 `while...sleep`），如果进程被中断则从头开始重启
2. 将代理限制为仅执行低风险、低风险的调用，如研究和摘要
3. 赋予代理执行更大、更有用操作的权限，然后就祈祷它不要搞砸


你可能会注意到这与 [因子 5 - 统一执行状态与业务状态](factor-05-unify-execution-state.md) 和 [因子 6 - 通过简单 API 启动/暂停/恢复](factor-06-launch-pause-resume.md) 密切相关，但可以独立实现。

[← 通过工具调用与人类沟通](factor-07-contact-humans-with-tools.md) | [紧凑错误处理 →](factor-09-compact-errors.md)
