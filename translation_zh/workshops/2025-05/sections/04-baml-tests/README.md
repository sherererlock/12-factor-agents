# 第 4 章 - 为 agent.baml 添加测试

让我们为 BAML agent 添加一些测试。

首先，保持 baml 日志处于启用状态

    export BAML_LOG=debug

接下来，让我们为 agent 添加一些测试

我们从一个简单的测试开始，检查 agent 处理基本计算的能力。


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

现在，让我们通过断言来改进测试！

断言是确保 agent 按预期工作的好方法，并且可以轻松扩展以检查更复杂的行为。


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

随着你添加更多测试，可以禁用日志以保持输出整洁。
在对特定测试进行迭代时，你可能会想重新启用它们。


    export BAML_LOG=off

现在，让我们添加一些更复杂的测试用例，
从正在进行的智能体上下文窗口中间恢复执行


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

让我们尝试运行它


    npx baml-cli test

