# 第 1 章 - CLI 和 Agent 循环

现在让我们添加 BAML 并创建第一个带有 CLI 接口的 agent。

首先，我们需要安装 [BAML](https://github.com/boundaryml/baml)，
这是一个用于提示和结构化输出的工具。


    npm install @boundaryml/baml

初始化 BAML

    npx baml-cli init

删除默认的 resume.baml

    rm baml_src/resume.baml

添加我们的入门 agent，一个我们将在此基础上构建的单个 baml 提示

    cp ./walkthrough/01-agent.baml baml_src/agent.baml

<details>
<summary>查看文件</summary>

```rust
// ./walkthrough/01-agent.baml
class DoneForNow {
  intent "done_for_now"
  message string 
}

client<llm> Qwen3 {
  provider "openai-generic"
  options {
    base_url env.BASETEN_BASE_URL
    api_key env.BASETEN_API_KEY 
  }
}

function DetermineNextStep(
    thread: string 
) -> DoneForNow {
    client Qwen3

    // use /nothink for now because the thinking tokens (or streaming thereof) screw with baml (i think (no pun intended))
    prompt #"
        {{ _.role("system") }}

        /nothink 

        You are a helpful assistant that can help with tasks.

        {{ _.role("user") }}

        You are working on the following thread:

        {{ thread }}

        What should the next step be?

        {{ ctx.output_format }}
    "#
}

test HelloWorld {
  functions [DetermineNextStep]
  args {
    thread #"
      {
        "type": "user_input",
        "data": "hello!"
      }
    "#
  }
}
```

</details>

生成 BAML 客户端代码

    npx baml-cli generate

为本节启用 BAML 日志

    export BAML_LOG=debug

添加 CLI 接口

    cp ./walkthrough/01-cli.ts src/cli.ts

<details>
<summary>查看文件</summary>

```ts
// ./walkthrough/01-cli.ts
// cli.ts lets you invoke the agent loop from the command line

import { agentLoop, Thread, Event } from "./agent";

export async function cli() {
    // Get command line arguments, skipping the first two (node and script name)
    const args = process.argv.slice(2);

    if (args.length === 0) {
        console.error("Error: Please provide a message as a command line argument");
        process.exit(1);
    }

    // Join all arguments into a single message
    const message = args.join(" ");

    // Create a new thread with the user's message as the initial event
    const thread = new Thread([{ type: "user_input", data: message }]);

    // Run the agent loop with the thread
    const result = await agentLoop(thread);
    console.log(result);
}
```

</details>

更新 index.ts 以使用 CLI

```diff
src/index.ts
+import { cli } from "./cli"
+
 async function hello(): Promise<void> {
     console.log('hello, world!')
 
 async function main() {
-    await hello()
+    await cli()
 }
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/01-index.ts src/index.ts

</details>

添加 agent 实现

    cp ./walkthrough/01-agent.ts src/agent.ts

<details>
<summary>查看文件</summary>

```ts
// ./walkthrough/01-agent.ts
import { b } from "../baml_client";

// tool call or a respond to human tool
type AgentResponse = Awaited<ReturnType<typeof b.DetermineNextStep>>;

export interface Event {
    type: string
    data: any;
}

export class Thread {
    events: Event[] = [];

    constructor(events: Event[]) {
        this.events = events;
    }

    serializeForLLM() {
        // can change this to whatever custom serialization you want to do, XML, etc
        // e.g. https://github.com/got-agents/agents/blob/59ebbfa236fc376618f16ee08eb0f3bf7b698892/linear-assistant-ts/src/agent.ts#L66-L105
        return JSON.stringify(this.events);
    }
}

// right now this just runs one turn with the LLM, but
// we'll update this function to handle all the agent logic
export async function agentLoop(thread: Thread): Promise<AgentResponse> {
    const nextStep = await b.DetermineNextStep(thread.serializeForLLM());
    return nextStep;
}
```

</details>

BAML 代码默认配置为使用 BASETEN_API_KEY

要获取 Baseten API 密钥和 URL，请在 [baseten.co](https://baseten.co) 创建一个账户，
然后从模型库中部署 [Qwen3 32B](https://www.baseten.co/library/qwen-3-32b/)。

```rust 
  function DetermineNextStep(thread: string) -> DoneForNow {
      client Qwen3
      // ...
```

如果你想不作任何更改直接运行示例，可以将 BASETEN_API_KEY 环境变量设置为任何有效的 baseten 密钥。

如果你想尝试更换模型，可以更改 `client` 行。

[BAML 客户端文档可在此处找到](https://docs.boundaryml.com/guide/baml-basics/switching-llms)

例如，你可以配置 [gemini](https://docs.boundaryml.com/ref/llm-client-providers/google-ai-gemini) 
或 [anthropic](https://docs.boundaryml.com/ref/llm-client-providers/anthropic) 作为你的模型提供商。

例如，要使用 openai 和 OPENAI_API_KEY，你可以这样做：

    client "openai/gpt-4o"


设置环境变量

    export BASETEN_API_KEY=...
    export BASETEN_BASE_URL=...

尝试一下

    npx tsx src/index.ts hello

你应该看到来自模型的熟悉响应

    {
      intent: 'done_for_now',
      message: 'Hello! How can I assist you today?'
    }

