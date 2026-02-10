# AITips - 个人 AI 小提示/片段集

存放一些个人使用的 AI 编码用提示词（*.md），如 commands、系统级 AGENTS.md 等。
以及其它一些零碎物件。


## 工作流（Human & AI）

0. 构想（Conception）。项目发起，人类主导。
1. 提案（Proposal）。明确边界，人类参与。
2. 实现（Implementation Plan）。由AI根据提案创建。
    - 设计，编码级。
    - 任务清单，实现步骤。
3. 实施（Apply）、Testing。AI执行，人类审视/旁观。


## 使用

### opencode (https://opencode.ai/)

#### AGENTS - 系统级提示词

克隆项目，创建符号链接（或复制）。

```bash
$> cd ~/.config/opencode
$> git clone https://github.com/zhliner/aiTips.git
$> ln -s aiTips/AGENTS.md AGENTS.md
```

> **注：**
> 本系统提示词仅为中文本地化说明，且中英双语。


#### commands

创建符号连接指向自定义命令集。

```bash
cd ~/.config/opencode/commands/
ln -s ~/.config/opencode/aiTips/commands myai  # 符号链接指向整个目录
```

启动 opencode: `/myai/...` 即可找到自定义命令。


## 提示

- 本集合中定义的文档都使用中文（zh-cn）。
- 表意的文字可能天生适合与AI交互吧？毕竟都是语义空间……

