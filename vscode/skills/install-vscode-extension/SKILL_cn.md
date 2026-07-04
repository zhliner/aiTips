---
name: install-vscode-extension
description: '如何从 extension ID 安装 VS Code 扩展。当用户希望通过安装扩展为其 VS Code 环境添加新功能时使用。'
---

# 安装 VS Code 扩展

1. VS Code 扩展通过其唯一的 extension ID 来标识，通常遵循 `publisher.extensionName` 格式。例如，Microsoft 的 Python 扩展的 ID 为 `ms-python.python`。
2. 要安装 VS Code 扩展，需要使用 VS Code 命令 `workbench.extensions.installExtension` 并传入 extension ID。参数格式为：
```
[extensionId, { enable: true, installPreReleaseVersion: boolean }]
```
> 注意：如果用户明确要求安装预发布版本，或者当前环境是 VS Code Insiders，则安装预发布版本。否则，安装稳定版本。
3. 通过 `copilot_runVscodeCommand` 工具运行该命令。确保将 `skipCheck` 参数传递为 true，以避免检查命令是否存在，因为我们知道它存在。
