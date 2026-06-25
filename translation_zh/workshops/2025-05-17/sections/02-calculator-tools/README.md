# 第 2 章 - 添加计算器工具

让我们为 agent 添加一些计算器工具。

让我们从为计算器添加工具定义开始

这些是简单的结构化输出，我们将要求模型作为 agent 循环中的"下一步"返回。


    cp ./walkthrough/02-tool_calculator.baml baml_src/tool_calculator.baml

<details>
<summary>查看文件</summary>

```rust
// ./walkthrough/02-tool_calculator.baml
type CalculatorTools = AddTool | SubtractTool | MultiplyTool | DivideTool


class AddTool {
    intent "add"
    a int | float
    b int | float
}

class SubtractTool {
    intent "subtract"
    a int | float
    b int | float
}

class MultiplyTool {
    intent "multiply"
    a int | float
    b int | float
}

class DivideTool {
    intent "divide"
    a int | float
    b int | float
}
```

</details>

现在，让我们更新 agent 的 DetermineNextStep 方法，
将计算器工具作为潜在的下一步暴露出来


```diff
baml_src/agent.baml
 function DetermineNextStep(
     thread: string 
-) -> DoneForNow {
+) -> CalculatorTools | DoneForNow {
     client Qwen3
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/02-agent.baml baml_src/agent.baml

</details>

生成更新的 BAML 客户端

    npx baml-cli generate

尝试计算器

    npx tsx src/index.ts 'can you add 3 and 4'

你应该看到对计算器的工具调用

    {
      intent: 'add',
      a: 3,
      b: 4
    }

