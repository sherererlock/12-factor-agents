# 第 11 章 - 通过邮件进行人工审批

在本节中，我们将添加通过邮件进行人工审批的支持。

这部分内容一开始会有点刻意，只是为了让大家理解基本概念 -

我们将从 CLI 调用工作流开始，但 `divide` 和 `request_more_information` 的审批将通过邮件处理，
然后最终的 `done_for_now` 答案将打印回 CLI

虽然有些刻意，但这是你从[因素 7 - 使用工具联系人类](../../../../content/factor-7-contact-humans-with-tools.md)中获得灵活性的一个很好的例子


在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

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

程序的最后一行应该会提到人工审核步骤

    nextStep { intent: 'divide', a: 4, b: 5 }
HumanLayer: Requested human approval from HumanLayer cloud

继续回复邮件并提供一些反馈：

![reject-email](../../../../workshops/2025-05/walkthrough/11-email-reject.png)


你应该会收到另一封邮件，其中包含根据你的反馈更新后的尝试！

你可以批准这次操作：

![approve-email](../../../../workshops/2025-05/walkthrough/11-email-approve.png)


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

你应该会收到一封请求澄清的邮件

    Can you clarify what 'xyz' represents in this context? Is it a specific number, variable, or something else?

你可以回复类似这样的内容

    use 8 instead of xyz

你应该会在 CLI 上看到类似这样的最终结果

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

先用 divide 试试：


    npx tsx src/index.ts 'can you divide 4 by 5'

你应该会看到使用自定义模板的略有不同的邮件

![custom-template-email](../../../../workshops/2025-05/walkthrough/11-email-custom.png)

按照流程继续操作，然后你可以尝试按自己的喜好更新模板

（如果你使用 cursor，只需高亮模板并要求"让它变得更好"就可以了）

也试着触发 "request_more_information"！


就这些了 - 在下一章中，我们将构建一个完全由邮件驱动的工作流 agent，使用 webhook 进行人工审批

