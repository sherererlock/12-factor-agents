# 工作坊 2025-07-16：Python/Jupyter Notebook 实现

* **主要工具**：`walkthroughgen_py.py` - 将 TypeScript 演练转换为 Jupyter notebook
* **配置**：`walkthrough.yaml` - 定义 notebook 结构和内容
* **输出**：`workshop_final.ipynb` - 生成的包含第 0-7 章的 notebook
* **测试**：`test_notebook_colab_sim.sh` - 模拟 Google Colab 环境

## 关键实现经验

* **Notebook 中不使用 async/await** - 所有 BAML 调用必须是同步的，移除所有异步模式
* **不使用 sys.argv** - 主函数直接接受参数：`main("hello")` 而非命令行参数
* **全局命名空间** - 在 cell 中定义的函数全局持久化，cell 之间没有模块导入
* **BAML 设置是可选的** - 仅在引入 BAML 时使用 `baml_setup: true` 步骤（第 1 章及以后）
* **get_baml_client() 模式** - 解决 Google Colab 导入缓存问题的必要方法
* **从 GitHub 获取 BAML 文件** - 使用 curl 获取，因为 Colab 无法显示本地 BAML 文件
* **重新生成 BAML** - 当 BAML 文件更改时在 run_main 中使用 `regenerate_baml: true`
* **移除导入** - 从 Python 文件中移除 `from baml_client import get_baml_client` 导入
* **IN_COLAB 检测** - 使用 try/except 对 google.colab 导入进行环境检测
* **人工输入处理** - get_human_input() 在 Colab 中使用真实的 input()，在本地使用自动响应

## 实现模式

* **walkthroughgen_py.py 增强** - 为 run_main 步骤添加了 kwargs 支持
* **测试模拟** - test_notebook_colab_sim.sh 创建包含所有依赖的干净虚拟环境
* **调试产物** - 测试运行结果保存在 ./tmp/test_TIMESTAMP/ 目录中
* **BAML 测试支持** - baml-cli test 在 notebook 中运行良好，与最初假设相反
* **工具执行** - agent 循环中的所有计算器操作（加/减/乘/除）
* **澄清流程** - ClarificationRequest 工具用于处理模糊输入
* **序列化格式** - JSON 与 XML 用于线程历史（XML 更节省 token）
* **渐进复杂度** - 从 hello world 开始，逐步添加 BAML、工具、循环、测试

## 各章节实现状态

* **第 0 章**：Hello World - 简单 Python 程序，无 BAML ✅
* **第 1 章**：CLI 和 Agent - BAML 介绍，基本 agent ✅
* **第 2 章**：计算器工具 - 工具定义（无执行） ✅
* **第 3 章**：工具循环 - 包含工具执行的完整 agent 循环 ✅
* **第 4 章**：BAML 测试 - 带断言的测试用例 ✅
* **第 5 章**：人类工具 - 带输入处理的澄清请求 ✅
* **第 6 章**：改进提示词 - 提示词中的推理步骤 ✅
* **第 7 章**：上下文序列化 - JSON/XML 线程格式 ✅
* **第 8-12 章**：跳过 - 基于服务器的功能不适合 notebook ⚠️

## 避免的常见陷阱

* **导入错误** - baml_client 导入在 notebook 中失败，使用全局 get_baml_client
* **异步模式** - Notebook 无法处理 async/await，一切必须是同步的
* **文件路径** - 使用 notebook 目录的绝对路径，处理 ./ 前缀
* **BAML 文件冲突** - 每章更新相同的文件（agent.baml）而非章节特定文件
* **工具注册** - 确保 agent 循环 switch 语句中处理了所有工具类型
* **测试预期** - BAML 测试可能有不同的输出，断言验证关键属性
* **环境差异** - 代码必须在 Colab 和本地测试环境中都能工作

## 测试命令

* 生成 notebook：`uv run python walkthroughgen_py.py walkthrough.yaml -o test.ipynb`
* 完整 Colab 模拟：`./test_notebook_colab_sim.sh`
* 运行 BAML 测试：`baml-cli test`（从包含 baml_src 的目录）

## 文件结构

* `walkthrough/*.py` - 每章代码的 Python 实现
* `walkthrough/*.baml` - notebook 执行期间从 GitHub 获取的 BAML 文件
* `walkthroughgen_py.py` - 主转换工具
* `walkthrough.yaml` - 包含所有章节的 notebook 定义
* `test_notebook_colab_sim.sh` - 完整 Colab 环境模拟
* `workshop_final.ipynb` - 为工作坊准备的最终生成 notebook
