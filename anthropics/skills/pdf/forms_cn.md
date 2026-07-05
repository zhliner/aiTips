**关键：你必须按顺序完成这些步骤。不要跳步直接写代码。**

如需填写 PDF 表单，首先检查 PDF 是否包含可填写的表单字段。在此文件所在目录下运行此脚本：`python scripts/check_fillable_fields <file.pdf>`，然后根据结果分别进入"可填写字段"或"不可填写字段"部分，按照相应说明操作。

# 可填写字段
如果 PDF 包含可填写的表单字段：
- 在此文件所在目录下运行此脚本：`python scripts/extract_form_field_info.py <input.pdf> <field_info.json>`。它将创建一个 JSON 文件，包含以下格式的字段列表：
```
[
  {
    "field_id": (字段的唯一 ID),
    "page": (页码，从 1 开始),
    "rect": ([左, 下, 右, 上] PDF 坐标系中的边界框，y=0 为页面底部),
    "type": ("text"、"checkbox"、"radio_group" 或 "choice"),
  },
  // 复选框包含 "checked_value" 和 "unchecked_value" 属性：
  {
    "field_id": (字段的唯一 ID),
    "page": (页码，从 1 开始),
    "type": "checkbox",
    "checked_value": (将此值设为字段值以选中复选框),
    "unchecked_value": (将此值设为字段值以取消选中复选框),
  },
  // 单选按钮组包含一个 "radio_options" 列表，列出所有可选项。
  {
    "field_id": (字段的唯一 ID),
    "page": (页码，从 1 开始),
    "type": "radio_group",
    "radio_options": [
      {
        "value": (将此值设为字段值以选中此单选选项),
        "rect": (此选项对应的单选按钮的边界框)
      },
      // 其他单选选项
    ]
  },
  // 多选字段包含一个 "choice_options" 列表，列出所有可选项：
  {
    "field_id": (字段的唯一 ID),
    "page": (页码，从 1 开始),
    "type": "choice",
    "choice_options": [
      {
        "value": (将此值设为字段值以选中此选项),
        "text": (选项的显示文本)
      },
      // 其他选项
    ],
  }
]
```
- 使用此脚本将 PDF 转换为 PNG（每页一张图片）（在此文件所在目录下运行）：
`python scripts/convert_pdf_to_images.py <file.pdf> <output_directory>`
然后分析图片以确定每个表单字段的用途（确保将边界框的 PDF 坐标转换为图片坐标）。
- 创建一个 `field_values.json` 文件，格式如下，包含每个字段要填入的值：
```
[
  {
    "field_id": "last_name", // 必须与 `extract_form_field_info.py` 输出的 field_id 匹配
    "description": "用户的姓氏",
    "page": 1, // 必须与 field_info.json 中的 "page" 值匹配
    "value": "Simpson"
  },
  {
    "field_id": "Checkbox12",
    "description": "用户年满 18 岁时需勾选的复选框",
    "page": 1,
    "value": "/On" // 如果是复选框，使用其 "checked_value" 值来勾选。如果是单选按钮组，使用 "radio_options" 中的某个 "value" 值。
  },
  // 更多字段
]
```
- 在此文件所在目录下运行 `fill_fillable_fields.py` 脚本以生成填写好的 PDF：
`python scripts/fill_fillable_fields.py <input pdf> <field_values.json> <output pdf>`
此脚本会验证你提供的字段 ID 和值是否有效；如果输出错误信息，请修正相应字段后重试。

# 不可填写字段
如果 PDF 没有可填写的表单字段，你需要添加文本注释。首先尝试从 PDF 结构中提取坐标（更精确），如有需要再回退到视觉估算。

## 步骤 1：优先尝试结构提取

运行此脚本提取文本标签、线条和复选框及其精确的 PDF 坐标：
`python scripts/extract_form_structure.py <input.pdf> form_structure.json`

这将创建一个 JSON 文件，包含：
- **labels**：每个文本元素及其精确坐标（x0、top、x1、bottom，单位为 PDF 磅值）
- **lines**：定义行边界的水平线
- **checkboxes**：作为复选框的小正方形矩形（含中心坐标）
- **row_boundaries**：根据水平线计算出的行上/下位置

**检查结果**：如果 `form_structure.json` 包含有意义的标签（与表单字段对应的文本元素），请使用**方法 A：基于结构的坐标**。如果 PDF 是扫描/图片形式且标签很少或没有，请使用**方法 B：视觉估算**。

---

## 方法 A：基于结构的坐标（首选）

当 `extract_form_structure.py` 在 PDF 中找到文本标签时使用此方法。

### A.1：分析结构

读取 form_structure.json 并识别：

1. **标签组**：组成单个标签的相邻文本元素（例如 "Last" + "Name"）
2. **行结构**：`top` 值相近的标签位于同一行
3. **字段列**：输入区域从标签结束处开始（x0 = label.x1 + 间距）
4. **复选框**：直接使用结构中的复选框坐标

**坐标系**：PDF 坐标系，y=0 位于页面顶部，y 值向下递增。

### A.2：检查缺失元素

结构提取可能无法检测到所有表单元素。常见情况：
- **圆形复选框**：仅正方形矩形会被检测为复选框
- **复杂图形**：装饰性元素或非标准表单控件
- **褪色或浅色元素**：可能无法被提取

如果你在 PDF 图片中看到 form_structure.json 中未列出的表单字段，则需要对这些特定字段使用**视觉分析**（参见下方的"混合方法"）。

### A.3：使用 PDF 坐标创建 fields.json

对于每个字段，根据提取的结构计算输入坐标：

**文本字段：**
- 输入 x0 = 标签 x1 + 5（标签后的小间距）
- 输入 x1 = 下一个标签的 x0，或行边界
- 输入 top = 与标签 top 相同
- 输入 bottom = 下方的行边界线，或标签 bottom + 行高

**复选框：**
- 直接使用 form_structure.json 中的复选框矩形坐标
- entry_bounding_box = [checkbox.x0, checkbox.top, checkbox.x1, checkbox.bottom]

使用 `pdf_width` 和 `pdf_height`（表示 PDF 坐标）创建 fields.json：
```json
{
  "pages": [
    {"page_number": 1, "pdf_width": 612, "pdf_height": 792}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "姓氏输入字段",
      "field_label": "Last Name",
      "label_bounding_box": [43, 63, 87, 73],
      "entry_bounding_box": [92, 63, 260, 79],
      "entry_text": {"text": "Smith", "font_size": 10}
    },
    {
      "page_number": 1,
      "description": "美国公民 是 复选框",
      "field_label": "Yes",
      "label_bounding_box": [260, 200, 280, 210],
      "entry_bounding_box": [285, 197, 292, 205],
      "entry_text": {"text": "X"}
    }
  ]
}
```

**重要**：使用 `pdf_width`/`pdf_height` 和直接从 form_structure.json 获取的坐标。

### A.4：验证边界框

填写前，检查边界框是否有错误：
`python scripts/check_bounding_boxes.py fields.json`

这将检查边界框是否相交以及输入框是否太小而无法容纳指定字号。在填写前修复所有报告的错误。

---

## 方法 B：视觉估算（备选方案）

当 PDF 为扫描/图片形式且结构提取未找到可用的文本标签（例如，所有文本显示为 "(cid:X)" 模式）时使用此方法。

### B.1：将 PDF 转换为图片

`python scripts/convert_pdf_to_images.py <input.pdf> <images_dir/>`

### B.2：初步字段识别

检查每页图片以识别表单区域并获得字段位置的**粗略估计**：
- 表单字段标签及其大致位置
- 输入区域（用于文本输入的线条、方框或空白区域）
- 复选框及其大致位置

对于每个字段，记录大致的像素坐标（暂时不需要精确）。

### B.3：缩放细化（对精确度至关重要）

对于每个字段，裁剪估计位置周围的区域以精确细化坐标。

**使用 ImageMagick 创建缩放裁剪图：**
```bash
magick <page_image> -crop <width>x<height>+<x>+<y> +repage <crop_output.png>
```

其中：
- `<x>, <y>` = 裁剪区域的左上角（使用粗略估计值减去留白）
- `<width>, <height>` = 裁剪区域的大小（字段区域加上每侧约 50px 的留白）

**示例：** 细化一个估计位置在 (100, 150) 附近的 "Name" 字段：
```bash
magick images_dir/page_1.png -crop 300x80+50+120 +repage crops/name_field.png
```

（注意：如果 `magick` 命令不可用，请尝试使用 `convert` 加相同参数）。

**检查裁剪后的图片**以确定精确坐标：
1. 确定输入区域开始的精确像素位置（标签之后）
2. 确定输入区域结束的位置（下一个字段或边缘之前）
3. 确定输入行/方框的顶部和底部

**将裁剪坐标转换回完整图片坐标：**
- full_x = crop_x + crop_offset_x
- full_y = crop_y + crop_offset_y

示例：如果裁剪起始于 (50, 120)，而输入框在裁剪图中的起始位置为 (52, 18)：
- entry_x0 = 52 + 50 = 102
- entry_top = 18 + 120 = 138

**对每个字段重复此操作**，尽可能将相邻字段归入同一次裁剪。

### B.4：使用细化后的坐标创建 fields.json

使用 `image_width` 和 `image_height`（表示图片坐标）创建 fields.json：
```json
{
  "pages": [
    {"page_number": 1, "image_width": 1700, "image_height": 2200}
  ],
  "form_fields": [
    {
      "page_number": 1,
      "description": "姓氏输入字段",
      "field_label": "Last Name",
      "label_bounding_box": [120, 175, 242, 198],
      "entry_bounding_box": [255, 175, 720, 218],
      "entry_text": {"text": "Smith", "font_size": 10}
    }
  ]
}
```

**重要**：使用 `image_width`/`image_height` 和缩放分析中细化后的像素坐标。

### B.5：验证边界框

填写前，检查边界框是否有错误：
`python scripts/check_bounding_boxes.py fields.json`

这将检查边界框是否相交以及输入框是否太小而无法容纳指定字号。在填写前修复所有报告的错误。

---

## 混合方法：结构 + 视觉

当结构提取对大多数字段有效但遗漏了部分元素（例如圆形复选框、不常见的表单控件）时使用此方法。

1. **使用方法 A** 处理 form_structure.json 中已检测到的字段
2. **将 PDF 转换为图片**，用于对缺失字段进行视觉分析
3. **使用缩放细化**（来自方法 B）处理缺失字段
4. **合并坐标**：对于结构提取的字段，使用 `pdf_width`/`pdf_height`。对于视觉估算的字段，必须将图片坐标转换为 PDF 坐标：
   - pdf_x = image_x * (pdf_width / image_width)
   - pdf_y = image_y * (pdf_height / image_height)
5. **在 fields.json 中使用单一坐标系** —— 将所有坐标转换为 PDF 坐标并使用 `pdf_width`/`pdf_height`

---

## 步骤 2：填写前验证

**填写前务必验证边界框：**
`python scripts/check_bounding_boxes.py fields.json`

这将检查：
- 边界框是否相交（会导致文本重叠）
- 输入框是否太小而无法容纳指定字号

在继续操作前修复 fields.json 中报告的所有错误。

## 步骤 3：填写表单

填写脚本会自动检测坐标系并处理转换：
`python scripts/fill_pdf_form_with_annotations.py <input.pdf> fields.json <output.pdf>`

## 步骤 4：验证输出

将填写好的 PDF 转换为图片并验证文本位置：
`python scripts/convert_pdf_to_images.py <output.pdf> <verify_images/>`

如果文本位置不正确：
- **方法 A**：检查是否使用了 form_structure.json 中的 PDF 坐标并配合 `pdf_width`/`pdf_height`
- **方法 B**：检查图片尺寸是否匹配以及坐标是否精确到像素
- **混合方法**：确保视觉估算字段的坐标转换正确
