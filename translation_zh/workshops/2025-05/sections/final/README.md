# 第 0 章 - Hello World

让我们从一个基本的 TypeScript 设置和一个 hello world 程序开始。

本指南使用 TypeScript 编写（是的，Python 版本即将推出）

在工作坊步骤的每个文件编辑之间有许多检查点，
所以即使你对 TypeScript 不太熟悉，
也应该能够跟上并运行每个示例。

要运行本指南，你需要安装较新版本的 nodejs 和 npm

你可以使用任何你喜欢的 nodejs 版本管理器，[homebrew](https://formulae.brew.sh/formula/node) 就可以


    brew install node@20

你应该能看到 node 版本

    node --version

复制初始 package.json

    cp ./walkthrough/00-package.json package.json

安装依赖

    npm install

复制 tsconfig.json

    cp ./walkthrough/00-tsconfig.json tsconfig.json

添加 .gitignore

    cp ./walkthrough/00-.gitignore .gitignore

创建 src 文件夹

    mkdir -p src

添加一个简单的 hello world index.ts

    cp ./walkthrough/00-index.ts src/index.ts

运行验证

    npx tsx src/index.ts

你应该会看到：

    hello, world!


# 第 1 章 - CLI 和 Agent 循环

现在让我们添加 BAML 并创建第一个带有 CLI 界面的 agent。

首先，我们需要安装 [BAML](https://github.com/boundaryml/baml)，
这是一个用于提示词和结构化输出的工具。


    npm install @boundaryml/baml

初始化 BAML

    npx baml-cli init

移除默认的 resume.baml

    rm baml_src/resume.baml

添加我们的起始 agent，一个我们将在其基础上构建的单一 baml 提示词

    cp ./walkthrough/01-agent.baml baml_src/agent.baml

生成 BAML 客户端代码

    npx baml-cli generate

在本节中启用 BAML 日志

    export BAML_LOG=debug

添加 CLI 界面

    cp ./walkthrough/01-cli.ts src/cli.ts

更新 index.ts 以使用 CLI

    cp ./walkthrough/01-index.ts src/index.ts

添加 agent 实现

    cp ./walkthrough/01-agent.ts src/agent.ts

BAML 代码默认配置为使用 OPENAI_API_KEY

在测试时，你可以根据需要将模型/提供者更改为其他选项

        client "openai/gpt-4o"

[BAML 客户端文档可以在这里找到](https://docs.boundaryml.com/guide/baml-basics/switching-llms)

例如，你可以配置 [gemini](https://docs.boundaryml.com/ref/llm-client-providers/google-ai-gemini)
或 [anthropic](https://docs.boundaryml.com/ref/llm-client-providers/anthropic) 作为你的模型提供者。

如果你想不加修改地运行示例，可以将 OPENAI_API_KEY 环境变量设置为任何有效的 openai 密钥。


    export OPENAI_API_KEY=...

试一试

    npx tsx src/index.ts hello

你应该会看到模型的熟悉响应

    {
  intent: 'done_for_now',
  message: 'Hello! How can I assist you today?'
}


# 第 2 章 - 添加计算器工具

让我们为 agent 添加一些计算器工具。

让我们从为计算器添加工具定义开始

这些是我们要求模型作为智能体循环中的"下一步"返回的简单结构化输出。


    cp ./walkthrough/02-tool_calculator.baml baml_src/tool_calculator.baml

现在，让我们更新 agent 的 DetermineNextStep 方法，
将计算器工具作为潜在的下一步暴露出来


    cp ./walkthrough/02-agent.baml baml_src/agent.baml

生成更新后的 BAML 客户端

    npx baml-cli generate

试一试计算器

    npx tsx src/index.ts 'can you add 3 and 4'

你应该会看到一个对计算器的工具调用

    {
  intent: 'add',
  a: 3,
  b: 4
}


# 第 3 章 - 在循环中处理工具调用

现在让我们添加一个真正的智能体循环，可以运行工具并从 LLM 获取最终答案。

首先，让我们更新 agent 以处理工具调用


    cp ./walkthrough/03-agent.ts src/agent.ts

现在，让我们试一试


    npx tsx src/index.ts 'can you add 3 and 4'

你应该会看到 agent 调用工具然后返回结果

    {
  intent: 'done_for_now',
  message: 'The sum of 3 and 4 is 7.'
}

对于下一步，我们将进行更复杂的计算，让我们关闭 baml 日志以获得更简洁的输出

    export BAML_LOG=off

尝试多步计算

    npx tsx src/index.ts 'can you add 3 and 4, then add 6 to that result'

你会注意到 multiply 和 divide 等工具不可用

    npx tsx src/index.ts 'can you multiply 3 and 4'

接下来，让我们为其余的计算器工具添加处理程序


    cp ./walkthrough/03b-agent.ts src/agent.ts

测试减法

    npx tsx src/index.ts 'can you subtract 3 from 4'

现在，让我们测试乘法工具


    npx tsx src/index.ts 'can you multiply 3 and 4'

最后，让我们测试一个包含多个操作的更复杂计算


    npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'


# 第 4 章 - 为 agent.baml 添加测试

让我们为 BAML agent 添加一些测试。

首先，保持 baml 日志处于启用状态

    export BAML_LOG=debug

接下来，让我们为 agent 添加一些测试

我们从一个简单的测试开始，检查 agent 处理基本计算的能力。


    cp ./walkthrough/04-agent.baml baml_src/agent.baml

运行测试

    npx baml-cli test

现在，让我们通过断言来改进测试！

断言是确保 agent 按预期工作的好方法，并且可以轻松扩展以检查更复杂的行为。


    cp ./walkthrough/04b-agent.baml baml_src/agent.baml

运行测试

    npx baml-cli test

随着你添加更多测试，可以禁用日志以保持输出整洁。
在对特定测试进行迭代时，你可能会想重新启用它们。


    export BAML_LOG=off

现在，让我们添加一些更复杂的测试用例，
从正在进行的智能体上下文窗口中间恢复执行


    cp ./walkthrough/04c-agent.baml baml_src/agent.baml

让我们尝试运行它


    npx baml-cli test


# 第 5 章 - 多个人类工具

在本节中，我们将添加对多个用于联系人类的工具的支持。


在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

首先，让我们添加一个可以向人类请求澄清的工具

这将与 "done_for_now" 工具不同，
可以用于在你的 agent 中更灵活地处理不同类型的人类交互。


    cp ./walkthrough/05-agent.baml baml_src/agent.baml

接下来，让我们重新生成客户端代码

注意 - 如果你使用的是 BAML 的 VSCode 扩展，
当你在编辑器中保存文件时，客户端会自动重新生成。


    npx baml-cli generate

现在，让我们更新 agent 以使用新的工具


    cp ./walkthrough/05-agent.ts src/agent.ts

接下来，让我们更新 CLI 以处理澄清请求，
通过在命令行上向用户请求输入


    cp ./walkthrough/05-cli.ts src/cli.ts

让我们试一试


    npx tsx src/index.ts 'can you multiply 3 and FD*(#F&& '

接下来，让我们添加一个测试来检查 agent 处理澄清请求的能力


    cp ./walkthrough/05b-agent.baml baml_src/agent.baml

现在我们可以再次运行测试


    npx baml-cli test

你会注意到新测试通过了，但 hello world 测试失败了

这是因为 agent 的默认行为是返回 "done_for_now"


    cp ./walkthrough/05c-agent.baml baml_src/agent.baml

验证测试通过

    npx baml-cli test


# 第 6 章 - 使用推理自定义你的提示词

在本节中，我们将探索如何使用推理步骤来自定义 agent 的提示词。

这是[因素 2 - 掌控你的提示词](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-2-own-your-prompts.md)的核心内容

AI That Works 上有一篇关于推理的深入文章：[推理模型与推理步骤](https://github.com/hellovai/ai-that-works/tree/main/2025-04-07-reasoning-models-vs-prompts)


在本节中，保持 baml 日志处于启用状态会很有帮助

    export BAML_LOG=debug

更新 agent 提示词以包含推理步骤


    cp ./walkthrough/06-agent.baml baml_src/agent.baml

生成更新后的客户端

    npx baml-cli generate

现在，你可以用一个简单的提示词来试一试


    npx tsx src/index.ts 'can you multiply 3 and 4'

你应该能看到 baml 日志输出中显示的推理步骤

#### 可选挑战

在你的工具输出格式中添加一个字段，将推理步骤包含在输出中！



# 第 7 章 - 自定义你的上下文窗口

在本节中，我们将探索如何自定义 agent 的上下文窗口。

这是[因素 3 - 掌控你的上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-3-own-your-context-window.md)的核心内容


更新 agent 以美化打印模型的上下文窗口


    cp ./walkthrough/07-agent.ts src/agent.ts

测试格式化

    BAML_LOG=info npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

接下来，让我们更新 agent 改用 XML 格式

这是一种非常流行的数据传递格式，

其中一个重要原因是 XML 的 token 效率很高。


    cp ./walkthrough/07b-agent.ts src/agent.ts

让我们试一试


    BAML_LOG=info npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

让我们更新测试以匹配新的输出格式


    cp ./walkthrough/07c-agent.baml baml_src/agent.baml

查看更新后的测试


    npx baml-cli test


# 第 8 章 - 添加 API 端点

添加一个 Express 服务器以通过 HTTP 暴露 agent。

在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

安装 Express 和类型定义

    npm install express && npm install --save-dev @types/express supertest

添加服务器实现

    cp ./walkthrough/08-server.ts src/server.ts

启动服务器

    npx tsx src/server.ts

使用 curl 测试（在另一个终端中）

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you add 3 and 4"}'

你应该会收到 agent 的回复，其中包含智能体追踪信息，最后是一条类似这样的消息：


    {"intent":"done_for_now","message":"The sum of 3 and 4 is 7."}


# 第 9 章 - 内存状态管理和异步澄清

添加状态管理和异步澄清支持。

在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

为线程添加一些简单的内存状态管理

    cp ./walkthrough/09-state.ts src/state.ts

更新服务器以使用状态管理

* 使用 `ThreadStore` 添加线程状态管理
* 从 /thread 端点返回线程 ID 和响应 URL
* 实现 GET /thread/:id
* 实现 POST /thread/:id/response


    cp ./walkthrough/09-server.ts src/server.ts

启动服务器

    npx tsx src/server.ts

测试澄清流程

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you multiply 3 and xyz"}'


# 第 10 章 - 添加人工审批

添加对操作的人工审批支持。

在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

更新服务器以处理人工审批

* 导入 `handleNextStep` 以执行已批准的操作
* 添加两种负载类型以区分审批和响应
* 在端点中分别处理响应和审批
* 在出错时显示更好的错误消息


    cp ./walkthrough/10-server.ts src/server.ts

为 agent 添加一些方法来处理审批和响应

    cp ./walkthrough/10-agent.ts src/agent.ts

启动服务器

    npx tsx src/server.ts

测试带审批的除法运算

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you divide 3 by 4"}'

你应该会看到：

    {
  "thread_id": "2b243b66-215a-4f37-8bc6-9ace3849043b",
  "events": [
    {
      "type": "user_input",
      "data": "can you divide 3 by 4"
    },
    {
      "type": "tool_call",
      "data": {
        "intent": "divide",
        "a": 3,
        "b": 4,
        "response_url": "/thread/2b243b66-215a-4f37-8bc6-9ace3849043b/response"
      }
    }
  ]
}

使用另一个 curl 调用拒绝请求，更改线程 ID

    curl -X POST 'http://localhost:3000/thread/{thread_id}/response' \
  -H "Content-Type: application/json" \
  -d '{"type": "approval", "approved": false, "comment": "I dont think thats right, use 5 instead of 4"}'

你应该会看到：最后一次工具调用现在变成了 `"intent":"divide","a":3,"b":5`

    {
  "events": [
    {
      "type": "user_input",
      "data": "can you divide 3 by 4"
    },
    {
      "type": "tool_call",
      "data": {
        "intent": "divide",
        "a": 3,
        "b": 4,
        "response_url": "/thread/2b243b66-215a-4f37-8bc6-9ace3849043b/response"
      }
    },
    {
      "type": "tool_response",
      "data": "user denied the operation with feedback: \"I dont think thats right, use 5 instead of 4\""
    },
    {
      "type": "tool_call",
      "data": {
        "intent": "divide",
        "a": 3,
        "b": 5,
        "response_url": "/thread/1f1f5ff5-20d7-4114-97b4-3fc52d5e0816/response"
      }
    }
  ]
}

现在你可以批准该操作了

    curl -X POST 'http://localhost:3000/thread/{thread_id}/response' \
  -H "Content-Type: application/json" \
  -d '{"type": "approval", "approved": true}'

你应该会看到最终消息包含工具响应和最终结果！

    ...
{
  "type": "tool_response",
  "data": 0.5
},
{
  "type": "done_for_now",
  "message": "I divided 3 by 6 and the result is 0.5. If you have any more operations or queries, feel free to ask!",
  "response_url": "/thread/2b469403-c497-4797-b253-043aae830209/response"
}


# 第 11 章 - 通过邮件进行人工审批

在本节中，我们将添加通过邮件进行人工审批的支持。

这部分内容一开始会有点刻意，只是为了让大家理解基本概念 -

我们将从 CLI 调用工作流开始，但 `divide` 和 `request_more_information` 的审批将通过邮件处理，
然后最终的 `done_for_now` 答案将打印回 CLI

虽然有些刻意，但这是你从[因素 7 - 使用工具联系人类](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-7-contact-humans-with-tools.md)中获得灵活性的一个很好的例子


在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

安装 HumanLayer

    npm install humanlayer

更新 CLI 以通过邮件将 `divide` 和 `request_more_information` 发送给人类

    cp ./walkthrough/11-cli.ts src/cli.ts

运行 CLI

    npx tsx src/index.ts 'can you divide 4 by 5'

程序的最后一行应该会提到人工审核步骤

    nextStep { intent: 'divide', a: 4, b: 5 }
HumanLayer: Requested human approval from HumanLayer cloud

继续回复邮件并提供一些反馈：

![reject-email](https://github.com/humanlayer/12-factor-agents/blob/main/workshops/2025-05/walkthrough/11-email-reject.png?raw=true)


你应该会收到另一封邮件，其中包含根据你的反馈更新后的尝试！

你可以批准这次操作：

![approve-email](https://github.com/humanlayer/12-factor-agents/blob/main/workshops/2025-05/walkthrough/11-email-approve.png?raw=true)


你的最终输出将如下所示

    nextStep {
 intent: 'done_for_now',
 message: 'The division of 4 by 5 is 0.8. If you have any other calculations or questions, feel free to ask!'
}
The division of 4 by 5 is 0.8. If you have any other calculations or questions, feel free to ask!

让我们也实现 `request_more_information` 流程


    cp ./walkthrough/11b-cli.ts src/cli.ts

让我们通过请求一个带有乱码输入的计算来测试 require_approval 流程：


    npx tsx src/index.ts 'can you multiply 4 and xyz'

你应该会收到一封请求澄清的邮件

    Can you clarify what 'xyz' represents in this context? Is it a specific number, variable, or something else?

你可以回复类似这样的内容

    use 8 instead of xyz

你应该会在 CLI 上看到类似这样的最终结果

    I have multiplied 4 and xyz, using the value 8 for xyz, resulting in 32.

作为最后一步，让我们探索使用自定义 HTML 模板来发送邮件


    cp ./walkthrough/11c-cli.ts src/cli.ts

先用 divide 试试：


    npx tsx src/index.ts 'can you divide 4 by 5'

你应该会看到使用自定义模板的略有不同的邮件

![custom-template-email](https://github.com/humanlayer/12-factor-agents/blob/main/workshops/2025-05/walkthrough/11-email-custom.png?raw=true)

按照流程继续操作，然后你可以尝试按自己的喜好更新模板

（如果你使用 cursor，只需高亮模板并要求"让它变得更好"就可以了）

也试着触发 "request_more_information"！


就这些了 - 在下一章中，我们将构建一个完全由邮件驱动的工作流 agent，使用 webhook 进行人工审批



# 第 XX 章 - HumanLayer Webhook 集成

前面的章节使用了 humanlayer SDK 的"同步模式" - 这意味着每次我们等待人工审批时，都会在一个循环中轮询直到收到人工响应。

这显然不是理想的做法，特别是对于生产工作负载，
因此在本节中，我们将实现[因素 6 - 使用简单 API 启动 / 暂停 / 恢复](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-6-launch-pause-resume.md)，
通过更新服务器在联系人类后结束处理，并使用 webhook 接收结果。


添加在服务器中初始化 humanlayer 的代码


    cp ./walkthrough/12-1-server-init.ts src/server.ts

接下来，让我们更新 /thread 端点以
  
1. 异步处理请求，立即返回
2. 在 request_more_information 和 done_for_now 调用时创建人类联系人


更新服务器以能够处理 request_clarification 响应

- 移除旧的 /response 端点和类型
- 更新 /thread 端点以异步运行处理，立即返回
- 在请求人工响应时发送 state.threadId
- 添加 handleHumanResponse 函数来处理人工响应
- 添加 /webhook 端点来处理 webhook 响应


    cp ./walkthrough/12a-server.ts src/server.ts

在另一个终端中启动服务器

    npx tsx src/server.ts

现在服务器正在运行，向 '/thread' 端点发送一个负载


__ 执行响应步骤

__ 现在处理 divide 的审批

__ 现在也处理 done_for_now

