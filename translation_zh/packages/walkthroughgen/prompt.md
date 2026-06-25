Walkthroughgen 是一个用于创建演练、教程、readme 和文档的工具。

## 使用方法

你通过编写一个描述演练的简单 yaml 文件来创建演练。在文件中，你引用在演练的每个步骤中应该存在的增量文件

```
├── walkthrough
│   ├── 00-package-lock.json
│   ├── 00-package.json
│   ├── 01-index.ts
│   ├── 02-cli.ts
│   └── 02-index.ts
└── walkthrough.yaml
```

你的 walkthrough.yaml 文件可能如下所示（可运行示例在 [examples/typescript-cli](./examples/typescript)）

```yaml
title: "setting up a typescript cli"
text: "this is a walkthrough for setting up a typescript cli"
targets:
  - markdown: "./build/walkthrough.md" # generates a walkthrough.md file
    onChange: # default behavior - on changes, show diffs and cp commands
      diff: true
      cp: true
    newFiles: # when new files are created, just show the copy command
      cat: false
      cp: true
  - final: "./build/final" # outputs the final project to the final folder
  - folders: "./build/by-section" # creates a separate working folder for each section
sections:
  - name: setup
    title: "Copy initial files"
    steps:
      - file: {src: ./walkthrough/00-package.json, dest: package.json}
      - file: {src: ./walkthrough/00-package-lock.json, dest: package-lock.json}
      - file: {src: ./walkthrough/00-tsconfig.json, dest: tsconfig.json}
  - name: initialize
    title: "Initialize the project"
    steps:
      - text: "initialize the project"
        command: |
          npm install
      - text: "then add index.ts"
        file: {src: ./walkthrough/01-index.ts, dest: src/index.ts}
      - text: "run it with tsx"
        command: |
          npx tsx src/index.ts
        results:
          - text: "you should see a hello world message"
            code: |
              hello world
  - name: add-cli
    title: "Add a CLI"
    steps:
      - text: "add a cli"
        file: {src: ./walkthrough/02-cli.ts, dest: src/cli.ts}
      - text: "add a cli"
        file: {src: ./walkthrough/02-index.ts, dest: src/index.ts}
```

使用以下命令构建项目：

```
npm i -g wtg
wtg build
```

根据你的目标，这将创建以下文件

```
├── walkthrough
│   ├── 00-package-lock.json
│   ├── 00-package.json
│   ├── 01-index.ts
│   ├── 02-cli.ts
│   └── 02-index.ts
├── build
│   ├── by-section
│   │   ├── 00-initialize # only contains the files in `init`
│   │   │   ├── readme.md # contains steps for this section
│   │   │   ├── package.json
│   │   │   ├── package-lock.json
│   │   │   └── tsconfig.json
│   │   └── 01-add-cli # contains the files up to the START of section 1
│   │       ├── readme.md # contains steps for this section
│   │       ├── package.json
│   │       ├── package-lock.json
│   │       ├── tsconfig.json
│   │       └── src
│   │           └── index.ts
│   ├── final
│   │   ├── package.json
│   │   ├── package-lock.json
│   │   ├── tsconfig.json
│   │   └── src
│   │       ├── cli.ts
│   │       └── index.ts
│   └── walkthrough.md

你的 walkthrough.md 文件将如下所示：

```markdown
# Setting up a typescript cli

this is a walkthrough for setting up a typescript cli

## Copy initial files

  cp walkthrough/00-package.json package.json
  cp walkthrough/00-package-lock.json package-lock.json
  cp walkthrough/00-tsconfig.json tsconfig.json

## Initialize the project

initialize the project

     npm install

then add index.ts


    cp walkthrough/01-index.ts src/index.ts

and run it with tsx

    npx tsx src/index.ts

you should see a hello world message

    hello world

## Add a CLI

add a cli

    ```
    ```
 
    cp walkthrough/02-cli.ts src/cli.ts

update index.ts to use the cli

    ```diff
      const main = async () => {
      +    return cli();
      };
        
      main();
    ```

    or just:

    cp walkthrough/02-index.ts src/index.ts

```

## 功能

### 目标

- `file`：生成单个 markdown 文件
- `folder`：为每个章节创建一组文件夹
- `final`：将最终项目输出到当前目录

### 初始化



### 章节

### 步骤

#### 步骤


## walkthroughgen 的 Walkthrough.yaml

## 实施计划

- [ ] 实现核心 walkthroughgen CLI - `wtg build` # 默认为当前目录中的 walkthrough.yaml
- 范围 1：生成 walkthrough.md
  - [ ] 为简单演练文件创建端到端测试，只是一个没有章节的单个 yaml 文件
  - [ ] 为带有单个章节的演练文件创建端到端测试
  - [ ] 测试 diff 和 cp 命令的生成
- 范围 2：生成 final/ 项目构建
  - [ ] 为带有 final 目标的演练文件创建端到端测试
- 范围 3：生成带有 readme 的按章节项目构建
  - [ ] 为带有 by-section 目标的演练文件创建端到端测试
