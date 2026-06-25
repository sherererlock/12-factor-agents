[← 返回 README](../README.md)

### 9. 将错误压缩进上下文窗口

这一点内容较短但值得一提。代理的好处之一是"自我修复"——对于短任务，LLM 可能会调用一个失败的工具。优秀的 LLM 有相当好的机会阅读错误消息或堆栈跟踪，并在后续工具调用中找出需要更改的内容。


大多数框架都实现了这一点，但你可以只做这一点而不做其他 11 个因子中的任何一个。下面是一个示例：


```python
thread = {"events": [initial_message]}

while True:
  next_step = await determine_next_step(thread_to_prompt(thread))
  thread["events"].append({
    "type": next_step.intent,
    "data": next_step,
  })
  try:
    result = await handle_next_step(thread, next_step) # our switch statement
  except Exception as e:
    # if we get an error, we can add it to the context window and try again
    thread["events"].append({
      "type": 'error',
      "data": format_error(e),
    })
    # loop, or do whatever else here to try to recover
```

你可能想为特定工具调用实现一个 errorCounter，将单个工具的尝试次数限制在约 3 次，或任何其他对你的用例有意义的逻辑。

```python
consecutive_errors = 0

while True:

  # ... existing code ...

  try:
    result = await handle_next_step(thread, next_step)
    thread["events"].append({
      "type": next_step.intent + '_result',
      data: result,
    })
    # success! reset the error counter
    consecutive_errors = 0
  except Exception as e:
    consecutive_errors += 1
    if consecutive_errors < 3:
      # do the loop and try again
      thread["events"].append({
        "type": 'error',
        "data": format_error(e),
      })
    else:
      # break the loop, reset parts of the context window, escalate to a human, or whatever else you want to do
      break
  }
}
```
达到某个连续错误阈值可能是[向人类升级](factor-07-contact-humans-with-tools.md)的好时机，无论是通过模型决策还是通过确定性地接管控制流。

[![195-factor-09-errors](../../img/195-factor-09-errors.gif)](https://github.com/user-attachments/assets/cd7ed814-8309-4baf-81a5-9502f91d4043)


<details>
<summary>[GIF 版本](../../img/195-factor-09-errors.gif)</summary>

![195-factor-09-errors](../../img/195-factor-09-errors.gif)

</details>

优势：

1. **自我修复**：LLM 可以阅读错误消息并在后续工具调用中找出需要更改的内容
2. **持久性**：即使某个工具调用失败，代理也可以继续运行

我相信你会发现，如果你这样做太多，你的代理会开始失控，并可能一遍又一遍地重复相同的错误。

这就是 [因子 8 - 掌控你的控制流](factor-08-own-your-control-flow.md) 和 [因子 3 - 掌控你的上下文构建](factor-03-own-your-context-window.md) 发挥作用的地方——你不需要只是把原始错误放回去，你可以完全重构它的表示方式，从上下文窗口中移除先前的事件，或者任何你发现能让代理回到正轨的确定性方法。

但防止错误失控的首要方法是拥抱 [因子 10 - 小而专注的代理](factor-10-small-focused-agents.md)。

[← 掌控你的控制流](factor-08-own-your-control-flow.md) | [小而专注的代理 →](factor-10-small-focused-agents.md)
