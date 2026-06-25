# Jupyter Notebook 测试框架

本文档描述了用于验证 Jupyter notebook 中任何功能的通用测试框架，并以测试 BAML 日志捕获为具体示例。

## 通用框架

### 概述

测试框架提供了一个完整的迭代循环，用于测试 notebook 实现：

1. **生成** 带有特定功能的测试 notebook
2. **执行** notebook 在模拟的 Google Colab 环境中运行
3. **分析** 已执行的 notebook，检查预期输出和行为
4. **报告** 清晰的通过/失败结果

### 核心组件

#### Notebook 模拟器 (`test_notebook_colab_sim.sh`)

模拟脚本为任何 notebook 创建一个逼真的 Google Colab 环境：

**环境设置：**
- 创建带时间戳的测试目录：`./tmp/test_YYYYMMDD_HHMMSS/`
- 设置全新的 Python 虚拟环境
- 安装 Jupyter 依赖（`notebook`、`nbconvert`、`ipykernel`）

**Notebook 执行：**
- 将测试 notebook 复制到干净环境
- 使用 `ExecutePreprocessor` 运行所有 cell（模拟 Colab 执行）
- **关键：** 在执行前激活虚拟环境
- **关键：** 将带有 cell 输出的已执行 notebook 保存回磁盘

**用法：**
```bash
./test_notebook_colab_sim.sh your_notebook.ipynb
```

模拟器将会：
- 执行 notebook 中的所有 cell
- 保留测试目录供检查
- 显示最终目录结构
- 报告成功/失败

#### 输出检查器 (`inspect_notebook.py`)

用于详细检查 notebook cell 输出的调试工具：

**功能：**
- 显示 cell 源代码和执行计数
- 显示所有输出类型（stream、execute_result、error）
- 高亮输出文本中的模式
- 显示带有回溯信息的执行错误
- 按关键字过滤 cell 以便集中调试

**用法：**
```bash
# 检查所有 cell
python3 inspect_notebook.py path/to/notebook.ipynb

# 过滤特定内容
python3 inspect_notebook.py path/to/notebook.ipynb "keyword"

# 查找错误
python3 inspect_notebook.py path/to/notebook.ipynb "error"
```

**示例输出：**
```
🔍 CELL 0 (code)
📝 SOURCE:
import sys
print("Hello!")
print("Error!", file=sys.stderr)

📤 OUTPUTS (2 outputs):
  Output 0: type=stream
    Text length: 7 chars
    > Hello!...
  Output 1: type=stream  
    Text length: 7 chars
    > Error!...
    🎯 Found patterns: ['Error']
```

### Notebook 测试的关键见解

#### 执行环境
1. **虚拟环境激活至关重要** - 没有它，执行会静默失败
2. **输出持久化必须是显式的** - `ExecutePreprocessor` 仅在内存中修改 notebook
3. **检查执行计数** - `execution_count=None` 意味着 cell 从未被执行
4. **处理不同的输出类型** - stream、execute_result、error、display_data

#### 常见调试步骤
1. **验证基本执行：**
   ```bash
   python3 -c "
   import json
   nb = json.load(open('path/to/notebook.ipynb'))
   print('Execution counts:', [cell.get('execution_count') for cell in nb['cells'] if cell['cell_type']=='code'])
   "
   ```

2. **检查执行错误：**
   ```bash
   python3 inspect_notebook.py path/to/notebook.ipynb "error"
   ```

3. **查找特定输出模式：**
   ```bash
   python3 inspect_notebook.py path/to/notebook.ipynb "your_pattern"
   ```

### 创建自定义测试

#### 1. 最小测试模板

创建一个测试基本功能的简单 notebook：

```json
{
  "cells": [
    {
      "cell_type": "code",
      "execution_count": null,
      "metadata": {},
      "outputs": [],
      "source": [
        "# Test basic execution\n",
        "print('Hello from notebook!')\n",
        "\n",
        "# Test file creation\n",
        "with open('test.txt', 'w') as f:\n",
        "    f.write('Test successful\\n')\n",
        "\n",
        "# Test error handling\n",
        "try:\n",
        "    result = your_function_to_test()\n",
        "    print(f'Result: {result}')\n",
        "except Exception as e:\n",
        "    print(f'Error: {e}')"
      ]
    }
  ],
  "metadata": {
    "kernelspec": {
      "display_name": "Python 3",
      "language": "python", 
      "name": "python3"
    }
  },
  "nbformat": 4,
  "nbformat_minor": 4
}
```

#### 2. 测试脚本模板

```bash
#!/bin/bash
set -e

echo "🧪 Testing [Your Feature]..."

# Clean up any previous test
rm -f test_notebook.ipynb

# Generate or copy your test notebook
cp your_test_notebook.ipynb test_notebook.ipynb

# Run in simulator
echo "🚀 Running test in sim..."
./test_notebook_colab_sim.sh test_notebook.ipynb

# Find the executed notebook
NOTEBOOK_DIR=$(ls -1dt tmp/test_* | head -1)
NOTEBOOK_PATH="$NOTEBOOK_DIR/test_notebook.ipynb"

# Analyze results
echo "📋 Analyzing results..."
python3 inspect_notebook.py "$NOTEBOOK_PATH" "your_search_term"

# Add your custom analysis
python3 -c "
import json
with open('$NOTEBOOK_PATH') as f:
    nb = json.load(f)

# Your custom analysis logic here
success = check_for_expected_outputs(nb)

if success:
    print('✅ PASS: Test succeeded!')
else:
    print('❌ FAIL: Test failed!')
    exit(1)
"

echo "🧹 Cleaning up..."
rm -f test_notebook.ipynb
```

---

## 用例：BAML 日志捕获测试

本节演示如何将通用框架用于特定用例：测试 notebook 中的 BAML 日志捕获。

### 问题陈述

BAML（一种语言模型框架）使用 FFI 绑定到 Rust 二进制文件，并将日志输出到 stderr。我们需要测试不同的日志捕获方法是否能在 Jupyter notebook cell 中成功捕获这些日志。

### 测试实现

#### 测试配置 (`simple_log_test.yaml`)

```yaml
title: "BAML Log Capture Test"
text: "Simple test for log capture"

sections:
  - title: "Log Capture Test"
    steps:
      - baml_setup: true
      - fetch_file:
          src: "walkthrough/01-agent.baml"
          dest: "baml_src/agent.baml"
      - file:
          src: "./simple_main.py"
      - text: "Testing log capture with show_logs=true:"
      - run_main:
          args: "What is 2+2?"
          show_logs: true
```

#### 测试函数 (`simple_main.py`)

```python
def main(message="What is 2+2?"):
    """Simple main function that calls BAML directly"""
    client = get_baml_client()
    
    # Call the BAML function - this should generate logs
    result = client.DetermineNextStep(f"User asked: {message}")
    
    print(f"Input: {message}")
    print(f"Result: {result}")
    return result
```

#### 日志捕获实现

`walkthroughgen_py.py` 中当前可用的实现：

```python
def run_with_baml_logs(func, *args, **kwargs):
    """Test log capture using IPython capture_output"""
    # Ensure BAML_LOG is set
    if 'BAML_LOG' not in os.environ:
        os.environ['BAML_LOG'] = 'info'
    
    print(f"[LOG CAPTURE TEST] Running with BAML_LOG={os.environ.get('BAML_LOG')}...")
    
    # Capture both stdout and stderr
    with capture_output() as captured:
        result = func(*args, **kwargs)
    
    # Display captured outputs
    if captured.stdout:
        print("=== Captured Stdout ===")
        print(captured.stdout)
    
    if captured.stderr:
        print("=== Captured BAML Logs ===")
        print(captured.stderr)
    else:
        print("=== No BAML Logs Captured ===")
    
    print("=== Function Result ===")
    print(result)
    
    return result
```

### 测试执行

#### 主测试脚本 (`test_log_capture.sh`)

```bash
#!/bin/bash
set -e

echo "🧪 Testing BAML Log Capture..."

# Generate test notebook from YAML config
echo "📝 Generating test notebook..."
uv run python walkthroughgen_py.py simple_log_test.yaml -o test_capture.ipynb

# Run in simulator  
echo "🚀 Running test in sim..."
./test_notebook_colab_sim.sh test_capture.ipynb

# Find the executed notebook
NOTEBOOK_DIR=$(ls -1dt tmp/test_* | head -1)
NOTEBOOK_PATH="$NOTEBOOK_DIR/test_notebook.ipynb"

echo "📋 Analyzing results from $NOTEBOOK_PATH..."

# Debug output
echo "🔍 Dumping debug info..."
python3 inspect_notebook.py "$NOTEBOOK_PATH" "run_with_baml_logs"

# Analyze for BAML log patterns
echo "📊 Running log capture analysis..."
python3 analyze_log_capture.py "$NOTEBOOK_PATH"

echo "🧹 Cleaning up..."
rm -f test_capture.ipynb
```

#### 分析脚本 (`analyze_log_capture.py`)

```python
#!/usr/bin/env python3
import json
import sys
import os

def check_logs(notebook_path):
    """Check if BAML logs were captured in the notebook"""
    
    with open(notebook_path) as f:
        nb = json.load(f)
    
    found_log_pattern = False
    found_capture_test = False
    
    for i, cell in enumerate(nb['cells']):
        if cell['cell_type'] == 'code' and 'outputs' in cell:
            source = ''.join(cell.get('source', []))
            if 'run_with_baml_logs' in source:
                found_capture_test = True
                print(f'Found log capture test in cell {i}')
                
                # Check outputs for BAML logs
                for output in cell['outputs']:
                    if output.get('output_type') == 'stream' and 'text' in output:
                        text = ''.join(output['text'])
                        # Look for the specific BAML log pattern
                        if '---Parsed Response (class DoneForNow)---' in text:
                            found_log_pattern = True
                            print(f'✅ FOUND BAML LOG PATTERN in cell {i} output!')
    
    return found_capture_test, found_log_pattern

# Run analysis and return pass/fail
capture_test_found, log_pattern_found = check_logs(sys.argv[1])

if not capture_test_found:
    print('❌ FAIL: No log capture test found in notebook')
    sys.exit(1)

if log_pattern_found:
    print('✅ PASS: BAML logs successfully captured in notebook output!')
    sys.exit(0)
else:
    print('❌ FAIL: BAML log pattern not found in captured output')
    sys.exit(1)
```

### 预期输出流程

#### 成功的测试运行：
```bash
$ ./test_log_capture.sh

🧪 Testing BAML Log Capture...
📝 Generating test notebook...
Generated notebook: test_capture.ipynb
🚀 Running test in sim...
🧪 Creating clean test environment in: ./tmp/test_20250716_191106
📁 Test directory will be preserved for inspection
🐍 Creating fresh Python virtual environment...
📦 Installing Jupyter dependencies...
🏃 Running notebook in clean environment...
✅ Notebook executed successfully!
💾 Executed notebook saved with outputs

📋 Analyzing results from tmp/test_20250716_191106/test_notebook.ipynb...
🔍 Dumping debug info...
Found log capture test in cell 11

📤 OUTPUTS (3 outputs):
  Output 0: type=stream
    Text length: 49 chars
    > [LOG CAPTURE TEST] Running with BAML_LOG=info......
  Output 1: type=stream
    Text length: 1272 chars
    > 2025-07-16T19:11:22.445 [BAML [92mINFO[0m] [35mFunction DetermineNextStep[0m...
    🎯 Found patterns: ['BAML', 'Parsed', 'Response']

📊 Running log capture analysis...
Found log capture test in cell 11
✅ FOUND BAML LOG PATTERN in cell 11 output!
✅ PASS: BAML logs successfully captured in notebook output!
🧹 Cleaning up...
```

### BAML 特定的关键见解

1. **BAML 日志输出到 stderr** - 由于 FFI 绑定到 Rust 二进制文件
2. **需要 `BAML_LOG=info`** - 环境变量控制详细程度
3. **日志包含 ANSI 颜色代码** - 需要处理终端格式化
4. **模式匹配** - 查找 `---Parsed Response (class DoneForNow)---` 以确认成功执行
5. **IPython capture_output() 有效** - 成功在 notebook 上下文中捕获 stderr

### 迭代循环的好处

此框架支持快速测试不同的日志捕获方法：

1. **修改** `walkthroughgen_py.py` 中的 `run_with_baml_logs` 函数
2. **运行** `./test_log_capture.sh`
3. **获得** 立即的通过/失败反馈
4. **调试** 使用 `inspect_notebook.py`（如需要）
5. **重复** 直到找到可行的实现

同样的模式可以应用于测试任何 notebook 功能：库集成、环境设置、输出格式化、错误处理等。
