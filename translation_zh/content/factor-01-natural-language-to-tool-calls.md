[← 返回 README](../README.md)

### 1. 自然语言转工具调用

在智能体构建中，最常见的模式之一是将自然语言转换为结构化的工具调用。这是一种强大的模式，它使你能够构建能够推理任务并执行任务的智能体。

![110-natural-language-tool-calls](../../img/110-natural-language-tool-calls.png)

当原子化地应用此模式时，就是将类似这样的自然语言短语

> 能帮我创建一个 750 美元的支付链接给 Terri，用于赞助二月份的 AI Tinkerers 聚会吗？

转换为一个描述 Stripe API 调用的结构化对象，例如

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

**注意**：实际上 Stripe API 要更复杂一些，[真正实现此功能的智能体](https://github.com/dexhorthy/mailcrew)（[视频](https://www.youtube.com/watch?v=f_cKnoPC_Oo)）需要先列出客户、列出产品、列出价格等，以便使用正确的 ID 构建此请求负载，或者将这些 ID 包含在提示词/上下文窗口中（我们将在下文中看到，这两者其实本质上是一样的！）

然后，确定性代码可以接管该请求负载并对其执行操作。（更多内容请参见[因素 3](factor-03-own-your-context-window.md)）

```python
# LLM 接收自然语言并返回结构化对象
nextStep = await llm.determineNextStep(
  """
  create a payment link for $750 to Jeff 
  for sponsoring the february AI tinkerers meetup
  """
  )

# 根据函数名处理结构化输出
if nextStep.function == 'create_payment_link':
    stripe.paymentlinks.create(nextStep.parameters)
    return  # 或者你想要的任何操作，见下文
elif nextStep.function == 'something_else':
    # ... 更多情况
    pass
else:  # 模型没有调用我们已知的工具
    # 执行其他操作
    pass
```

**注意**：虽然一个完整的智能体会接收 API 调用的结果并循环处理，最终返回类似这样的内容

> 我已成功为 Terri 创建了一个 750 美元的支付链接，用于赞助二月份的 AI Tinkerers 聚会。链接如下：https://buy.stripe.com/test_1234567890

**但实际上**，我们在这里将跳过这个步骤，把它留到另一个因素中讨论，你可以选择是否也要加入该部分（取决于你！）

[← 我们是如何走到这里的](brief-history-of-software.md) | [掌控你的提示词 →](factor-02-own-your-prompts.md)
