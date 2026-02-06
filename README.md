# AITips - 个人 AI 小提示/片段集

存放一些个人使用的 AI 编码用提示词（*.md），如 commands、系统级 AGENTS.md 等。
以及其它一些零碎物件。


## 工作流（Human & AI）

0. 项目构想。人类行为。
1. 提案（Proposal）。明确边界，人类参与。
2. 实现（Implementation Plan）。编码级设计。
3. 任务清单。Agent 执行流程。
4. 方案实施（Apply）…… Test…… Done!


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

- 本集合中定义的文档都使用中文（zh-cn）。
- 表意的文字可能天生适合与AI交互吧？毕竟都是语义空间……

