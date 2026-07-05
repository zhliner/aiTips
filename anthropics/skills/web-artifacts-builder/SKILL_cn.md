---
name: web-artifacts-builder
description: Suite of tools for creating elaborate, multi-component claude.ai HTML artifacts using modern frontend web technologies (React, Tailwind CSS, shadcn/ui). Use for complex artifacts requiring state management, routing, or shadcn/ui components - not for simple single-file HTML/JSX artifacts.
license: Complete terms in LICENSE.txt
---

# Web Artifacts Builder（Web Artifact 构建器）

要构建功能强大的前端 claude.ai artifact，请按以下步骤操作：
1. 使用 `scripts/init-artifact.sh` 初始化前端项目
2. 通过编辑生成的代码来开发 artifact
3. 使用 `scripts/bundle-artifact.sh` 将所有代码打包为单个 HTML 文件
4. 向用户展示 artifact
5. （可选）测试 artifact

**技术栈**：React 18 + TypeScript + Vite + Parcel（打包）+ Tailwind CSS + shadcn/ui

## 设计与样式指南

非常重要：为了避免所谓的"AI 风格泛滥"（AI slop），请避免过度使用居中布局、紫色渐变、统一的圆角和 Inter 字体。

## 快速开始

### 步骤 1：初始化项目

运行初始化脚本创建新的 React 项目：
```bash
bash scripts/init-artifact.sh <project-name>
cd <project-name>
```

这将创建一个完整配置的项目，包含：
- ✅ React + TypeScript（通过 Vite）
- ✅ Tailwind CSS 3.4.1 及 shadcn/ui 主题系统
- ✅ 路径别名（`@/`）已配置
- ✅ 预装 40+ 个 shadcn/ui 组件
- ✅ 包含所有 Radix UI 依赖
- ✅ Parcel 已配置用于打包（通过 .parcelrc）
- ✅ 兼容 Node 18+（自动检测并锁定 Vite 版本）

### 步骤 2：开发 Artifact

编辑生成的文件来构建 artifact。参见下方**常见开发任务**获取指引。

### 步骤 3：打包为单个 HTML 文件

将 React 应用打包为单个 HTML artifact：
```bash
bash scripts/bundle-artifact.sh
```

这将生成 `bundle.html`——一个自包含的 artifact，所有 JavaScript、CSS 和依赖均已内联。此文件可以直接作为 artifact 在 Claude 对话中分享。

**要求**：项目根目录下必须存在 `index.html`。

**脚本执行内容**：
- 安装打包依赖（parcel、@parcel/config-default、parcel-resolver-tspaths、html-inline）
- 创建支持路径别名的 `.parcelrc` 配置
- 使用 Parcel 构建（不生成 source map）
- 使用 html-inline 将所有资源内联到单个 HTML 文件中

### 步骤 4：向用户分享 Artifact

最后，在对话中将打包好的 HTML 文件分享给用户，以便他们以 artifact 形式查看。

### 步骤 5：测试/预览 Artifact（可选）

注意：此步骤完全可选。仅在必要时或用户要求时执行。

要测试/预览 artifact，可使用可用工具（包括其他 Skill 或内置工具如 Playwright 或 Puppeteer）。通常应避免提前测试 artifact，因为这会增加从请求到完成 artifact 可见之间的延迟。如果需要测试或出现问题，请在展示 artifact 之后再进行。

## 参考

- **shadcn/ui 组件**：https://ui.shadcn.com/docs/components
