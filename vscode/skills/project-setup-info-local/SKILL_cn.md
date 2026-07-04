---
name: project-setup-info-local
description: '全面的项目搭建步骤，帮助用户在 VS Code 工作区中创建完整的项目结构；此工具专为完整的项目初始化和脚手架搭建而设计，而非用于创建单个文件。何时使用此工具：用户想从零开始创建一个完整的新项目；搭建整个项目框架（TypeScript 项目、React 应用、Node.js 服务器等）；初始化具有完整结构的 Model Context Protocol（MCP）服务器；创建具有适当脚手架的 VS Code 扩展；搭建 Next.js、Vite 或其他基于框架的项目；用户要求"新建项目"、"创建工作区"、"搭建一个 [框架] 项目"；需要建立包含依赖项、配置文件和文件夹结构的完整开发环境。何时不应使用此工具：创建单个文件或小段代码示例；向现有项目添加单个文件；对现有代码库进行修改；用户要求"创建一个文件"或"添加一个组件"；简单的代码示例或演示；调试或修复现有代码。此工具提供完整的项目搭建，包括：文件夹结构创建；package.json 和依赖管理；配置文件（tsconfig、eslint 等）；初始模板代码；开发环境搭建；构建和运行说明。对于现有项目中的单个文件，请使用其他文件创建工具。'
---

# 如何搭建项目

确定用户想要创建什么类型的项目，然后据此选择下方对应的搭建说明来执行。仅在文件夹为空时搭建项目，或者在刚刚调用工具创建工作区之后进行。

## vscode-extension

使用 Yeoman 和 Generator-Code 创建 VS Code 扩展的模板。

运行此命令：

```
npx --package yo --package generator-code -- yo code . --skipOpen
```
该命令具有以下参数：

- `-t, --extensionType`：指定扩展类型：ts、js、command-ts、command-js、colortheme、language、snippets、keymap、extensionpack、localization、commandweb、notebook。默认为 `ts`
- `-n, --extensionDisplayName`：设置扩展的显示名称。
- `--extensionId`：设置扩展的唯一 ID。如果用户未要求唯一 ID，则不要选择此选项。
- `--extensionDescription`：提供扩展的描述信息。
- `--pkgManager`：指定包管理器：npm、yarn 或 pnpm。默认为 `npm`。
- `--bundler`：使用 webpack 或 esbuild 打包扩展。
- `--gitInit`：为扩展初始化 Git 仓库。
- `--snippetFolder`：指定代码片段文件夹的位置。
- `--snippetLanguage`：设置 snippets 的语言。

### 规则

1. 不要从命令中移除任何参数。仅在用户要求时添加参数。
2. 调用 `get_vscode_api` 工具，传入用户的查询以获取相关参考。
3. 在 `get_vscode_api` 工具完成后，再开始修改项目。

## next-js

基于 React 的框架，用于构建服务端渲染的 Web 应用。

运行此命令：

```
npx create-next-app@latest .
```
该命令具有以下参数：

- `--ts, --typescript`：初始化为 TypeScript 项目。这是默认选项。
- `--js, --javascript`：初始化为 JavaScript 项目。
- `--tailwind`：使用 Tailwind CSS 配置初始化。这是默认选项。
- `--eslint`：使用 ESLint 配置初始化。
- `--app`：初始化为 App Router 项目。
- `--src-dir`：在 'src/' 目录内初始化。
- `--turbopack`：默认启用 Turbopack 进行开发。
- `--import-alias <prefix/*>`：指定要使用的导入别名。（默认为 "@/*"）
- `--api`：使用 App Router 初始化无头 API。
- `--empty`：初始化空项目。
- `--use-npm`：明确告知 CLI 使用 npm 引导应用。
- `--use-pnpm`：明确告知 CLI 使用 pnpm 引导应用。
- `--use-yarn`：明确告知 CLI 使用 Yarn 引导应用。
- `--use-bun`：明确告知 CLI 使用 Bun 引导应用。

## vite

专注于速度和性能的前端构建工具。可与 React、Vue、Preact、Lit、Svelte、Solid 和 Qwik 配合使用。

运行此命令：

```
npx create-vite@latest .
```
该命令具有以下参数：

- `-t, --template NAME`：使用特定模板。可用模板：vanilla-ts、vanilla、vue-ts、vue、react-ts、react、react-swc-ts、react-swc、preact-ts、preact、lit-ts、lit、svelte-ts、svelte、solid-ts、solid、qwik-ts、qwik

## mcp-server

Model Context Protocol（MCP）服务器项目。此项目支持多种编程语言，包括 TypeScript、JavaScript、Python、C#、Java 和 Kotlin。

### 规则

1. 首先，访问 https://github.com/modelcontextprotocol 查找所请求语言的正确 SDK 和搭建说明。如果未指定语言，默认使用 TypeScript。
2. 使用 `fetch_webpage` 工具从 https://modelcontextprotocol.io/llms-full.txt 查找正确的实现说明。
3. 更新 .github 目录中的 copilot-instructions.md 文件，添加对 SDK 文档的引用。
4. 在项目根目录的 `.vscode` 文件夹中创建 `mcp.json` 文件，内容如下：`{ "servers": { "mcp-server-name": { "type": "stdio", "command": "command-to-run", "args": [list-of-args] } } }`。
   - mcp-server-name：MCP 服务器的名称。创建一个能体现此 MCP 服务器功能的唯一名称。
   - command-to-run：用于启动 MCP 服务器的命令。即用于运行刚创建的项目的命令。
   - list-of-args：传递给命令的参数。即用于运行刚创建的项目的参数列表。
5. 根据所选语言安装任何所需的 VS Code 扩展（例如，Python 项目需安装 Python 扩展）。
6. 告知用户现在可以使用 VS Code 调试此 MCP 服务器。

## python-script

简单的 Python 脚本项目，当只需创建单个脚本时应选择此项。

必需扩展：`ms-python.python`、`ms-python.vscode-python-envs`

### 规则

1. 调用 `copilot_runVscodeCommand` 工具，在 VS Code 中正确创建新的 Python 脚本项目。使用以下参数调用该命令。
   注意 "python-script" 和 "true" 是常量，而 "New Project Name" 和 "/path/to/new/project" 分别是项目名称和路径的占位符。
   ```json
   {
     "name": "python-envs.createNewProjectFromTemplate",
     "commandId": "python-envs.createNewProjectFromTemplate",
     "args": ["python-script", "true", "New Project Name", "/path/to/new/project"]
   }
   ```

## python-package

Python 包项目，可用于创建可分发的包。

必需扩展：`ms-python.python`、`ms-python.vscode-python-envs`

### 规则

1. 调用 `run_vscode_command` 工具，在 VS Code 中正确创建新的 Python 包项目。使用以下参数调用该命令。
   注意 "python-package" 和 "true" 是常量，而 "New Package Name" 和 "/path/to/new/project" 分别是包名和路径的占位符。
   ```json
   {
     "name": "python-envs.createNewProjectFromTemplate",
     "commandId": "python-envs.createNewProjectFromTemplate",
     "args": ["python-package", "true", "New Package Name", "/path/to/new/project"]
   }
   ```
