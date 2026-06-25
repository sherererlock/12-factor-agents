# 第 0 章 - Hello World

让我们从一个基本的 TypeScript 设置和一个 hello world 程序开始。

本指南使用 TypeScript 编写（是的，Python 版本即将推出）

在工作坊的每个文件编辑步骤之间有许多检查点，
因此即使你对 TypeScript 不太熟悉，
也应该能够跟上并运行每个示例。

要运行本指南，你需要安装相对较新版本的 Node.js 和 npm

你可以使用任何你喜欢的 Node.js 版本管理器，[homebrew](https://formulae.brew.sh/formula/node) 就可以


    brew install node@20

你应该能看到 node 版本

    node --version

复制初始 package.json

    cp ./walkthrough/00-package.json package.json

<details>
<summary>查看文件</summary>

```json
// ./walkthrough/00-package.json
{
    "name": "my-agent",
    "version": "0.1.0",
    "private": true,
    "scripts": {
      "dev": "tsx src/index.ts",
      "build": "tsc"
    },
    "dependencies": {
      "tsx": "^4.15.0",
      "typescript": "^5.0.0"
    },
    "devDependencies": {
      "@types/node": "^20.0.0",
      "@typescript-eslint/eslint-plugin": "^6.0.0",
      "@typescript-eslint/parser": "^6.0.0",
      "eslint": "^8.0.0"
    }
  }
```

</details>

安装依赖

    npm install

复制 tsconfig.json

    cp ./walkthrough/00-tsconfig.json tsconfig.json

<details>
<summary>查看文件</summary>

```json
// ./walkthrough/00-tsconfig.json
{
    "compilerOptions": {
      "target": "ES2017",
      "lib": ["esnext"],
      "allowJs": true,
      "skipLibCheck": true,
      "strict": true,
      "noEmit": true,
      "esModuleInterop": true,
      "module": "esnext",
      "moduleResolution": "bundler",
      "resolveJsonModule": true,
      "isolatedModules": true,
      "jsx": "preserve",
      "incremental": true,
      "plugins": [],
      "paths": {
        "@/*": ["./*"]
      }
    },
    "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
    "exclude": ["node_modules", "walkthrough"]
  }
```

</details>

添加 .gitignore

    cp ./walkthrough/00-.gitignore .gitignore

<details>
<summary>查看文件</summary>

```gitignore
// ./walkthrough/00-.gitignore
baml_client/
node_modules/
```

</details>

创建 src 文件夹

    mkdir -p src

添加一个简单的 hello world index.ts

    cp ./walkthrough/00-index.ts src/index.ts

<details>
<summary>查看文件</summary>

```ts
// ./walkthrough/00-index.ts
async function hello(): Promise<void> {
    console.log('hello, world!')
}

async function main() {
    await hello()
}

main().catch(console.error)
```

</details>

运行以验证

    npx tsx src/index.ts

你应该看到：

    hello, world!

