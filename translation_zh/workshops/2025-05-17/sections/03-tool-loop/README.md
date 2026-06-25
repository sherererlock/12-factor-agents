# 第 3 章 - 在循环中处理工具调用

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

恭喜，你已经迈出了手动构建 agent 循环的第一步。

从这里开始，我们将开始融入一些更中级和高级的
12-factor agent 概念。

