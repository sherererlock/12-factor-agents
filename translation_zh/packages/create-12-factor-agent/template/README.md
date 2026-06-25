# 第 0 章 - Hello World

让我们从一个基本的 TypeScript 设置和一个 hello world 程序开始。

本指南使用 TypeScript 编写（是的，Python 版本即将推出）

在工作坊步骤的每次文件编辑之间有很多检查点，
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

运行以验证

    npx tsx src/index.ts

你应该看到：

    hello, world!


# 第 1 章 - CLI 和代理循环

现在让我们添加 BAML 并创建第一个带有 CLI 接口的代理。

首先，我们需要安装 [BAML](https://github.com/boundaryml/baml)，
这是一个用于提示和结构化输出的工具。


    npm install @boundaryml/baml

初始化 BAML

    npx baml-cli init

删除默认的 resume.baml

    rm baml_src/resume.baml

添加我们的起始代理，一个我们将在此基础上构建的单个 baml 提示

    cp ./walkthrough/01-agent.baml baml_src/agent.baml

生成 BAML 客户端代码

    npx baml-cli generate

为本节启用 BAML 日志

    export BAML_LOG=debug

添加 CLI 接口

    cp ./walkthrough/01-cli.ts src/cli.ts

更新 index.ts 以使用 CLI

    cp ./walkthrough/01-index.ts src/index.ts

添加代理实现

    cp ./walkthrough/01-agent.ts src/agent.ts

BAML 代码默认配置为使用 BASETEN_API_KEY

要获取 Baseten API 密钥和 URL，请在 [baseten.co](https://baseten.co) 创建账户，
然后从模型库部署 [Qwen3 32B](https://www.baseten.co/library/qwen-3-32b/)。

```rust 
  function DetermineNextStep(thread: string) -> DoneForNow {
      client Qwen3
      // ...
```

如果你想不做任何更改就运行示例，可以将 BASETEN_API_KEY 环境变量设置为任何有效的 baseten 密钥。

如果你想尝试更换模型，可以更改 `client` 行。

[BAML 客户端文档可以在这里找到](https://docs.boundaryml.com/guide/baml-basics/switching-llms)

例如，你可以配置 [gemini](https://docs.boundaryml.com/ref/llm-client-providers/google-ai-gemini) 
或 [anthropic](https://docs.boundaryml.com/ref/llm-client-providers/anthropic) 作为你的模型提供商。

例如，要使用带有 OPENAI_API_KEY 的 openai，你可以这样做：

    client "openai/gpt-4o"


设置你的环境变量

    export BASETEN_API_KEY=...
export BASETEN_BASE_URL=...

尝试一下

    npx tsx src/index.ts hello

你应该看到来自模型的熟悉响应

    {
      intent: 'done_for_now',
      message: 'Hello! How can I assist you today?'
    }


# 第 2 章 - 添加计算器工具

让我们为代理添加一些计算器工具。

让我们从为计算器添加工具定义开始

这些是我们将要求模型作为代理循环中的"下一步"返回的简单结构化输出。


    cp ./walkthrough/02-tool_calculator.baml baml_src/tool_calculator.baml

现在，让我们更新代理的 DetermineNextStep 方法，
将计算器工具作为潜在的下一步暴露出来


    cp ./walkthrough/02-agent.baml baml_src/agent.baml

生成更新的 BAML 客户端

    npx baml-cli generate

试试计算器

    npx tsx src/index.ts 'can you add 3 and 4'

你应该看到对计算器的工具调用

    {
      intent: 'add',
      a: 3,
      b: 4
    }


# 第 3 章 - 在循环中处理工具调用

现在让我们添加一个真正的代理循环，可以运行工具并从 LLM 获取最终答案。

首先，让我们更新代理以处理工具调用


    cp ./walkthrough/03-agent.ts src/agent.ts

现在，让我们试试


    npx tsx src/index.ts 'can you add 3 and 4'

你应该看到代理调用工具然后返回结果

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

接下来，让我们为其余的计算器工具添加处理器


    cp ./walkthrough/03b-agent.ts src/agent.ts

测试减法

    npx tsx src/index.ts 'can you subtract 3 from 4'

现在，让我们测试乘法工具


    npx tsx src/index.ts 'can you multiply 3 and 4'

最后，让我们测试一个包含多个操作的更复杂计算


    npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

恭喜，你已经迈出了手动构建代理循环的第一步。

从这里开始，我们将开始融入一些更中级和高级的
12 因代理概念。



# 第 4 章 - 为 agent.baml 添加测试

让我们为 BAML 代理添加一些测试。

首先，保持 baml 日志启用

    export BAML_LOG=debug

接下来，让我们为代理添加一些测试

我们将从一个简单的测试开始，检查代理处理基本计算的能力。


    cp ./walkthrough/04-agent.baml baml_src/agent.baml

运行测试

    npx baml-cli test

现在，让我们用断言来改进测试！

断言是确保代理按预期工作的好方法，
并且可以轻松扩展以检查更复杂的行为。


    cp ./walkthrough/04b-agent.baml baml_src/agent.baml

运行测试

    npx baml-cli test

随着你添加更多测试，你可以禁用日志以保持输出整洁。
你可能想在迭代特定测试时启用它们。


    export BAML_LOG=off

现在，让我们添加一些更复杂的测试用例，
我们从正在进行的代理上下文窗口的中间恢复


    cp ./walkthrough/04c-agent.baml baml_src/agent.baml

让我们尝试运行它


    npx baml-cli test


# 第 5 章 - 多个人类工具

在本节中，我们将添加对多种用于联系人类的工具的支持。


在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

首先，让我们添加一个可以向人类请求澄清的工具

这将不同于 "done_for_now" 工具，
可以用来更灵活地处理代理中不同类型的人类交互。


    cp ./walkthrough/05-agent.baml baml_src/agent.baml

接下来，让我们重新生成客户端代码

注意 - 如果你正在使用 BAML 的 VSCode 扩展，
当你在编辑器中保存文件时，客户端将自动重新生成。


    npx baml-cli generate

现在，让我们更新代理以使用新工具


    cp ./walkthrough/05-agent.ts src/agent.ts

接下来，让我们更新 CLI 以通过在 CLI 上请求用户输入来处理澄清请求


    cp ./walkthrough/05-cli.ts src/cli.ts

让我们试试


    npx tsx src/index.ts 'can you multiply 3 and FD*(#F&& '

接下来，让我们添加一个测试，检查代理处理澄清请求的能力


    cp ./walkthrough/05b-agent.baml baml_src/agent.baml

现在我们可以再次运行测试


    npx baml-cli test

你会注意到新测试通过了，但 hello world 测试失败了

这是因为代理的默认行为是返回 "done_for_now"


    cp ./walkthrough/05c-agent.baml baml_src/agent.baml

验证测试通过

    npx baml-cli test


# 第 6 章 - 使用推理自定义你的提示

在本节中，我们将探索如何使用推理步骤自定义代理的提示。

这是 [因子 2 - 掌控你的提示](../../../../content/factor-2-own-your-prompts.md) 的核心

AI That Works 上有一篇关于推理的深入文章 [推理模型 versus 推理步骤](https://github.com/hellovai/ai-that-works/tree/main/2025-04-07-reasoning-models-vs-prompts)


在本节中，保持 baml 日志启用会有所帮助

    export BAML_LOG=debug

更新代理提示以包含推理步骤


    cp ./walkthrough/06-agent.baml baml_src/agent.baml

生成更新的客户端

    npx baml-cli generate

现在，你可以用一个简单的提示试试


    npx tsx src/index.ts 'can you multiply 3 and 4'

你应该看到 baml 日志输出显示推理步骤

#### 可选挑战

在你的工具输出格式中添加一个字段，将推理步骤包含在输出中！



# 第 7 章 - 自定义你的上下文窗口

在本节中，我们将探索如何自定义代理的上下文窗口。

这是 [因子 3 - 掌控你的上下文窗口](../../../../content/factor-3-own-your-context-window.md) 的核心


更新代理以美化打印模型的上下文窗口


    cp ./walkthrough/07-agent.ts src/agent.ts

测试格式化

    BAML_LOG=info npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

接下来，让我们更新代理以使用 XML 格式

这是一种非常流行的数据传递给模型的格式，

除其他原因外，还因为 XML 的 token 效率。


    cp ./walkthrough/07b-agent.ts src/agent.ts

让我们试试


    BAML_LOG=info npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

让我们更新测试以匹配新的输出格式


    cp ./walkthrough/07c-agent.baml baml_src/agent.baml

查看更新的测试


    npx baml-cli test


# 第 8 章 - 添加 API 端点

添加一个 Express 服务器以通过 HTTP 暴露代理。

在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

安装 Express 和类型

    npm install express && npm install --save-dev @types/express supertest

添加服务器实现

    cp ./walkthrough/08-server.ts src/server.ts

启动服务器

    npx tsx src/server.ts

用 curl 测试（在另一个终端）

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you add 3 and 4"}'

你应该得到来自代理的答案，其中包含代理跟踪，以如下消息结尾：


    {"intent":"done_for_now","message":"The sum of 3 and 4 is 7."}


# 第 9 章 - 内存状态和异步澄清

添加状态管理和异步澄清支持。

在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

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


# 第 10 章 - 添加人类审批

添加对操作的人类审批支持。

在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

更新服务器以处理人类审批

* 导入 `handleNextStep` 以执行已批准的操作
* 添加两种有效载荷类型以区分审批和响应
* 在端点中分别处理响应和审批
* 出错时显示更好的错误消息


    cp ./walkthrough/10-server.ts src/server.ts

为代理添加一些方法以处理审批和响应

    cp ./walkthrough/10-agent.ts src/agent.ts

启动服务器

    npx tsx src/server.ts

测试带审批的除法

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you divide 3 by 4"}'

你应该看到：

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

用另一个 curl 调用拒绝请求，更改线程 ID

    curl -X POST 'http://localhost:3000/thread/{thread_id}/response' \
  -H "Content-Type: application/json" \
  -d '{"type": "approval", "approved": false, "comment": "I dont think thats right, use 5 instead of 4"}'

你应该看到：最后一个工具调用现在是 `"intent":"divide","a":3,"b":5`

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

现在你可以批准该操作

    curl -X POST 'http://localhost:3000/thread/{thread_id}/response' \
  -H "Content-Type: application/json" \
  -d '{"type": "approval", "approved": true}'

你应该看到最终消息包含工具响应和最终结果！

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


# 第 11 章 - 通过电子邮件进行人类审批

在本节中，我们将添加通过电子邮件进行人类审批的支持。

这开始会有点刻意，只是为了把概念弄清楚 -

我们将从 CLI 调用工作流开始，但 `divide` 和 `request_more_information` 的审批将通过电子邮件处理，
然后最终的 `done_for_now` 答案将打印回 CLI

虽然刻意，但这是你从 [因子 7 - 通过工具调用与人类沟通](../../../../content/factor-7-contact-humans-with-tools.md) 获得灵活性的一个很好的例子


在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

安装 HumanLayer

    npm install humanlayer

更新 CLI 以通过电子邮件将 `divide` 和 `request_more_information` 发送给人类

    cp ./walkthrough/11-cli.ts src/cli.ts

运行 CLI

    npx tsx src/index.ts 'can you divide 4 by 5'

你的程序的最后一行应该提到人类审查步骤

    nextStep { intent: 'divide', a: 4, b: 5 }
    HumanLayer: Requested human approval from HumanLayer cloud

继续回复电子邮件并提供一些反馈：

![reject-email](../../../../../workshops/2025-05/walkthrough/11-email-reject.png)


你应该收到另一封包含基于你反馈的更新尝试的电子邮件！

你可以继续批准这个：

![approve-email](../../../../../workshops/2025-05/walkthrough/11-email-approve.png)


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

你应该收到一封包含澄清请求的电子邮件

    Can you clarify what 'xyz' represents in this context? Is it a specific number, variable, or something else?

你可以回复类似这样的内容

    use 8 instead of xyz

你应该在 CLI 上看到类似这样的最终结果

    I have multiplied 4 and xyz, using the value 8 for xyz, resulting in 32.

作为最后一步，让我们探索使用自定义 HTML 模板来发送电子邮件


    cp ./walkthrough/11c-cli.ts src/cli.ts

首先尝试 divide：


    npx tsx src/index.ts 'can you divide 4 by 5'

你应该看到使用自定义模板的略有不同的电子邮件

![custom-template-email](../../../../../workshops/2025-05/walkthrough/11-email-custom.png)

请随意按照流程运行，然后你可以尝试根据你的喜好更新模板

（如果你正在使用 cursor，只需突出显示模板并要求"让它更好"就应该可以了）

也尝试触发 "request_more_information"！


就是这样 - 在下一章中，我们将构建一个完全由电子邮件驱动的
使用 webhooks 进行人类审批的工作流代理



# 第 XX 章 - HumanLayer Webhook 集成

前面的部分使用了 humanlayer SDK 的"同步模式"——这意味着
每次我们等待人类审批时，我们都在循环中轮询直到收到人类响应。

这显然不理想，特别是对于生产工作负载，
因此在本节中，我们将通过更新服务器在联系人类后结束处理，并使用 webhooks 接收结果来实现 [因子 6 - 通过简单 API 启动 / 暂停 / 恢复](../../../../content/factor-6-launch-pause-resume.md)。


添加在服务器中初始化 humanlayer 的代码


    cp ./walkthrough/12-1-server-init.ts src/server.ts

接下来，让我们更新 /thread 端点以

1. 异步处理请求，立即返回
2. 在 request_more_information 和 done_for_now 调用时创建人类联系


更新服务器以能够处理 request_clarification 响应

- 移除旧的 /response 端点和类型
- 更新 /thread 端点以异步运行处理，立即返回
- 在请求人类响应时发送 state.threadId
- 添加 handleHumanResponse 函数以处理人类响应
- 添加 /webhook 端点以处理 webhook 响应


    cp ./walkthrough/12a-server.ts src/server.ts

在另一个终端启动服务器

    npx tsx src/server.ts

现在服务器正在运行，向 '/thread' 端点发送有效载荷


__ 执行响应步骤

__ 现在处理 divide 的审批

__ 现在也处理 done_for_now
