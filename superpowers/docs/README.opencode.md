# Superpowers for OpenCode（OpenCode 的 Superpowers）

在 [OpenCode.ai](https://opencode.ai) 中使用 Superpowers 的完整指南。

## Quick Install（快速安装）

告诉 OpenCode：

```
Clone https://github.com/obra/superpowers to ~/.config/opencode/superpowers, then create directory ~/.config/opencode/plugins, then symlink ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js to ~/.config/opencode/plugins/superpowers.js, then symlink ~/.config/opencode/superpowers/skills to ~/.config/opencode/skills/superpowers, then restart opencode.
```

## Manual Installation（手动安装）

### Prerequisites（前提条件）

- 已安装 [OpenCode.ai](https://opencode.ai)
- 已安装 Git

### macOS / Linux

```bash
# 1. 安装 Superpowers（或更新已有的）
if [ -d ~/.config/opencode/superpowers ]; then
  cd ~/.config/opencode/superpowers && git pull
else
  git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers
fi

# 2. 创建目录
mkdir -p ~/.config/opencode/plugins ~/.config/opencode/skills

# 3. 删除旧的符号链接/目录（如果存在）
rm -f ~/.config/opencode/plugins/superpowers.js
rm -rf ~/.config/opencode/skills/superpowers

# 4. 创建符号链接
ln -s ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js ~/.config/opencode/plugins/superpowers.js
ln -s ~/.config/opencode/superpowers/skills ~/.config/opencode/skills/superpowers

# 5. 重启 OpenCode
```

#### Verify Installation（验证安装）

```bash
ls -l ~/.config/opencode/plugins/superpowers.js
ls -l ~/.config/opencode/skills/superpowers
```

两者都应显示指向 superpowers 目录的符号链接。

### Windows

**前提条件：**
- 已安装 Git
- 已启用**开发者模式**或拥有**管理员权限**
  - Windows 10：设置 → 更新和安全 → 开发者选项
  - Windows 11：设置 → 系统 → 开发者选项

选择下方的 shell：[Command Prompt](#command-prompt) | [PowerShell](#powershell) | [Git Bash](#git-bash)

#### Command Prompt

以管理员身份运行，或启用开发者模式：

```cmd
:: 1. 安装 Superpowers
git clone https://github.com/obra/superpowers.git "%USERPROFILE%\.config\opencode\superpowers"

:: 2. 创建目录
mkdir "%USERPROFILE%\.config\opencode\plugins" 2>nul
mkdir "%USERPROFILE%\.config\opencode\skills" 2>nul

:: 3. 删除已有链接（重新安装时安全）
del "%USERPROFILE%\.config\opencode\plugins\superpowers.js" 2>nul
rmdir "%USERPROFILE%\.config\opencode\skills\superpowers" 2>nul

:: 4. 创建插件符号链接（需要开发者模式或管理员权限）
mklink "%USERPROFILE%\.config\opencode\plugins\superpowers.js" "%USERPROFILE%\.config\opencode\superpowers\.opencode\plugins\superpowers.js"

:: 5. 创建技能连接点（无需特殊权限）
mklink /J "%USERPROFILE%\.config\opencode\skills\superpowers" "%USERPROFILE%\.config\opencode\superpowers\skills"

:: 6. 重启 OpenCode
```

#### PowerShell

以管理员身份运行，或启用开发者模式：

```powershell
# 1. 安装 Superpowers
git clone https://github.com/obra/superpowers.git "$env:USERPROFILE\.config\opencode\superpowers"

# 2. 创建目录
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\plugins"
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\skills"

# 3. 删除已有链接（重新安装时安全）
Remove-Item "$env:USERPROFILE\.config\opencode\plugins\superpowers.js" -Force -ErrorAction SilentlyContinue
Remove-Item "$env:USERPROFILE\.config\opencode\skills\superpowers" -Force -ErrorAction SilentlyContinue

# 4. 创建插件符号链接（需要开发者模式或管理员权限）
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.config\opencode\plugins\superpowers.js" -Target "$env:USERPROFILE\.config\opencode\superpowers\.opencode\plugins\superpowers.js"

# 5. 创建技能连接点（无需特殊权限）
New-Item -ItemType Junction -Path "$env:USERPROFILE\.config\opencode\skills\superpowers" -Target "$env:USERPROFILE\.config\opencode\superpowers\skills"

# 6. 重启 OpenCode
```

#### Git Bash

注意：Git Bash 的原生 `ln` 命令会复制文件而不是创建符号链接。请使用 `cmd //c mklink` 代替（`//c` 是 Git Bash 中 `/c` 的语法）。

```bash
# 1. 安装 Superpowers
git clone https://github.com/obra/superpowers.git ~/.config/opencode/superpowers

# 2. 创建目录
mkdir -p ~/.config/opencode/plugins ~/.config/opencode/skills

# 3. 删除已有链接（重新安装时安全）
rm -f ~/.config/opencode/plugins/superpowers.js 2>/dev/null
rm -rf ~/.config/opencode/skills/superpowers 2>/dev/null

# 4. 创建插件符号链接（需要开发者模式或管理员权限）
cmd //c "mklink \"$(cygpath -w ~/.config/opencode/plugins/superpowers.js)\" \"$(cygpath -w ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js)\""

# 5. 创建技能连接点（无需特殊权限）
cmd //c "mklink /J \"$(cygpath -w ~/.config/opencode/skills/superpowers)\" \"$(cygpath -w ~/.config/opencode/superpowers/skills)\""

# 6. 重启 OpenCode
```

#### WSL Users（WSL 用户）

如果在 WSL 内运行 OpenCode，请改用 [macOS / Linux](#macos--linux) 的安装说明。

#### Verify Installation（验证安装）

**Command Prompt：**
```cmd
dir /AL "%USERPROFILE%\.config\opencode\plugins"
dir /AL "%USERPROFILE%\.config\opencode\skills"
```

**PowerShell：**
```powershell
Get-ChildItem "$env:USERPROFILE\.config\opencode\plugins" | Where-Object { $_.LinkType }
Get-ChildItem "$env:USERPROFILE\.config\opencode\skills" | Where-Object { $_.LinkType }
```

在输出中查找 `<SYMLINK>` 或 `<JUNCTION>`。

#### Troubleshooting Windows（Windows 故障排除）

**"You do not have sufficient privilege" 错误：**
- 在 Windows 设置中启用开发者模式，或
- 右键点击终端 → "以管理员身份运行"

**"Cannot create a file when that file already exists"：**
- 先运行删除命令（步骤 3），然后重试

**Symlinks not working after git clone（git clone 后符号链接不工作）：**
- 运行 `git config --global core.symlinks true` 然后重新 clone

## Usage（使用）

### Finding Skills（查找技能）

使用 OpenCode 原生的 `skill` 工具列出所有可用技能：

```
use skill tool to list skills
```

### Loading a Skill（加载技能）

使用 OpenCode 原生的 `skill` 工具加载特定技能：

```
use skill tool to load superpowers/brainstorming
```

### Personal Skills（个人技能）

在 `~/.config/opencode/skills/` 中创建你自己的技能：

```bash
mkdir -p ~/.config/opencode/skills/my-skill
```

创建 `~/.config/opencode/skills/my-skill/SKILL.md`：

```markdown
---
name: my-skill
description: Use when [condition] - [what it does]
---

# My Skill

[Your skill content here]
```

### Project Skills（项目技能）

在你的 OpenCode 项目中创建项目特定技能：

```bash
# 在你的 OpenCode 项目中
mkdir -p .opencode/skills/my-project-skill
```

创建 `.opencode/skills/my-project-skill/SKILL.md`：

```markdown
---
name: my-project-skill
description: Use when [condition] - [what it does]
---

# My Project Skill

[Your skill content here]
```

## Skill Locations（技能位置）

OpenCode 从以下位置发现技能：

1. **项目技能**（`.opencode/skills/`）- 最高优先级
2. **个人技能**（`~/.config/opencode/skills/`）
3. **Superpowers 技能**（`~/.config/opencode/skills/superpowers/`）- 通过符号链接

## Features（功能）

### Automatic Context Injection（自动上下文注入）

插件通过 `experimental.chat.system.transform` hook 自动注入 superpowers 上下文。这会在每次请求时将 "using-superpowers" 技能内容添加到系统提示中。

### Native Skills Integration（原生技能集成）

Superpowers 使用 OpenCode 原生的 `skill` 工具进行技能发现和加载。技能通过符号链接放入 `~/.config/opencode/skills/superpowers/`，使它们与你的个人技能和项目技能并列显示。

### Tool Mapping（工具映射）

为 Claude Code 编写的技能会自动适配 OpenCode。引导程序提供映射说明：

- `TodoWrite` → `update_plan`
- `Task` with subagents → OpenCode 的 `@mention` 系统
- `Skill` tool → OpenCode 原生的 `skill` 工具
- File operations → 原生 OpenCode 工具

## Architecture（架构）

### Plugin Structure（插件结构）

**位置：** `~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`

**组件：**
- `experimental.chat.system.transform` hook 用于引导注入
- 读取并注入 "using-superpowers" 技能内容

### Skills（技能）

**位置：** `~/.config/opencode/skills/superpowers/`（符号链接到 `~/.config/opencode/superpowers/skills/`）

技能由 OpenCode 的原生技能系统发现。每个技能都有一个带 YAML frontmatter 的 `SKILL.md` 文件。

## Updating（更新）

```bash
cd ~/.config/opencode/superpowers
git pull
```

重启 OpenCode 以加载更新。

## Troubleshooting（故障排除）

### Plugin not loading（插件未加载）

1. 检查插件是否存在：`ls ~/.config/opencode/superpowers/.opencode/plugins/superpowers.js`
2. 检查符号链接/连接点：`ls -l ~/.config/opencode/plugins/`（macOS/Linux）或 `dir /AL %USERPROFILE%\.config\opencode\plugins`（Windows）
3. 检查 OpenCode 日志：`opencode run "test" --print-logs --log-level DEBUG`
4. 在日志中查找插件加载消息

### Skills not found（技能未找到）

1. 验证技能符号链接：`ls -l ~/.config/opencode/skills/superpowers`（应指向 superpowers/skills/）
2. 使用 OpenCode 的 `skill` 工具列出可用技能
3. 检查技能结构：每个技能需要一个带有效 frontmatter 的 `SKILL.md` 文件

### Windows: Module not found error（Windows：模块未找到错误）

如果你在 Windows 上看到 `Cannot find module` 错误：
- **原因：** Git Bash 的 `ln -sf` 会复制文件而不是创建符号链接
- **修复：** 改用 `mklink /J` 目录连接点（参见 Windows 安装步骤）

### Bootstrap not appearing（引导未出现）

1. 验证 using-superpowers 技能是否存在：`ls ~/.config/opencode/superpowers/skills/using-superpowers/SKILL.md`
2. 检查 OpenCode 版本是否支持 `experimental.chat.system.transform` hook
3. 插件更改后重启 OpenCode

## Getting Help（获取帮助）

- 报告问题：https://github.com/obra/superpowers/issues
- 主文档：https://github.com/obra/superpowers
- OpenCode 文档：https://opencode.ai/docs/

## Testing（测试）

验证你的安装：

```bash
# 检查插件是否加载
opencode run --print-logs "hello" 2>&1 | grep -i superpowers

# 检查技能是否可被发现
opencode run "use skill tool to list all skills" 2>&1 | grep -i superpowers

# 检查引导注入
opencode run "what superpowers do you have?"
```

代理应该提到拥有 superpowers 并能够列出 `superpowers/` 中的技能。
