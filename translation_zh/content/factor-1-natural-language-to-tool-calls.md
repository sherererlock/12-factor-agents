[← 返回 README](../README.md)

### 1. 自然语言转工具调用

Agent 构建中最常见的模式之一是将自然语言转换为结构化的工具调用。这是一种强大的模式，允许你构建能够推理任务并执行它们的 Agent。

![110-natural-language-tool-calls](../../img/110-natural-language-tool-calls.png)

当原子化地应用此模式时，就是将类似这样的自然语言

> 你能为 Terri 创建一个 750 美元的支付链接吗，用于赞助二月份的 AI tinkerers 聚会？

转换为描述 Stripe API 调用的结构化对象，例如

```json
{
  "function": {
    "name": "create_payment_link",
    "parameters": {
      "amount": 750,
      "customer": "cust_128934ddasf9",
      "product": "prod_8675309",
      "price": "prc_09874329fds",
      "quantity": 1,
      "memo": "Hey Jeff - see below for the payment link for the february ai tinkerers meetup"
    }
  }
}
```

**注意**：实际上 Stripe API 要更复杂一些，[真正实现此功能的 Agent](https://github.com/dexhorthy/mailcrew)（[视频](https://www.youtube.com/watch?v=f_cKnoPC_Oo)）会列出客户、产品、价格等来构建带有正确 ID 的请求体，或者将这些 ID 包含在 prompt/上下文窗口中（我们稍后会看到这两者其实差不多是一回事！）

从那里，确定性代码可以接收请求体并对其进行处理。（更多内容请参见 [Factor 3](factor-03-own-your-context-window.md)）

```python
# LLM 接收自然语言并返回结构化对象
nextStep = await llm.determineNextStep(
  """
  create a payment link for $750 to Jeff 
  for sponsoring the february AI tinkerers meetup
  """
  )

# 根据功能处理结构化输出
if nextStep.function == 'create_payment_link':
    stripe.paymentlinks.create(nextStep.parameters)
    return  # 或者任何你想做的，见下文
elif nextStep.function == 'something_else':
    # ... 更多情况
    pass
else:  # 模型没有调用我们已知的工具
    # 做其他事情
    pass
```

**注意**：虽然一个完整的 Agent 会接收 API 调用结果并循环处理，最终返回类似这样的内容

> 我已成功为 Terri 创建了一个 750 美元的支付链接，用于赞助二月份的 AI tinkerers 聚会。链接如下：https://buy.stripe.com/test_1234567890

**但实际上**，我们将在这里跳过这个步骤，把它留给另一个 Factor，你可以选择是否也一并采用（由你决定！）

[← 我们如何走到这里](brief-history-of-software.md) | [掌控你的 Prompt →](factor-02-own-your-prompts.md)
