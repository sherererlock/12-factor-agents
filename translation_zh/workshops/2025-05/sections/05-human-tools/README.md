# 第 5 章 - 多个人类工具

在本节中，我们将添加对多个用于联系人类的工具的支持。


在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

首先，让我们添加一个可以向人类请求澄清的工具

这将与 "done_for_now" 工具不同，
可以用于在你的 agent 中更灵活地处理不同类型的人类交互。


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

注意 - 如果你使用的是 BAML 的 VSCode 扩展，
当你在编辑器中保存文件时，客户端会自动重新生成。


    npx baml-cli generate

现在，让我们更新 agent 以使用新的工具


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
通过在命令行上向用户请求输入


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

接下来，让我们添加一个测试来检查 agent 处理澄清请求的能力


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

现在我们可以再次运行测试


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

