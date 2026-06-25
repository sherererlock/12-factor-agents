# 从零开始构建 12-factor agent 模板

从一个空白的 TS 仓库开始，逐步构建一个 12-factor agent。本教程将引导你创建一个遵循 12-factor 方法论的 TypeScript agent。

## 清理

确保从一个干净的状态开始

清理现有文件

    rm -rf baml_src/ && rm -rf src/

## 第 0 章 - Hello World

让我们从一个基本的 TypeScript 设置和一个 hello world 程序开始。

本指南使用 TypeScript 编写（是的，Python 版本即将推出）

在工作坊的每个文件编辑步骤之间有许多检查点，
因此即使你对 TypeScript 不太熟悉，
也应该能够跟上并运行每个示例。

要运行本指南，你需要安装相对较新版本的 Node.js 和 npm

你可以使用任何你喜欢的 Node.js 版本管理器，[homebrew](https://formulae.brew.sh/formula/node) 就可以


    brew install node@20

你应该能看到 node 版本

    node --version

复制初始 package.json

    cp ./walkthrough/00-package.json package.json

<details>
<summary>查看文件</summary>

```json
// ./walkthrough/00-package.json
{
    "name": "my-agent",
    "version": "0.1.0",
    "private": true,
    "scripts": {
      "dev": "tsx src/index.ts",
      "build": "tsc"
    },
    "dependencies": {
      "tsx": "^4.15.0",
      "typescript": "^5.0.0"
    },
    "devDependencies": {
      "@types/node": "^20.0.0",
      "@typescript-eslint/eslint-plugin": "^6.0.0",
      "@typescript-eslint/parser": "^6.0.0",
      "eslint": "^8.0.0"
    }
  }
```

</details>

安装依赖

    npm install

复制 tsconfig.json

    cp ./walkthrough/00-tsconfig.json tsconfig.json

<details>
<summary>查看文件</summary>

```json
// ./walkthrough/00-tsconfig.json
{
    "compilerOptions": {
      "target": "ES2017",
      "lib": ["esnext"],
      "allowJs": true,
      "skipLibCheck": true,
      "strict": true,
      "noEmit": true,
      "esModuleInterop": true,
      "module": "esnext",
      "moduleResolution": "bundler",
      "resolveJsonModule": true,
      "isolatedModules": true,
      "jsx": "preserve",
      "incremental": true,
      "plugins": [],
      "paths": {
        "@/*": ["./*"]
      }
    },
    "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
    "exclude": ["node_modules", "walkthrough"]
  }
```

</details>

添加 .gitignore

    cp ./walkthrough/00-.gitignore .gitignore

<details>
<summary>查看文件</summary>

```gitignore
// ./walkthrough/00-.gitignore
baml_client/
node_modules/
```

</details>

创建 src 文件夹

添加一个简单的 hello world index.ts

    cp ./walkthrough/00-index.ts src/index.ts

<details>
<summary>查看文件</summary>

```ts
// ./walkthrough/00-index.ts
async function hello(): Promise<void> {
    console.log('hello, world!')
}

async function main() {
    await hello()
}

main().catch(console.error)
```

</details>

运行以验证

    npx tsx src/index.ts

你应该看到：

    hello, world!

## 第 1 章 - CLI 和 Agent 循环

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

function DetermineNextStep(
    thread: string 
) -> DoneForNow {
    client "openai/gpt-4o"

    prompt #"
        {{ _.role("system") }}

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

BAML 代码默认配置为使用 OPENAI_API_KEY

在测试时，你可以随意更改模型/提供商

        client "openai/gpt-4o"

[BAML 客户端文档可在此处找到](https://docs.boundaryml.com/guide/baml-basics/switching-llms)

例如，你可以配置 [gemini](https://docs.boundaryml.com/ref/llm-client-providers/google-ai-gemini) 
或 [anthropic](https://docs.boundaryml.com/ref/llm-client-providers/anthropic) 作为你的模型提供商。

如果你想不作任何更改直接运行示例，可以将 OPENAI_API_KEY 环境变量设置为任何有效的 openai 密钥。


    export OPENAI_API_KEY=...

尝试一下

    npx tsx src/index.ts hello

你应该看到来自模型的熟悉响应

    {
  intent: 'done_for_now',
  message: 'Hello! How can I assist you today?'
}

## 第 2 章 - 添加计算器工具

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
     client "openai/gpt-4o"
 
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

## 第 3 章 - 在循环中处理工具调用

现在让我们添加一个真正的 agent 循环，它可以运行工具并从 LLM 获取最终答案。

首先，让我们更新 agent 以处理工具调用


```diff
src/agent.ts
 }
 
-// right now this just runs one turn with the LLM, but
-// we'll update this function to handle all the agent logic
-export async function agentLoop(thread: Thread): Promise<AgentResponse> {
-    const nextStep = await b.DetermineNextStep(thread.serializeForLLM());
-    return nextStep;
+
+
+export async function agentLoop(thread: Thread): Promise<string> {
+
+    while (true) {
+        const nextStep = await b.DetermineNextStep(thread.serializeForLLM());
+        console.log("nextStep", nextStep);
+
+        switch (nextStep.intent) {
+            case "done_for_now":
+                // response to human, return the next step object
+                return nextStep.message;
+            case "add":
+                thread.events.push({
+                    "type": "tool_call",
+                    "data": nextStep
+                });
+                const result = nextStep.a + nextStep.b;
+                console.log("tool_response", result);
+                thread.events.push({
+                    "type": "tool_response",
+                    "data": result
+                });
+                continue;
+            default:
+                throw new Error(`Unknown intent: ${nextStep.intent}`);
+        }
+    }
 }
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/03-agent.ts src/agent.ts

</details>

现在，让我们试一试


    npx tsx src/index.ts 'can you add 3 and 4'

你应该看到 agent 调用工具然后返回结果

    {
  intent: 'done_for_now',
  message: 'The sum of 3 and 4 is 7.'
}

对于下一步，我们将进行更复杂的计算，让我们关闭 baml 日志以获得更简洁的输出

    export BAML_LOG=off

尝试多步计算

    npx tsx src/index.ts 'can you add 3 and 4, then add 6 to that result'

你会注意到乘法和除法等工具不可用

    npx tsx src/index.ts 'can you multiply 3 and 4'

接下来，让我们为其余计算器工具添加处理程序


```diff
src/agent.ts
-import { b } from "../baml_client";
+import { AddTool, SubtractTool, DivideTool, MultiplyTool, b } from "../baml_client";
 
-// tool call or a respond to human tool
-type AgentResponse = Awaited<ReturnType<typeof b.DetermineNextStep>>;
-
 export interface Event {
     type: string
 }
 
+export type CalculatorTool = AddTool | SubtractTool | MultiplyTool | DivideTool;
 
+export async function handleNextStep(nextStep: CalculatorTool, thread: Thread): Promise<Thread> {
+    let result: number;
+    switch (nextStep.intent) {
+        case "add":
+            result = nextStep.a + nextStep.b;
+            console.log("tool_response", result);
+            thread.events.push({
+                "type": "tool_response",
+                "data": result
+            });
+            return thread;
+        case "subtract":
+            result = nextStep.a - nextStep.b;
+            console.log("tool_response", result);
+            thread.events.push({
+                "type": "tool_response",
+                "data": result
+            });
+            return thread;
+        case "multiply":
+            result = nextStep.a * nextStep.b;
+            console.log("tool_response", result);
+            thread.events.push({
+                "type": "tool_response",
+                "data": result
+            });
+            return thread;
+        case "divide":
+            result = nextStep.a / nextStep.b;
+            console.log("tool_response", result);
+            thread.events.push({
+                "type": "tool_response",
+                "data": result
+            });
+            return thread;
+    }
+}
 
 export async function agentLoop(thread: Thread): Promise<string> {
         console.log("nextStep", nextStep);
 
+        thread.events.push({
+            "type": "tool_call",
+            "data": nextStep
+        });
+
         switch (nextStep.intent) {
             case "done_for_now":
                 return nextStep.message;
             case "add":
-                thread.events.push({
-                    "type": "tool_call",
-                    "data": nextStep
-                });
-                const result = nextStep.a + nextStep.b;
-                console.log("tool_response", result);
-                thread.events.push({
-                    "type": "tool_response",
-                    "data": result
-                });
-                continue;
-            default:
-                throw new Error(`Unknown intent: ${nextStep.intent}`);
+            case "subtract":
+            case "multiply":
+            case "divide":
+                thread = await handleNextStep(nextStep, thread);
         }
     }
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/03b-agent.ts src/agent.ts

</details>

测试减法

    npx tsx src/index.ts 'can you subtract 3 from 4'

现在，让我们测试乘法工具


    npx tsx src/index.ts 'can you multiply 3 and 4'

最后，让我们测试一个包含多个运算的更复杂计算


    npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

## 第 4 章 - 为 agent.baml 添加测试

让我们为 BAML agent 添加一些测试。

首先，保持 baml 日志启用

    export BAML_LOG=debug

接下来，让我们为 agent 添加一些测试

我们将从一个简单的测试开始，检查 agent 处理
基本计算的能力。


```diff
baml_src/agent.baml
     "#
   }
+
+test MathOperation {
+  functions [DetermineNextStep]
+  args {
+    thread #"
+      {
+        "type": "user_input",
+        "data": "can you multiply 3 and 4?"
+      }
+    "#
+  }
+}
+
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/04-agent.baml baml_src/agent.baml

</details>

运行测试

    npx baml-cli test

现在，让我们用断言来改进测试！

断言是确保 agent 按预期工作的好方法，
并且可以轻松扩展以检查更复杂的行为。


```diff
baml_src/agent.baml
     "#
   }
+  @@assert(hello, {{this.intent == "done_for_now"}})
 }
 
     "#
   }
+  @@assert(math_operation, {{this.intent == "multiply"}})
 }
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/04b-agent.baml baml_src/agent.baml

</details>

运行测试

    npx baml-cli test

随着你添加更多测试，你可以禁用日志以保持输出整洁。
你可能希望在迭代特定测试时启用它们。


    export BAML_LOG=off

现在，让我们添加一些更复杂的测试用例，
我们从正在进行的 agent 上下文窗口中间恢复


```diff
baml_src/agent.baml
     "#
   }
-  @@assert(hello, {{this.intent == "done_for_now"}})
+  @@assert(intent, {{this.intent == "done_for_now"}})
 }
 
     "#
   }
-  @@assert(math_operation, {{this.intent == "multiply"}})
+  @@assert(intent, {{this.intent == "multiply"}})
 }
 
+test LongMath {
+  functions [DetermineNextStep]
+  args {
+    thread #"
+      [
+        {
+          "type": "user_input",
+          "data": "can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result?"
+        },
+        {
+          "type": "tool_call",
+          "data": {
+            "intent": "multiply",
+            "a": 3,
+            "b": 4
+          }
+        },
+        {
+          "type": "tool_response",
+          "data": 12
+        },
+        {
+          "type": "tool_call", 
+          "data": {
+            "intent": "divide",
+            "a": 12,
+            "b": 2
+          }
+        },
+        {
+          "type": "tool_response",
+          "data": 6
+        },
+        {
+          "type": "tool_call",
+          "data": {
+            "intent": "add", 
+            "a": 6,
+            "b": 12
+          }
+        },
+        {
+          "type": "tool_response",
+          "data": 18
+        }
+      ]
+    "#
+  }
+  @@assert(intent, {{this.intent == "done_for_now"}})
+  @@assert(answer, {{"18" in this.message}})
+}
+
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/04c-agent.baml baml_src/agent.baml

</details>

让我们试着运行它


    npx baml-cli test

## 第 5 章 - 多个人类工具

在本节中，我们将添加对多个用于
联系人类的工具的支持。


在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

首先，让我们添加一个可以向人类请求澄清的工具

这将与 "done_for_now" 工具不同，
可以用来更灵活地处理 agent 中不同类型的人类交互。


```diff
baml_src/agent.baml
+// human tools are async requests to a human
+type HumanTools = ClarificationRequest | DoneForNow
+
+class ClarificationRequest {
+  intent "request_more_information" @description("you can request more information from me")
+  message string
+}
+
 class DoneForNow {
   intent "done_for_now"
-  message string 
+
+  message string @description(#"
+    message to send to the user about the work that was done. 
+  "#)
 }
 
 function DetermineNextStep(
     thread: string 
-) -> CalculatorTools | DoneForNow {
+) -> HumanTools | CalculatorTools {
     client "openai/gpt-4o"
 
 }
 
+
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/05-agent.baml baml_src/agent.baml

</details>

接下来，让我们重新生成客户端代码

注意 - 如果你使用的是 VSCode 的 BAML 扩展，
当你在编辑器中保存文件时，客户端会自动重新生成。


    npx baml-cli generate

现在，让我们更新 agent 以使用新工具


```diff
src/agent.ts
 }
 
-export async function agentLoop(thread: Thread): Promise<string> {
+export async function agentLoop(thread: Thread): Promise<Thread> {
 
     while (true) {
         switch (nextStep.intent) {
             case "done_for_now":
-                // response to human, return the next step object
-                return nextStep.message;
+            case "request_more_information":
+                // response to human, return the thread
+                return thread;
             case "add":
             case "subtract":
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/05-agent.ts src/agent.ts

</details>

接下来，让我们更新 CLI 以处理澄清请求，
通过在 CLI 上请求用户输入


```diff
src/cli.ts
 // cli.ts lets you invoke the agent loop from the command line
 
-import { agentLoop, Thread, Event } from "./agent";
+import { agentLoop, Thread, Event } from "../src/agent";
+
+
 
 export async function cli() {
     // Get command line arguments, skipping the first two (node and script name)
     // Run the agent loop with the thread
     const result = await agentLoop(thread);
-    console.log(result);
+    let lastEvent = result.events.slice(-1)[0];
+
+    while (lastEvent.data.intent === "request_more_information") {
+        const message = await askHuman(lastEvent.data.message);
+        thread.events.push({ type: "human_response", data: message });
+        const result = await agentLoop(thread);
+        lastEvent = result.events.slice(-1)[0];
+    }
+
+    // print the final result
+    // optional - you could loop here too
+    console.log(lastEvent.data.message);
+    process.exit(0);
 }
+
+async function askHuman(message: string) {
+    const readline = require('readline').createInterface({
+        input: process.stdin,
+        output: process.stdout
+    });
+
+    return new Promise((resolve) => {
+        readline.question(`${message}\n> `, (answer: string) => {
+            resolve(answer);
+        });
+    });
+}
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/05-cli.ts src/cli.ts

</details>

让我们试一试


    npx tsx src/index.ts 'can you multiply 3 and FD*(#F&& '

接下来，让我们添加一个测试，检查 agent 处理
澄清请求的能力


```diff
baml_src/agent.baml
 
 
+
+test MathOperationWithClarification {
+  functions [DetermineNextStep]
+  args {
+    thread #"
+          [{"type":"user_input","data":"can you multiply 3 and feee9ff10"}]
+      "#
+  }
+  @@assert(intent, {{this.intent == "request_more_information"}})
+}
+
+test MathOperationPostClarification {
+  functions [DetermineNextStep]
+  args {
+    thread #"
+        [
+        {"type":"user_input","data":"can you multiply 3 and FD*(#F&& ?"},
+        {"type":"tool_call","data":{"intent":"request_more_information","message":"It seems like there was a typo or mistake in your request. Could you please clarify or provide the correct numbers you would like to multiply?"}},
+        {"type":"human_response","data":"lets try 12 instead"},
+      ]
+      "#
+  }
+  @@assert(intent, {{this.intent == "multiply"}})
+  @@assert(a, {{this.b == 12}})
+  @@assert(b, {{this.a == 3}})
+}
+        
+
+
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/05b-agent.baml baml_src/agent.baml

</details>

现在我们可以再次运行测试了


    npx baml-cli test

你会注意到新测试通过了，但 hello world 测试失败了

这是因为 agent 的默认行为是返回 "done_for_now"


```diff
baml_src/agent.baml
     "#
   }
-  @@assert(intent, {{this.intent == "done_for_now"}})
+  @@assert(intent, {{this.intent == "request_more_information"}})
 }
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/05c-agent.baml baml_src/agent.baml

</details>

验证测试通过

    npx baml-cli test

## 第 6 章 - 使用推理自定义你的提示

在本节中，我们将探索如何使用推理步骤自定义 agent 的提示。

这是 [factor 2 - 拥有你的提示](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-2-own-your-prompts.md) 的核心

AI That Works 上有一篇关于推理的深入文章 [推理模型与推理步骤](https://github.com/hellovai/ai-that-works/tree/main/2025-04-07-reasoning-models-vs-prompts)


在本节中，保持 baml 日志启用会很有帮助

    export BAML_LOG=debug

更新 agent 提示以包含推理步骤


```diff
baml_src/agent.baml
 
         {{ ctx.output_format }}
+
+        First, always plan out what to do next, for example:
+
+        - ...
+        - ...
+        - ...
+
+        {...} // schema
     "#
 }
   @@assert(b, {{this.a == 3}})
 }
-        
-
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/06-agent.baml baml_src/agent.baml

</details>

生成更新的客户端

    npx baml-cli generate

现在，你可以用一个简单的提示来尝试一下


    npx tsx src/index.ts 'can you multiply 3 and 4'

你应该看到来自 baml 日志的输出显示了推理步骤

#### 可选挑战

在你的工具输出格式中添加一个字段，在输出中包含推理步骤！


## 第 7 章 - 自定义你的上下文窗口

在本节中，我们将探索如何自定义 agent 的上下文窗口。

这是 [factor 3 - 拥有你的上下文窗口](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-3-own-your-context-window.md) 的核心


更新 agent 以美化打印模型的上下文窗口


```diff
src/agent.ts
         // can change this to whatever custom serialization you want to do, XML, etc
         // e.g. https://github.com/got-agents/agents/blob/59ebbfa236fc376618f16ee08eb0f3bf7b698892/linear-assistant-ts/src/agent.ts#L66-L105
-        return JSON.stringify(this.events);
+        return JSON.stringify(this.events, null, 2);
     }
 }
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/07-agent.ts src/agent.ts

</details>

测试格式化

    BAML_LOG=info npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

接下来，让我们更新 agent 以使用 XML 格式

这是一种非常流行的数据传递格式，

除其他优点外，还因为 XML 的 token 效率。


```diff
src/agent.ts
 
     serializeForLLM() {
-        // can change this to whatever custom serialization you want to do, XML, etc
-        // e.g. https://github.com/got-agents/agents/blob/59ebbfa236fc376618f16ee08eb0f3bf7b698892/linear-assistant-ts/src/agent.ts#L66-L105
-        return JSON.stringify(this.events, null, 2);
+        return this.events.map(e => this.serializeOneEvent(e)).join("\n");
     }
+
+    trimLeadingWhitespace(s: string) {
+        return s.replace(/^[ \t]+/gm, '');
+    }
+
+    serializeOneEvent(e: Event) {
+        return this.trimLeadingWhitespace(`
+            <${e.data?.intent || e.type}>
+            ${
+            typeof e.data !== 'object' ? e.data :
+            Object.keys(e.data).filter(k => k !== 'intent').map(k => `${k}: ${e.data[k]}`).join("\n")}
+            </${e.data?.intent || e.type}>
+        `)
+    }
 }
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/07b-agent.ts src/agent.ts

</details>

让我们试一试


    BAML_LOG=info npx tsx src/index.ts 'can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result'

让我们更新测试以匹配新的输出格式


```diff
baml_src/agent.baml
         {{ ctx.output_format }}
 
-        First, always plan out what to do next, for example:
+        Always think about what to do next first, like:
 
         - ...
   args {
     thread #"
-      {
-        "type": "user_input",
-        "data": "hello!"
-      }
+      <user_input>
+        hello!
+      </user_input>
     "#
   }
   args {
     thread #"
-      {
-        "type": "user_input",
-        "data": "can you multiply 3 and 4?"
-      }
+      <user_input>
+        can you multiply 3 and 4?
+      </user_input>
     "#
   }
   args {
     thread #"
-      [
-        {
-          "type": "user_input",
-          "data": "can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result?"
-        },
-        {
-          "type": "tool_call",
-          "data": {
-            "intent": "multiply",
-            "a": 3,
-            "b": 4
-          }
-        },
-        {
-          "type": "tool_response",
-          "data": 12
-        },
-        {
-          "type": "tool_call", 
-          "data": {
-            "intent": "divide",
-            "a": 12,
-            "b": 2
-          }
-        },
-        {
-          "type": "tool_response",
-          "data": 6
-        },
-        {
-          "type": "tool_call",
-          "data": {
-            "intent": "add", 
-            "a": 6,
-            "b": 12
-          }
-        },
-        {
-          "type": "tool_response",
-          "data": 18
-        }
-      ]
+         <user_input>
+    can you multiply 3 and 4, then divide the result by 2 and then add 12 to that result?
+    </user_input>
+
+
+    <multiply>
+    a: 3
+    b: 4
+    </multiply>
+
+
+    <tool_call>
+    12
+    </tool_call>
+
+
+    <divide>
+    a: 12
+    b: 2
+    </divide>
+
+
+    <tool_call>
+    6
+    </tool_call>
+
+
+    <add>
+    a: 6
+    b: 12
+    </add>
+
+
+    <tool_call>
+    18
+    </tool_call>
+
     "#
   }
   args {
     thread #"
-          [{"type":"user_input","data":"can you multiply 3 and feee9ff10"}]
+          <user_input>
+          can you multiply 3 and fe1iiaff10
+          </user_input>
       "#
   }
   args {
     thread #"
-        [
-        {"type":"user_input","data":"can you multiply 3 and FD*(#F&& ?"},
-        {"type":"tool_call","data":{"intent":"request_more_information","message":"It seems like there was a typo or mistake in your request. Could you please clarify or provide the correct numbers you would like to multiply?"}},
-        {"type":"human_response","data":"lets try 12 instead"},
-      ]
+        <user_input>
+        can you multiply 3 and FD*(#F&& ?
+        </user_input>
+
+        <request_more_information>
+        message: It seems like there was a typo or mistake in your request. Could you please clarify or provide the correct numbers you would like to multiply?
+        </request_more_information>
+
+        <human_response>
+        lets try 12 instead
+        </human_response>
       "#
   }
   @@assert(intent, {{this.intent == "multiply"}})
 }
         
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/07c-agent.baml baml_src/agent.baml

</details>

查看更新后的测试


    npx baml-cli test

## 第 8 章 - 添加 API 端点

添加一个 Express 服务器以通过 HTTP 暴露 agent。

在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

安装 Express 和类型

    npm install express && npm install --save-dev @types/express supertest

添加服务器实现

    cp ./walkthrough/08-server.ts src/server.ts

<details>
<summary>查看文件</summary>

```ts
// ./walkthrough/08-server.ts
import express from 'express';
import { Thread, agentLoop } from '../src/agent';

const app = express();
app.use(express.json());
app.set('json spaces', 2);

// POST /thread - Start new thread
app.post('/thread', async (req, res) => {
    const thread = new Thread([{
        type: "user_input",
        data: req.body.message
    }]);
    const result = await agentLoop(thread);
    res.json(result);
});

// GET /thread/:id - Get thread status 
app.get('/thread/:id', (req, res) => {
    // optional - add state
    res.status(404).json({ error: "Not implemented yet" });
});

const port = process.env.PORT || 3000;
app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});

export { app };
```

</details>

启动服务器

    npx tsx src/server.ts

用 curl 测试（在另一个终端）

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you add 3 and 4"}'

你应该得到一个来自 agent 的响应，其中包含
agent 跟踪，以如下消息结尾： 

    {"intent":"done_for_now","message":"The sum of 3 and 4 is 7."}

## 第 9 章 - 内存状态和异步澄清

添加状态管理和异步澄清支持。

在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

为线程添加一些简单的内存状态管理

    cp ./walkthrough/09-state.ts src/state.ts

<details>
<summary>查看文件</summary>

```ts
// ./walkthrough/09-state.ts
import crypto from 'crypto';
import { Thread } from '../src/agent';


// you can replace this with any simple state management,
// e.g. redis, sqlite, postgres, etc
export class ThreadStore {
    private threads: Map<string, Thread> = new Map();
    
    create(thread: Thread): string {
        const id = crypto.randomUUID();
        this.threads.set(id, thread);
        return id;
    }
    
    get(id: string): Thread | undefined {
        return this.threads.get(id);
    }
    
    update(id: string, thread: Thread): void {
        this.threads.set(id, thread);
    }
}
```

</details>

更新服务器以使用状态管理

* 使用 `ThreadStore` 添加线程状态管理
* 从 /thread 端点返回线程 ID 和响应 URL
* 实现 GET /thread/:id 
* 实现 POST /thread/:id/response


```diff
src/server.ts
 import express from 'express';
 import { Thread, agentLoop } from '../src/agent';
+import { ThreadStore } from '../src/state';
 
 const app = express();
 app.set('json spaces', 2);
 
+const store = new ThreadStore();
+
 // POST /thread - Start new thread
 app.post('/thread', async (req, res) => {
         data: req.body.message
     }]);
-    const result = await agentLoop(thread);
-    res.json(result);
+    
+    const threadId = store.create(thread);
+    const newThread = await agentLoop(thread);
+    
+    store.update(threadId, newThread);
+
+    const lastEvent = newThread.events[newThread.events.length - 1];
+    // If we exited the loop, include the response URL so the client can
+    // push a new message onto the thread
+    lastEvent.data.response_url = `/thread/${threadId}/response`;
+
+    console.log("returning last event from endpoint", lastEvent);
+
+    res.json({ 
+        thread_id: threadId,
+        ...newThread 
+    });
 });
 
 app.get('/thread/:id', (req, res) => {
-    // optional - add state
-    res.status(404).json({ error: "Not implemented yet" });
+    const thread = store.get(req.params.id);
+    if (!thread) {
+        return res.status(404).json({ error: "Thread not found" });
+    }
+    res.json(thread);
 });
 
+// POST /thread/:id/response - Handle clarification response
+app.post('/thread/:id/response', async (req, res) => {
+    let thread = store.get(req.params.id);
+    if (!thread) {
+        return res.status(404).json({ error: "Thread not found" });
+    }
+    
+    thread.events.push({
+        type: "human_response",
+        data: req.body.message
+    });
+    
+    // loop until stop event
+    const newThread = await agentLoop(thread);
+    
+    store.update(req.params.id, newThread);
+
+    const lastEvent = newThread.events[newThread.events.length - 1];
+    lastEvent.data.response_url = `/thread/${req.params.id}/response`;
+
+    console.log("returning last event from endpoint", lastEvent);
+    
+    res.json(newThread);
+});
+
 const port = process.env.PORT || 3000;
 app.listen(port, () => {
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/09-server.ts src/server.ts

</details>

启动服务器

    npx tsx src/server.ts

测试澄清流程

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you multiply 3 and xyz"}'

## 第 10 章 - 添加人类审批

添加对操作的人类审批支持。

在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

更新服务器以处理人类审批

* 导入 `handleNextStep` 以执行已批准的操作
* 添加两种负载类型以区分审批和响应
* 在端点中分别处理响应和审批
* 当出现问题时显示更好的错误消息


```diff
src/server.ts
 import express from 'express';
-import { Thread, agentLoop } from '../src/agent';
+import { Thread, agentLoop, handleNextStep } from '../src/agent';
 import { ThreadStore } from '../src/state';
 
 });
 
+
+type ApprovalPayload = {
+    type: "approval";
+    approved: boolean;
+    comment?: string;
+}
+
+type ResponsePayload = {
+    type: "response";
+    response: string;
+}
+
+type Payload = ApprovalPayload | ResponsePayload;
+
 // POST /thread/:id/response - Handle clarification response
 app.post('/thread/:id/response', async (req, res) => {
         return res.status(404).json({ error: "Thread not found" });
     }
+
+    const body: Payload = req.body;
+
+    let lastEvent = thread.events[thread.events.length - 1];
+
+    if (thread.awaitingHumanResponse() && body.type === 'response') {
+        thread.events.push({
+            type: "human_response",
+            data: body.response
+        });
+    } else if (thread.awaitingHumanApproval() && body.type === 'approval' && !body.approved) {
+        // push feedback onto the thread
+        thread.events.push({
+            type: "tool_response",
+            data: `user denied the operation with feedback: "${body.comment}"`
+        });
+    } else if (thread.awaitingHumanApproval() && body.type === 'approval' && body.approved) {
+        // approved, run the tool, pushing results onto the thread
+        await handleNextStep(lastEvent.data, thread);
+    } else {
+        res.status(400).json({
+            error: "Invalid request: " + body.type,
+            awaitingHumanResponse: thread.awaitingHumanResponse(),
+            awaitingHumanApproval: thread.awaitingHumanApproval()
+        });
+        return;
+    }
+
     
-    thread.events.push({
-        type: "human_response",
-        data: req.body.message
-    });
-    
     // loop until stop event
     const newThread = await agentLoop(thread);
     store.update(req.params.id, newThread);
 
-    const lastEvent = newThread.events[newThread.events.length - 1];
+    lastEvent = newThread.events[newThread.events.length - 1];
     lastEvent.data.response_url = `/thread/${req.params.id}/response`;
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/10-server.ts src/server.ts

</details>

为 agent 添加一些方法以处理审批和响应

```diff
src/agent.ts
         `)
     }
+
+    awaitingHumanResponse(): boolean {
+        const lastEvent = this.events[this.events.length - 1];
+        return ['request_more_information', 'done_for_now'].includes(lastEvent.data.intent);
+    }
+
+    awaitingHumanApproval(): boolean {
+        const lastEvent = this.events[this.events.length - 1];
+        return lastEvent.data.intent === 'divide';
+    }
 }
 
                 // response to human, return the thread
                 return thread;
+            case "divide":
+                // divide is scary, return it for human approval
+                return thread;
             case "add":
             case "subtract":
             case "multiply":
-            case "divide":
                 thread = await handleNextStep(nextStep, thread);
         }
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/10-agent.ts src/agent.ts

</details>

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

现在你可以批准该操作了

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

## 第 11 章 - 通过邮件进行人类审批

在本节中，我们将添加通过邮件进行人类审批的支持。

这开始会有点刻意，只是为了掌握概念 - 

我们将从 CLI 调用工作流开始，但 `divide` 和 `request_more_information` 的审批将通过邮件处理，
然后最终的 `done_for_now` 答案将打印回 CLI

虽然有点刻意，但这是你从
[factor 7 - 使用工具联系人类](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-7-contact-humans-with-tools.md) 获得灵活性的绝佳示例


在本节中，我们将禁用 baml 日志。如果你想查看更多细节，可以选择启用它们。

    export BAML_LOG=off

安装 HumanLayer

    npm install humanlayer

更新 CLI 以通过邮件将 `divide` 和 `request_more_information` 发送给人类

```diff
src/cli.ts
 // cli.ts lets you invoke the agent loop from the command line
 
+import { humanlayer } from "humanlayer";
 import { agentLoop, Thread, Event } from "../src/agent";
 
-
-
 export async function cli() {
     // Get command line arguments, skipping the first two (node and script name)
 
     // Run the agent loop with the thread
-    const result = await agentLoop(thread);
-    let lastEvent = result.events.slice(-1)[0];
+    let newThread = await agentLoop(thread);
+    let lastEvent = newThread.events.slice(-1)[0];
 
-    while (lastEvent.data.intent === "request_more_information") {
-        const message = await askHuman(lastEvent.data.message);
-        thread.events.push({ type: "human_response", data: message });
-        const result = await agentLoop(thread);
-        lastEvent = result.events.slice(-1)[0];
+    while (lastEvent.data.intent !== "done_for_now") {
+        const responseEvent = await askHuman(lastEvent);
+        thread.events.push(responseEvent);
+        newThread = await agentLoop(thread);
+        lastEvent = newThread.events.slice(-1)[0];
     }
 
     // print the final result
     console.log(lastEvent.data.message);
     process.exit(0);
 }
 
-async function askHuman(message: string) {
+async function askHuman(lastEvent: Event): Promise<Event> {
+    if (process.env.HUMANLAYER_API_KEY) {
+        return await askHumanEmail(lastEvent);
+    } else {
+        return await askHumanCLI(lastEvent.data.message);
+    }
+}
+
+async function askHumanCLI(message: string): Promise<Event> {
     const readline = require('readline').createInterface({
         input: process.stdin,
     return new Promise((resolve) => {
         readline.question(`${message}\n> `, (answer: string) => {
-            resolve(answer);
+            resolve({ type: "human_response", data: answer });
         });
     });
 }
+
+export async function askHumanEmail(lastEvent: Event): Promise<Event> {
+    if (!process.env.HUMANLAYER_EMAIL) {
+        throw new Error("missing or invalid parameters: HUMANLAYER_EMAIL");
+    }
+    const hl = humanlayer({ //reads apiKey from env
+        // name of this agent
+        runId: "12fa-cli-agent",
+        verbose: true,
+        contactChannel: {
+            // agent should request permission via email
+            email: {
+                address: process.env.HUMANLAYER_EMAIL,
+            }
+        }
+    }) 
+
+    if (lastEvent.data.intent === "divide") {
+        // fetch approval synchronously - this will block until reply
+        const response = await hl.fetchHumanApproval({
+            spec: {
+                fn: "divide",
+                kwargs: {
+                    a: lastEvent.data.a,
+                    b: lastEvent.data.b
+                }
+            }
+        })
+
+        if (response.approved) {
+            const result = lastEvent.data.a / lastEvent.data.b;
+            console.log("tool_response", result);
+            return {
+                "type": "tool_response",
+                "data": result
+            };
+        } else {
+            return {
+                "type": "tool_response",
+                "data": `user denied operation ${lastEvent.data.intent}
+                with feedback: ${response.comment}`
+            };
+        }
+    }
+    throw new Error(`unknown tool: ${lastEvent.data.intent}`)
+}
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/11-cli.ts src/cli.ts

</details>

运行 CLI

    npx tsx src/index.ts 'can you divide 4 by 5'

你的程序最后一行应该提到人类审查步骤

    nextStep { intent: 'divide', a: 4, b: 5 }
HumanLayer: Requested human approval from HumanLayer cloud

去回复邮件并提供一些反馈：

![reject-email](https://github.com/humanlayer/12-factor-agents/blob/main/workshops/2025-05/walkthrough/11-email-reject.png?raw=true)


你应该收到另一封包含基于你反馈的更新尝试的邮件！

你可以继续批准这个：

![approve-email](https://github.com/humanlayer/12-factor-agents/blob/main/workshops/2025-05/walkthrough/11-email-approve.png?raw=true)


你的最终输出将如下所示

    nextStep {
 intent: 'done_for_now',
 message: 'The division of 4 by 5 is 0.8. If you have any other calculations or questions, feel free to ask!'
}
The division of 4 by 5 is 0.8. If you have any other calculations or questions, feel free to ask!

让我们也实现 `request_more_information` 流程


```diff
src/cli.ts
     }) 
 
+    if (lastEvent.data.intent === "request_more_information") {
+        // fetch response synchronously - this will block until reply
+        const response = await hl.fetchHumanResponse({
+            spec: {
+                msg: lastEvent.data.message
+            }
+        })
+        return {
+            "type": "tool_response",
+            "data": response
+        }
+    }
+    
     if (lastEvent.data.intent === "divide") {
         // fetch approval synchronously - this will block until reply
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/11b-cli.ts src/cli.ts

</details>

让我们通过请求一个带有乱码输入的计算来测试 require_approval 流程：


    npx tsx src/index.ts 'can you multiply 4 and xyz'

你应该收到一封包含澄清请求的邮件

    Can you clarify what 'xyz' represents in this context? Is it a specific number, variable, or something else?

你可以回复类似这样的内容

    use 8 instead of xyz

你应该在 CLI 上看到类似这样的最终结果

    I have multiplied 4 and xyz, using the value 8 for xyz, resulting in 32.

作为最后一步，让我们探索使用自定义 HTML 模板来发送邮件


```diff
src/cli.ts
             email: {
                 address: process.env.HUMANLAYER_EMAIL,
+                // custom email body - jinja
+                template: `{% if type == 'request_more_information' %}
+{{ event.spec.msg }}
+{% else %}
+agent {{ event.run_id }} is requesting approval for {{event.spec.fn}}
+with args: {{event.spec.kwargs}}
+<br><br>
+reply to this email to approve
+{% endif %}`
             }
         }
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/11c-cli.ts src/cli.ts

</details>

先用除法试试：


    npx tsx src/index.ts 'can you divide 4 by 5'

你应该看到使用自定义模板的略有不同的邮件

![custom-template-email](https://github.com/humanlayer/12-factor-agents/blob/main/workshops/2025-05/walkthrough/11-email-custom.png?raw=true)

可以跟着流程走，然后你可以尝试按自己的喜好更新模板

（如果你使用 cursor，只需高亮模板并要求"让它更好"就应该可以了）

也试试触发 "request_more_information"！


就是这样 - 在下一章中，我们将构建一个完全由邮件驱动的
使用 webhooks 进行人类审批的工作流 agent 


## 第 XX 章 - HumanLayer Webhook 集成

前面的部分使用了 humanlayer SDK 的"同步模式" - 这意味着
每次我们等待人类审批时，我们都在循环中轮询直到收到人类响应。

这显然不是理想的方式，特别是对于生产工作负载，
因此在本部分中，我们将通过更新服务器在联系人类后结束处理，并使用 webhooks 接收结果来实现 [factor 6 - 使用简单 API 启动 / 暂停 / 恢复](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-6-launch-pause-resume.md)。


添加在服务器中初始化 humanlayer 的代码


```diff
src/server.ts
 import { Thread, agentLoop, handleNextStep } from '../src/agent';
 import { ThreadStore } from '../src/state';
+import { humanlayer } from 'humanlayer';
 
 const app = express();
 const store = new ThreadStore();
 
+const getHumanlayer = () => {
+    const HUMANLAYER_EMAIL = process.env.HUMANLAYER_EMAIL;
+    if (!HUMANLAYER_EMAIL) {
+        throw new Error("missing or invalid parameters: HUMANLAYER_EMAIL");
+    }
+
+    const HUMANLAYER_API_KEY = process.env.HUMANLAYER_API_KEY;
+    if (!HUMANLAYER_API_KEY) {
+        throw new Error("missing or invalid parameters: HUMANLAYER_API_KEY");
+    }
+    return humanlayer({
+        runId: `12fa-agent`,
+        contactChannel: {
+            email: { address: HUMANLAYER_EMAIL }
+        }
+    });
+}
+
 // POST /thread - Start new thread
 app.post('/thread', async (req, res) => {
     
     // loop until stop event
-    const newThread = await agentLoop(thread);
+    const result = await agentLoop(thread);
 
-    store.update(req.params.id, newThread);
+    store.update(req.params.id, result);
 
-    lastEvent = newThread.events[newThread.events.length - 1];
+    lastEvent = result.events[result.events.length - 1];
     lastEvent.data.response_url = `/thread/${req.params.id}/response`;
 
     console.log("returning last event from endpoint", lastEvent);
     
-    res.json(newThread);
+    res.json(result);
 });
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/12-1-server-init.ts src/server.ts

</details>

接下来，让我们更新 /thread 端点以
  
1. 异步处理请求，立即返回
2. 在 request_more_information 和 done_for_now 调用时创建人类联系


更新服务器以能够处理 request_clarification 响应

- 移除旧的 /response 端点和类型
- 更新 /thread 端点以异步运行处理，立即返回
- 在请求人类响应时发送 state.threadId
- 添加 handleHumanResponse 函数来处理人类响应
- 添加 /webhook 端点来处理 webhook 响应


```diff
src/server.ts
-import express from 'express';
+import express, { Request, Response } from 'express';
 import { Thread, agentLoop, handleNextStep } from '../src/agent';
 import { ThreadStore } from '../src/state';
-import { humanlayer } from 'humanlayer';
+import { humanlayer, V1Beta2HumanContactCompleted } from 'humanlayer';
 
 const app = express();
     });
 }
-
 // POST /thread - Start new thread
-app.post('/thread', async (req, res) => {
+app.post('/thread', async (req: Request, res: Response) => {
     const thread = new Thread([{
         type: "user_input",
     }]);
     
-    const threadId = store.create(thread);
-    const newThread = await agentLoop(thread);
-    
-    store.update(threadId, newThread);
+    // run agent loop asynchronously, return immediately
+    Promise.resolve().then(async () => {
+        const threadId = store.create(thread);
+        const newThread = await agentLoop(thread);
+        
+        store.update(threadId, newThread);
 
-    const lastEvent = newThread.events[newThread.events.length - 1];
-    // If we exited the loop, include the response URL so the client can
-    // push a new message onto the thread
-    lastEvent.data.response_url = `/thread/${threadId}/response`;
+        const lastEvent = newThread.events[newThread.events.length - 1];
 
-    console.log("returning last event from endpoint", lastEvent);
-
-    res.json({ 
-        thread_id: threadId,
-        ...newThread 
+        if (thread.awaitingHumanResponse()) {
+            const hl = getHumanlayer();
+            // create a human contact - returns immediately
+            hl.createHumanContact({
+                spec: {
+                    msg: lastEvent.data.message,
+                    state: {
+                        thread_id: threadId,
+                    }
+                }
+            });
+        }
     });
+
+    res.json({ status: "processing" });
 });
 
 // GET /thread/:id - Get thread status
-app.get('/thread/:id', (req, res) => {
+app.get('/thread/:id', (req: Request, res: Response) => {
     const thread = store.get(req.params.id);
     if (!thread) {
 });
 
+type WebhookResponse = V1Beta2HumanContactCompleted;
 
-type ApprovalPayload = {
-    type: "approval";
-    approved: boolean;
-    comment?: string;
-}
+const handleHumanResponse = async (req: Request, res: Response) => {
 
-type ResponsePayload = {
-    type: "response";
-    response: string;
 }
 
-type Payload = ApprovalPayload | ResponsePayload;
+app.post('/webhook', async (req: Request, res: Response) => {
+    console.log("webhook response", req.body);
+    const response = req.body as WebhookResponse;
 
-// POST /thread/:id/response - Handle clarification response
-app.post('/thread/:id/response', async (req, res) => {
-    let thread = store.get(req.params.id);
+    // response is guaranteed to be set on a webhook
+    const humanResponse: string = response.event.status?.response as string;
+
+    const threadId = response.event.spec.state?.thread_id;
+    if (!threadId) {
+        return res.status(400).json({ error: "Thread ID not found" });
+    }
+
+    const thread = store.get(threadId);
     if (!thread) {
         return res.status(404).json({ error: "Thread not found" });
     }
 
-    const body: Payload = req.body;
-
-    let lastEvent = thread.events[thread.events.length - 1];
-
-    if (thread.awaitingHumanResponse() && body.type === 'response') {
-        thread.events.push({
-            type: "human_response",
-            data: body.response
-        });
-    } else if (thread.awaitingHumanApproval() && body.type === 'approval' && !body.approved) {
-        // push feedback onto the thread
-        thread.events.push({
-            type: "tool_response",
-            data: `user denied the operation with feedback: "${body.comment}"`
-        });
-    } else if (thread.awaitingHumanApproval() && body.type === 'approval' && body.approved) {
-        // approved, run the tool, pushing results onto the thread
-        await handleNextStep(lastEvent.data, thread);
-    } else {
-        res.status(400).json({
-            error: "Invalid request: " + body.type,
-            awaitingHumanResponse: thread.awaitingHumanResponse(),
-            awaitingHumanApproval: thread.awaitingHumanApproval()
-        });
-        return;
+    if (!thread.awaitingHumanResponse()) {
+        return res.status(400).json({ error: "Thread is not awaiting human response" });
     }
 
-    
-    // loop until stop event
-    const result = await agentLoop(thread);
-
-    store.update(req.params.id, result);
-
-    lastEvent = result.events[result.events.length - 1];
-    lastEvent.data.response_url = `/thread/${req.params.id}/response`;
-
-    console.log("returning last event from endpoint", lastEvent);
-    
-    res.json(result);
 });
 
```

<details>
<summary>跳过此步骤</summary>

    cp ./walkthrough/12a-server.ts src/server.ts

</details>

在另一个终端中启动服务器

    npx tsx src/server.ts

服务器运行后，向 '/thread' 端点发送一个负载


__ 执行响应步骤

__ 现在处理 divide 的审批

__ 现在也处理 done_for_now

