# 第 6 章 - 使用推理自定义你的提示词

在本节中，我们将探索如何使用推理步骤来自定义 agent 的提示词。

这是[因素 2 - 掌控你的提示词](../../../../content/factor-2-own-your-prompts.md)的核心内容

AI That Works 上有一篇关于推理的深入文章：[推理模型与推理步骤](https://github.com/hellovai/ai-that-works/tree/main/2025-04-07-reasoning-models-vs-prompts)


在本节中，保持 baml 日志处于启用状态会很有帮助

    export BAML_LOG=debug

更新 agent 提示词以包含推理步骤


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

生成更新后的客户端

    npx baml-cli generate

现在，你可以用一个简单的提示词来试一试


    npx tsx src/index.ts 'can you multiply 3 and 4'

你应该能看到 baml 日志输出中显示的推理步骤

#### 可选挑战

在你的工具输出格式中添加一个字段，将推理步骤包含在输出中！

