# 第 8 章 - 添加 API 端点

添加一个 Express 服务器以通过 HTTP 暴露 agent。

在本节中，我们将禁用 baml 日志。你可以选择性地启用它们以查看更多细节。

    export BAML_LOG=off

安装 Express 和类型定义

    npm install express && npm install --save-dev @types/express supertest

添加服务器实现

    cp ./walkthrough/08-server.ts src/server.ts

<details>
<summary>显示文件</summary>

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

使用 curl 测试（在另一个终端中）

    curl -X POST http://localhost:3000/thread \
  -H "Content-Type: application/json" \
  -d '{"message":"can you add 3 and 4"}'

你应该会收到 agent 的回复，其中包含智能体追踪信息，最后是一条类似这样的消息：


    {"intent":"done_for_now","message":"The sum of 3 and 4 is 7."}

