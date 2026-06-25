# Walkthroughgen

Walkthroughgen 是一个用于创建演练、教程、readme 和文档的工具。它通过从简单的 YAML 配置生成 markdown 和工作目录来帮助你维护分步指南。

## 功能

- 📝 **Markdown 生成**：创建包含 diff、代码块和可折叠部分的精美 markdown 文件
- 📁 **工作目录**：为演练的每个章节生成单独的目录
- 🔄 **增量更改**：跟踪和显示步骤之间的更改
- 🎯 **多目标**：输出到 markdown、章节文件夹和最终项目状态
- 📦 **文件管理**：复制文件、创建目录和运行命令
- 🔍 **丰富的 Diff**：显示文件版本之间有意义的差异
- 📚 **章节 README**：生成每个章节的文档

## 安装

```bash
npm install -g walkthroughgen
```

## 快速开始

1. 创建一个 `walkthrough.yaml` 文件：

```yaml
title: "My Tutorial"
text: "A step-by-step guide"
targets:
  - markdown: "./walkthrough.md"
    onChange:
      diff: true
      cp: true
  - folders:
      path: "./by-section"
      final:
        dirName: "final"
sections:
  - name: setup
    title: "Initial Setup"
    steps:
      - file: {src: ./files/package.json, dest: package.json}
      - command: "npm install"
```

2. 运行生成器：

```bash
walkthroughgen generate walkthrough.yaml
```

## 目录结构

一个典型的演练项目如下所示：

```
my-tutorial/
├── walkthrough/          # Source files for each step
│   ├── 00-package.json
│   ├── 01-index.ts
│   └── 02-config.ts
├── walkthrough.yaml     # Walkthrough configuration
└── build/              # Generated output
    ├── by-section/    # Section-by-section working directories
    │   ├── 00-setup/
    │   └── 01-config/
    ├── final/         # Final project state
    └── walkthrough.md # Generated markdown
```

## Walkthrough.yaml 配置

### 顶级字段

- `title`：演练的标题
- `text`：介绍文本
- `targets`：输出配置
- `sections`：教程章节

### 目标

#### Markdown 目标

```yaml
targets:
  - markdown: "./output.md"
    onChange:
      diff: true  # Show diffs for changed files
      cp: true    # Show cp commands
    newFiles:
      cat: false  # Don't show file contents
      cp: true    # Show cp commands
```

#### 文件夹目标

```yaml
targets:
  - folders:
      path: "./by-section"        # Base path for section folders
      skip: ["cleanup"]          # Sections to skip
      final:
        dirName: "final"        # Name for final state directory
```

### 章节

每个章节代表教程中的一个逻辑步骤：

```yaml
sections:
  - name: setup              # Used for folder naming and skip array
    title: "Initial Setup"   # Display title
    text: "Setup steps..."   # Section description
    steps:
      # ... steps ...
```

### 步骤

步骤定义要执行的操作：

#### 文件复制
```yaml
steps:
  - text: "Copy package.json"
    file:
      src: ./files/package.json
      dest: package.json
```

#### 目录创建
```yaml
steps:
  - text: "Create src directory"
    dir:
      create: true
      path: src
```

#### 命令执行
```yaml
steps:
  - text: "Install dependencies"
    command: "npm install"
    incremental: true  # run when building up folders target
```

#### 命令结果
```yaml
steps:
  - command: "npm run test"
    results:
      - text: "You should see:"
        code: |
          All tests passed!
```

## 生成的输出

### Markdown 功能

- **文件 Diff**：显示版本之间的更改
- **复制命令**：易于遵循的文件复制说明
- **可折叠部分**：隐藏/显示文件内容
- **代码高亮**：各种语言的语法高亮

示例 markdown 输出：

~~~markdown
# Initial Setup

Copy the package.json:

    cp ./files/package.json package.json

<details>
<summary>show file</summary>

```json
{
  "name": "my-project",
  "version": "1.0.0"
}
```
</details>

Install dependencies:

    npm install

You should see:

    added 123 packages
~~~

### 章节文件夹

`folders` 目标创建：

1. 每个章节的目录
2. 章节特定的 README.md 文件
3. 工作项目状态
4. 可选的最终状态目录

## 示例

查看 [examples](./examples) 目录获取完整示例：

- [TypeScript CLI](./examples/typescript)：基本 TypeScript 项目设置
- [Walkthroughgen](./examples/walkthroughgen)：自文档化示例

## 提示

1. 使用有意义的章节名称 - 它们将成为文件夹名称
2. 在步骤文本中包含上下文
3. 对修改状态的命令使用 `incremental: true`
4. 利用 diff 来突出重要更改
5. 使用 `skip` 数组从输出中排除设置/清理章节

## 贡献

欢迎贡献！请阅读 [CONTRIBUTING.md](./CONTRIBUTING.md) 了解详情。

## 许可证

本项目基于 MIT 许可证 - 查看 [LICENSE](./LICENSE) 文件了解详情。
