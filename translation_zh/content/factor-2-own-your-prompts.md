[← 返回 README](../README.md)

### 2. 掌控你的 Prompt

不要将你的 prompt 工程外包给框架。

![120-own-your-prompts](../../img/120-own-your-prompts.png)

顺便说一下，[这远非新颖的建议：](https://hamel.dev/blog/posts/prompt/)

![image](https://github.com/user-attachments/assets/575bab37-0f96-49fb-9ce3-9a883cdd420b)

一些框架提供类似这样的"黑盒"方式：

```python
agent = Agent(
  role="...",
  goal="...",
  personality="...",
  tools=[tool1, tool2, tool3]
)

task = Task(
  instructions="...",
  expected_output=OutputModel
)

result = agent.run(task)
```

这对于引入一些顶级的 prompt 工程来帮助你起步非常好，但通常很难调优和/或逆向工程，以获得恰好正确的 token 输入到你的模型中。

相反，你应该掌控你的 prompt，将它们视为一等代码：

```rust
function DetermineNextStep(thread: string) -> DoneForNow | ListGitTags | DeployBackend | DeployFrontend | RequestMoreInformation {
  prompt #"
    {{ _.role("system") }}
    
    You are a helpful assistant that manages deployments for frontend and backend systems.
    You work diligently to ensure safe and successful deployments by following best practices
    and proper deployment procedures.
    
    Before deploying any system, you should check:
    - The deployment environment (staging vs production)
    - The correct tag/version to deploy
    - The current system status
    
    You can use tools like deploy_backend, deploy_frontend, and check_deployment_status
    to manage deployments. For sensitive deployments, use request_approval to get
    human verification.
    
    Always think about what to do first, like:
    - Check current deployment status
    - Verify the deployment tag exists
    - Request approval if needed
    - Deploy to staging before production
    - Monitor deployment progress
    
    {{ _.role("user") }}

    {{ thread }}
    
    What should the next step be?
  "#
}
```

（上面的示例使用 [BAML](https://github.com/boundaryml/baml) 来生成 prompt，但你可以使用任何你喜欢的 prompt 工程工具，甚至只是手动模板化）

如果函数签名看起来有点奇怪，我们将在 [Factor 4 - 工具就是结构化输出](factor-04-tools-are-structured-outputs.md)中详细说明

```typescript
function DetermineNextStep(thread: string) -> DoneForNow | ListGitTags | DeployBackend | DeployFrontend | RequestMoreInformation {
```

掌控 prompt 的核心优势：

1. **完全控制**：精确编写你的 Agent 需要的指令，没有黑盒抽象
2. **测试与评估**：像对待其他代码一样为你的 prompt 构建测试和评估
3. **迭代**：根据实际表现快速修改 prompt
4. **透明性**：确切知道你的 Agent 使用的是什么指令
5. **角色技巧**：利用支持非标准 user/assistant 角色用法的 API——例如，现已弃用的非聊天风格的 OpenAI "completions" API。这包括一些所谓的"模型诱导"技术

记住：你的 prompt 是应用逻辑与 LLM 之间的主要接口。

完全掌控你的 prompt 能为你提供生产级 Agent 所需的灵活性和 prompt 控制能力。

我不知道什么是最好的 prompt，但我知道你希望能够灵活地尝试一切。

[← 自然语言转工具调用](factor-01-natural-language-to-tool-calls.md) | [掌控你的上下文窗口 →](factor-03-own-your-context-window.md)
