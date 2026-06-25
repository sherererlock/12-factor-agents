# 第 10 章 - 添加人工审批

添加对操作的人工审批支持。

在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

更新服务器以处理人工审批

* 导入 `handleNextStep` 以执行已批准的操作
* 添加两种负载类型以区分审批和响应
* 在端点中分别处理响应和审批
* 在出错时显示更好的错误消息


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

为 agent 添加一些方法来处理审批和响应

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

