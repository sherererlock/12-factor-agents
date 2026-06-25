# 第 XX 章 - HumanLayer Webhook 集成

前面的章节使用了 humanlayer SDK 的"同步模式" - 这意味着每次我们等待人工审批时，都会在一个循环中轮询直到收到人工响应。

这显然不是理想的做法，特别是对于生产工作负载，
因此在本节中，我们将实现[因素 6 - 使用简单 API 启动 / 暂停 / 恢复](https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-6-launch-pause-resume.md)，
通过更新服务器在联系人类后结束处理，并使用 webhook 接收结果。


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
2. 在 request_more_information 和 done_for_now 调用时创建人类联系人


更新服务器以能够处理 request_clarification 响应

- 移除旧的 /response 端点和类型
- 更新 /thread 端点以异步运行处理，立即返回
- 在请求人工响应时发送 state.threadId
- 添加 handleHumanResponse 函数来处理人工响应
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

现在服务器正在运行，向 '/thread' 端点发送一个负载


__ 执行响应步骤

__ 现在处理 divide 的审批

__ 现在也处理 done_for_now

