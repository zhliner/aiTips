---
name: pptx
description: "当涉及 .pptx 文件的任何操作时使用此技能——无论是作为输入、输出还是两者兼有。包括：创建幻灯片、演示文稿或提案演示；读取、解析或从任何 .pptx 文件中提取文本（即使提取的内容将用于其他地方，如邮件或摘要）；编辑、修改或更新现有演示文稿；合并或拆分幻灯片文件；处理模板、布局、演讲者备注或注释。当用户提到 \"deck\"、\"slides\"、\"presentation\" 或引用 .pptx 文件名时触发此技能，无论他们计划如何处理内容。如果 .pptx 文件需要被打开、创建或处理，请使用此技能。"
license: Proprietary. LICENSE.txt has complete terms
---

# PPTX 技能

## 快速参考

| 任务 | 指南 |
|------|------|
| 读取/分析内容 | `python -m markitdown presentation.pptx` |
| 编辑或基于模板创建 | 阅读 [editing.md](editing.md) |
| 从零创建 | 阅读 [pptxgenjs.md](pptxgenjs.md) |

---

## 读取内容

```bash
# 文本提取
python -m markitdown presentation.pptx

# 视觉概览
python scripts/thumbnail.py presentation.pptx

# 原始 XML
python scripts/office/unpack.py presentation.pptx unpacked/
```

---

## 编辑工作流

**阅读 [editing.md](editing.md) 获取完整详情。**

1. 使用 `thumbnail.py` 分析模板
2. 解包 → 操作幻灯片 → 编辑内容 → 清理 → 打包

---

## 从零创建

**阅读 [pptxgenjs.md](pptxgenjs.md) 获取完整详情。**

当没有模板或参考演示文稿时使用。

---

## 设计建议

**不要创建无聊的幻灯片。** 白底纯文本要点不会打动任何人。为每张幻灯片考虑以下列表中的创意。

### 开始之前

- **选择大胆且贴合内容的配色方案**：配色应让人感觉是为当前主题量身定制的。如果将你的配色换到完全不同的演示文稿中仍然"适用"，说明你的选择还不够具体。
- **主色主导而非平均分配**：一种颜色应占主导地位（60-70% 的视觉权重），搭配 1-2 种辅助色和一个鲜明的强调色。切勿让所有颜色平均分配。
- **深浅对比**：标题页和结尾页使用深色背景，内容页使用浅色（"三明治"结构）。或者全程使用深色以营造高端感。
- **坚持一种视觉主题**：选择一个独特的元素并重复使用——圆角图片框、彩色圆圈中的图标、粗单侧边框。将其贯穿每张幻灯片。

### 配色方案

选择与主题匹配的颜色——不要默认使用通用的蓝色。以下配色方案可作为灵感：

| 主题 | 主色 | 辅助色 | 强调色 |
|------|------|--------|--------|
| **午夜行政** | `1E2761`（海军蓝） | `CADCFC`（冰蓝） | `FFFFFF`（白色） |
| **森林与苔藓** | `2C5F2D`（森林绿） | `97BC62`（苔藓绿） | `F5F5F5`（奶油色） |
| **珊瑚活力** | `F96167`（珊瑚色） | `F9E795`（金色） | `2F3C7E`（海军蓝） |
| **暖色陶土** | `B85042`（陶土色） | `E7E8D1`（沙色） | `A7BEAE`（鼠尾草绿） |
| **海洋渐变** | `065A82`（深蓝） | `1C7293`（蓝绿） | `21295C`（午夜蓝） |
| **炭灰极简** | `36454F`（炭灰色） | `F2F2F2`（灰白色） | `212121`（黑色） |
| **蓝绿信任** | `028090`（蓝绿色） | `00A896`（海沫绿） | `02C39A`（薄荷绿） |
| **浆果与奶油** | `6D2E46`（浆果色） | `A26769`（灰玫瑰色） | `ECE2D0`（奶油色） |
| **鼠尾草宁静** | `84B59F`（鼠尾草绿） | `69A297`（桉叶绿） | `50808E`（石板灰） |
| **樱桃大胆** | `990011`（樱桃红） | `FCF6F5`（灰白色） | `2F3C7E`（海军蓝） |

### 每张幻灯片

**每张幻灯片都需要一个视觉元素**——图片、图表、图标或形状。纯文本幻灯片容易被遗忘。

**布局选项：**
- 两栏布局（文字在左，插图在右）
- 图标 + 文本行（彩色圆圈中的图标、粗体标题、下方描述）
- 2x2 或 2x3 网格（一侧为图片，另一侧为内容块网格）
- 半出血图片（占据整个左侧或右侧）配合内容叠加

**数据展示：**
- 大号数据标注（60-72pt 的大数字配下方小标签）
- 对比列（前/后、优/劣、并排选项）
- 时间线或流程图（编号步骤、箭头）

**视觉润色：**
- 章节标题旁的小彩色圆圈中的图标
- 关键数据或标语使用斜体强调文本

### 字体排版

**选择有趣的字体搭配** —— 不要默认使用 Arial。选择一个有个性的标题字体，搭配一种简洁的正文字体。

| 标题字体 | 正文字体 |
|----------|----------|
| Georgia | Calibri |
| Arial Black | Arial |
| Calibri | Calibri Light |
| Cambria | Calibri |
| Trebuchet MS | Calibri |
| Impact | Arial |
| Palatino | Garamond |
| Consolas | Calibri |

| 元素 | 字号 |
|------|------|
| 幻灯片标题 | 36-44pt 粗体 |
| 章节标题 | 20-24pt 粗体 |
| 正文 | 14-16pt |
| 说明文字 | 10-12pt 弱化色 |

### 间距

- 最小边距 0.5 英寸
- 内容块之间 0.3-0.5 英寸
- 留出呼吸空间——不要填满每一寸

### 避免（常见错误）

- **不要重复相同布局** —— 在不同幻灯片间变换列、卡片和标注样式
- **不要居中正文** —— 段落和列表左对齐；仅标题居中
- **不要忽视大小对比** —— 标题需要 36pt+ 才能与 14-16pt 正文形成对比
- **不要默认使用蓝色** —— 选择能反映特定主题的颜色
- **不要随意混用间距** —— 选择 0.3" 或 0.5" 的间距并一致使用
- **不要只设计一张幻灯片而其余保持朴素** —— 要么全力投入，要么全程保持简洁
- **不要创建纯文本幻灯片** —— 添加图片、图标、图表或视觉元素；避免纯标题 + 要点
- **不要忘记文本框内边距** —— 当对齐线条或形状与文本边缘时，在文本框上设置 `margin: 0` 或偏移形状以考虑内边距
- **不要使用低对比度元素** —— 图标和文本都需要与背景形成强对比；避免浅色背景上使用浅色文本，或深色背景上使用深色文本
- **切勿在标题下方使用强调线** —— 这是 AI 生成幻灯片的典型特征；改用留白或背景色

---

## QA（必需）

**假设存在问题。你的任务是找到它们。**

你的第一次渲染几乎从不正确。将 QA 视为一次 bug 排查，而非确认步骤。如果第一次检查没发现问题，说明你检查得不够仔细。

### 内容 QA

```bash
python -m markitdown output.pptx
```

检查是否有缺失内容、拼写错误、顺序错误。

**使用模板时，检查是否有残留的占位符文本：**

```bash
python -m markitdown output.pptx | grep -iE "xxxx|lorem|ipsum|this.*(page|slide).*layout"
```

如果 grep 返回结果，在宣布完成之前先修复它们。

### 视觉 QA

**⚠️ 使用子代理** —— 即使只有 2-3 张幻灯片。你已经盯着代码看了很久，会看到你以为存在的东西而非实际存在的东西。子代理拥有全新的视角。

将幻灯片转换为图片（参见[转换为图片](#转换为图片)），然后使用以下提示词：

```
Visual inspect these slides. Assume there are issues — find them.

Look for:
- Overlapping elements (text through shapes, lines through words, stacked elements)
- Text overflow or cut off at edges/box boundaries
- Decorative lines positioned for single-line text but title wrapped to two lines
- Source citations or footers colliding with content above
- Elements too close (< 0.3" gaps) or cards/sections nearly touching
- Uneven gaps (large empty area in one place, cramped in another)
- Insufficient margin from slide edges (< 0.5")
- Columns or similar elements not aligned consistently
- Low-contrast text (e.g., light gray text on cream-colored background)
- Low-contrast icons (e.g., dark icons on dark backgrounds without a contrasting circle)
- Text boxes too narrow causing excessive wrapping
- Leftover placeholder content

For each slide, list issues or areas of concern, even if minor.

Read and analyze these images:
1. /path/to/slide-01.jpg (Expected: [brief description])
2. /path/to/slide-02.jpg (Expected: [brief description])

Report ALL issues found, including minor ones.
```

### 验证循环

1. 生成幻灯片 → 转换为图片 → 检查
2. **列出发现的问题**（如果没发现问题，更加严格地重新检查）
3. 修复问题
4. **重新验证受影响的幻灯片** —— 一个修复往往会引发另一个问题
5. 重复直到完整一轮检查没有发现新问题

**在完成至少一次修复-验证循环之前，不要宣布完成。**

---

## 转换为图片

将演示文稿转换为单独的幻灯片图片以进行视觉检查：

```bash
python scripts/office/soffice.py --headless --convert-to pdf output.pptx
pdftoppm -jpeg -r 150 output.pdf slide
```

这将创建 `slide-01.jpg`、`slide-02.jpg` 等文件。

修复后重新渲染特定幻灯片：

```bash
pdftoppm -jpeg -r 150 -f N -l N output.pdf slide-fixed
```

---

## 依赖项

- `pip install "markitdown[pptx]"` - 文本提取
- `pip install Pillow` - 缩略图网格
- `npm install -g pptxgenjs` - 从零创建
- LibreOffice（`soffice`）- PDF 转换（通过 `scripts/office/soffice.py` 为沙箱环境自动配置）
- Poppler（`pdftoppm`）- PDF 转图片
