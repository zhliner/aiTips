# AITools - 个人 AI 工具集

主要存放一些个人使用的 AI 编码工具（*.md），比如 commands、系统级 AGENTS.md 等。


## 使用

### opencode (https://opencode.ai/)

#### AGENTS - 系统级提示词

复制 `AGENTS.md` 文件到 `~/.config/opencode/` 下即可。


#### commands

在 `~/.config/opencode/commands/` 中创建符号连接，指向自定义命令集。

```bash
cd ~/.config/opencode/
git clone https://github.com/zhliner/aitools.git
cd commands/
ln -s ~/.config/opencode/aitools/commands myai  # 符号链接指向整个目录
```

然后在启动之后的 opencode 命令行中：`/myai/...` 找到自定义命令。


## 提示

- 本集合中定义的任何工具皆使用中文（zh-cn）。
- 中文为表意文字，实际上天生适合与AI交互（语义空间）。

