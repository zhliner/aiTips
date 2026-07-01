# Cross-Platform Polyglot Hooks for Claude Code（Claude Code 跨平台多语言 Hooks）

Claude Code 插件需要在 Windows、macOS 和 Linux 上都能工作的 hooks。本文档介绍了实现这一点的多语言包装技术。

## The Problem（问题）

Claude Code 通过系统默认 shell 运行 hook 命令：
- **Windows**：CMD.exe
- **macOS/Linux**：bash 或 sh

这带来了几个挑战：

1. **脚本执行**：Windows CMD 无法直接执行 `.sh` 文件——它会尝试在文本编辑器中打开
2. **路径格式**：Windows 使用反斜杠（`C:\path`），Unix 使用正斜杠（`/path`）
3. **环境变量**：`$VAR` 语法在 CMD 中不起作用
4. **PATH 中没有 `bash`**：即使安装了 Git Bash，CMD 运行时 `bash` 也不在 PATH 中

## The Solution: Polyglot `.cmd` Wrapper（解决方案：多语言 `.cmd` 包装器）

多语言脚本在多种语言中同时是有效语法。我们的包装器在 CMD 和 bash 中都是有效的：

```cmd
: << 'CMDBLOCK'
@echo off
"C:\Program Files\Git\bin\bash.exe" -l -c "\"$(cygpath -u \"$CLAUDE_PLUGIN_ROOT\")/hooks/session-start.sh\""
exit /b
CMDBLOCK

# Unix shell runs from here
"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
```

### How It Works（工作原理）

#### On Windows (CMD.exe)（在 Windows 上）

1. `: << 'CMDBLOCK'` - CMD 将 `:` 视为标签（如 `:label`），忽略 `<< 'CMDBLOCK'`
2. `@echo off` - 禁止命令回显
3. bash.exe 命令运行时带以下参数：
   - `-l`（登录 shell）以获取包含 Unix 工具的正确 PATH
   - `cygpath -u` 将 Windows 路径转换为 Unix 格式（`C:\foo` → `/c/foo`）
4. `exit /b` - 退出批处理脚本，CMD 在此停止
5. `CMDBLOCK` 之后的所有内容 CMD 永远不会到达

#### On Unix (bash/sh)（在 Unix 上）

1. `: << 'CMDBLOCK'` - `:` 是空操作，`<< 'CMDBLOCK'` 开始一个 heredoc
2. 直到 `CMDBLOCK` 的所有内容都被 heredoc 消费（忽略）
3. `# Unix shell runs from here` - 注释
4. 脚本直接使用 Unix 路径运行

## File Structure（文件结构）

```
hooks/
├── hooks.json           # 指向 .cmd 包装器
├── session-start.cmd    # 多语言包装器（跨平台入口点）
└── session-start.sh     # 实际 hook 逻辑（bash 脚本）
```

### hooks.json

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/session-start.cmd\""
          }
        ]
      }
    ]
  }
}
```

注意：路径必须加引号，因为 `${CLAUDE_PLUGIN_ROOT}` 在 Windows 上可能包含空格（例如 `C:\Program Files\...`）。

## Requirements（要求）

### Windows
- 必须安装 **Git for Windows**（提供 `bash.exe` 和 `cygpath`）
- 默认安装路径：`C:\Program Files\Git\bin\bash.exe`
- 如果 Git 安装在其他位置，需要修改包装器

### Unix (macOS/Linux)
- 标准 bash 或 sh shell
- `.cmd` 文件必须有执行权限（`chmod +x`）

## Writing Cross-Platform Hook Scripts（编写跨平台 Hook 脚本）

实际的 hook 逻辑放在 `.sh` 文件中。为确保它在 Windows 上通过 Git Bash 工作：

### Do（推荐做法）：
- 尽可能使用纯 bash 内置命令
- 使用 `$(command)` 而不是反引号
- 所有变量扩展都加引号：`"$VAR"`
- 使用 `printf` 或 here-docs 进行输出

### Avoid（避免做法）：
- 可能不在 PATH 中的外部命令（sed、awk、grep）
- 如果必须使用它们，它们在 Git Bash 中可用，但确保 PATH 设置正确（使用 `bash -l`）

### Example: JSON Escaping Without sed/awk（示例：不使用 sed/awk 的 JSON 转义）

不要这样：
```bash
escaped=$(echo "$content" | sed 's/\\/\\\\/g' | sed 's/"/\\"/g' | awk '{printf "%s\\n", $0}')
```

使用纯 bash：
```bash
escape_for_json() {
    local input="$1"
    local output=""
    local i char
    for (( i=0; i<${#input}; i++ )); do
        char="${input:$i:1}"
        case "$char" in
            $'\\') output+='\\' ;;
            '"') output+='\"' ;;
            $'\n') output+='\n' ;;
            $'\r') output+='\r' ;;
            $'\t') output+='\t' ;;
            *) output+="$char" ;;
        esac
    done
    printf '%s' "$output"
}
```

## Reusable Wrapper Pattern（可复用包装器模式）

对于有多个 hooks 的插件，你可以创建一个通用包装器，将脚本名作为参数传入：

### run-hook.cmd
```cmd
: << 'CMDBLOCK'
@echo off
set "SCRIPT_DIR=%~dp0"
set "SCRIPT_NAME=%~1"
"C:\Program Files\Git\bin\bash.exe" -l -c "cd \"$(cygpath -u \"%SCRIPT_DIR%\")\" && \"./%SCRIPT_NAME%\""
exit /b
CMDBLOCK

# Unix shell runs from here
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]:-$0}")" && pwd)"
SCRIPT_NAME="$1"
shift
"${SCRIPT_DIR}/${SCRIPT_NAME}" "$@"
```

### hooks.json using the reusable wrapper（使用可复用包装器的 hooks.json）
```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" validate-bash.sh"
          }
        ]
      }
    ]
  }
}
```

## Troubleshooting（故障排除）

### "bash is not recognized"
CMD 找不到 bash。包装器使用完整路径 `C:\Program Files\Git\bin\bash.exe`。如果 Git 安装在其他位置，请更新路径。

### "cygpath: command not found" 或 "dirname: command not found"
Bash 没有以登录 shell 运行。确保使用了 `-l` 标志。

### Path has weird `\/` in it（路径中出现奇怪的 `\/`）
`${CLAUDE_PLUGIN_ROOT}` 展开为以反斜杠结尾的 Windows 路径，然后附加了 `/hooks/...`。使用 `cygpath` 转换整个路径。

### Script opens in text editor instead of running（脚本在文本编辑器中打开而不是运行）
hooks.json 直接指向了 `.sh` 文件。改为指向 `.cmd` 包装器。

### Works in terminal but not as hook（在终端中可以但作为 hook 不行）
Claude Code 可能以不同方式运行 hooks。通过模拟 hook 环境来测试：
```powershell
$env:CLAUDE_PLUGIN_ROOT = "C:\path\to\plugin"
cmd /c "C:\path\to\plugin\hooks\session-start.cmd"
```

## Related Issues（相关问题）

- [anthropics/claude-code#9758](https://github.com/anthropics/claude-code/issues/9758) - .sh 脚本在 Windows 上在编辑器中打开
- [anthropics/claude-code#3417](https://github.com/anthropics/claude-code/issues/3417) - Hooks 在 Windows 上不工作
- [anthropics/claude-code#6023](https://github.com/anthropics/claude-code/issues/6023) - CLAUDE_PROJECT_DIR 未找到
