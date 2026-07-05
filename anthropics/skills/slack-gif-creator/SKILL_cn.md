---
name: slack-gif-creator
description: Knowledge and utilities for creating animated GIFs optimized for Slack. Provides constraints, validation tools, and animation concepts. Use when users request animated GIFs for Slack like "make me a GIF of X doing Y for Slack."
license: Complete terms in LICENSE.txt
---

# Slack GIF Creator（Slack GIF 创建器）

一套用于创建针对 Slack 优化的动画 GIF 的工具集，提供实用工具和知识。

## Slack 要求

**尺寸：**
- Emoji GIF：128x128（推荐）
- 消息 GIF：480x480

**参数：**
- FPS：10-30（越低文件越小）
- 颜色数：48-128（越少文件越小）
- 时长：Emoji GIF 建议控制在 3 秒以内

## 核心工作流

```python
from core.gif_builder import GIFBuilder
from PIL import Image, ImageDraw

# 1. 创建构建器
builder = GIFBuilder(width=128, height=128, fps=10)

# 2. 生成帧
for i in range(12):
    frame = Image.new('RGB', (128, 128), (240, 248, 255))
    draw = ImageDraw.Draw(frame)

    # 使用 PIL 基本图元绘制动画
    # （圆形、多边形、线条等）

    builder.add_frame(frame)

# 3. 优化后保存
builder.save('output.gif', num_colors=48, optimize_for_emoji=True)
```

## 绘制图形

### 处理用户上传的图片
如果用户上传了图片，需要判断他们的意图：
- **直接使用**（如"给这张图做动画"、"把这张图拆分成帧"）
- **作为参考灵感**（如"做一个类似这种风格的"）

使用 PIL 加载和处理图片：
```python
from PIL import Image

uploaded = Image.open('file.png')
# 直接使用，或仅作为颜色/风格的参考
```

### 从零开始绘制
从零绘制图形时，使用 PIL 的 ImageDraw 基本图元：

```python
from PIL import ImageDraw

draw = ImageDraw.Draw(frame)

# 圆形/椭圆
draw.ellipse([x1, y1, x2, y2], fill=(r, g, b), outline=(r, g, b), width=3)

# 星形、三角形及任意多边形
points = [(x1, y1), (x2, y2), (x3, y3), ...]
draw.polygon(points, fill=(r, g, b), outline=(r, g, b), width=3)

# 线条
draw.line([(x1, y1), (x2, y2)], fill=(r, g, b), width=5)

# 矩形
draw.rectangle([x1, y1, x2, y2], fill=(r, g, b), outline=(r, g, b), width=3)
```

**不要使用：** Emoji 字体（跨平台表现不可靠），也不要假定本 skill 中预置了现成的图形资源。

### 让图形更美观

图形应当看起来精致且有创意，而不是粗糙简陋。以下是具体建议：

**使用更粗的线条** - 轮廓线和线条始终设置 `width=2` 或更大。细线（width=1）看起来锯齿感强且不够专业。

**增加视觉层次感**：
- 使用渐变背景（`create_gradient_background`）
- 叠加多个形状以增加复杂度（例如一颗大星形内部嵌套一颗小星形）

**让形状更有趣**：
- 不要只画一个普通的圆——添加高光、光环或纹理
- 星形可以加上光晕效果（在后方绘制更大、半透明的版本）
- 组合多种形状（星星 + 闪光、圆 + 光环）

**注意配色**：
- 使用鲜明、互补的色彩
- 增加对比度（浅色形状配深色轮廓，深色形状配浅色轮廓）
- 考虑整体构图

**对于复杂形状**（心形、雪花等）：
- 使用多边形和椭圆的组合
- 仔细计算点位以保证对称性
- 添加细节（心形可以加一条高光弧线，雪花可以有精细的分支）

发挥创意，注重细节！一个好的 Slack GIF 应当看起来精致，而不是像占位图。

## 可用工具

### GIFBuilder（`core.gif_builder`）
组装帧并针对 Slack 进行优化：
```python
builder = GIFBuilder(width=128, height=128, fps=10)
builder.add_frame(frame)  # 添加 PIL Image
builder.add_frames(frames)  # 添加帧列表
builder.save('out.gif', num_colors=48, optimize_for_emoji=True, remove_duplicates=True)
```

### Validators（`core.validators`）
检查 GIF 是否满足 Slack 要求：
```python
from core.validators import validate_gif, is_slack_ready

# 详细验证
passes, info = validate_gif('my.gif', is_emoji=True, verbose=True)

# 快速检查
if is_slack_ready('my.gif'):
    print("Ready!")
```

### 缓动函数（`core.easing`）
实现平滑运动，替代线性变化：
```python
from core.easing import interpolate

# 进度值从 0.0 到 1.0
t = i / (num_frames - 1)

# 应用缓动
y = interpolate(start=0, end=400, t=t, easing='ease_out')

# 可用缓动类型：linear、ease_in、ease_out、ease_in_out、
#              bounce_out、elastic_out、back_out
```

### 帧辅助函数（`core.frame_composer`）
常用场景的便捷函数：
```python
from core.frame_composer import (
    create_blank_frame,         # 纯色背景
    create_gradient_background,  # 垂直渐变背景
    draw_circle,                # 圆形绘制辅助
    draw_text,                  # 简单文本渲染
    draw_star                   # 五角星
)
```

## 动画概念

### 抖动/振动
通过振荡偏移对象位置：
- 使用 `math.sin()` 或 `math.cos()` 结合帧索引
- 添加小幅随机变化以获得自然感
- 应用于 x 和/或 y 坐标

### 脉冲/心跳
有节奏地缩放对象大小：
- 使用 `math.sin(t * frequency * 2 * math.pi)` 实现平滑脉冲
- 心跳效果：两次快速脉冲后暂停（调整正弦波形）
- 在基础尺寸的 0.8 到 1.2 倍之间缩放

### 弹跳
对象下落并弹起：
- 使用 `interpolate()` 配合 `easing='bounce_out'` 实现落地效果
- 使用 `easing='ease_in'` 实现下落（加速）效果
- 通过每帧增加 y 方向速度来模拟重力

### 旋转
绕中心旋转对象：
- PIL：`image.rotate(angle, resample=Image.BICUBIC)`
- 摇摆效果：用正弦波控制角度而非线性变化

### 淡入/淡出
逐渐出现或消失：
- 创建 RGBA 图像，调整 alpha 通道
- 或使用 `Image.blend(image1, image2, alpha)`
- 淡入：alpha 从 0 到 1
- 淡出：alpha 从 1 到 0

### 滑动
将对象从屏幕外移动到目标位置：
- 起始位置：帧边界之外
- 终止位置：目标位置
- 使用 `interpolate()` 配合 `easing='ease_out'` 实现平滑停止
- 超调效果：使用 `easing='back_out'`

### 缩放
通过缩放和位移实现变焦效果：
- 放大：从 0.1 缩放到 2.0，裁剪中心区域
- 缩小：从 2.0 缩放到 1.0
- 可添加运动模糊增强戏剧效果（PIL 滤镜）

### 爆炸/粒子喷射
创建向外辐射的粒子：
- 生成具有随机角度和速度的粒子
- 更新每个粒子：`x += vx`，`y += vy`
- 施加重力：`vy += gravity_constant`
- 粒子随时间淡出（降低 alpha 值）

## 优化策略

仅在需要减小文件大小时，实施以下部分方法：

1. **减少帧数** - 降低 FPS（10 而非 20）或缩短时长
2. **减少颜色数** - `num_colors=48` 而非 128
3. **缩小尺寸** - 128x128 而非 480x480
4. **去除重复帧** - 在 save() 中设置 `remove_duplicates=True`
5. **Emoji 模式** - `optimize_for_emoji=True` 自动优化

```python
# Emoji 最大优化
builder.save(
    'emoji.gif',
    num_colors=48,
    optimize_for_emoji=True,
    remove_duplicates=True
)
```

## 设计理念

本 skill 提供：
- **知识**：Slack 的要求和动画概念
- **工具**：GIFBuilder、验证器、缓动函数
- **灵活性**：使用 PIL 基本图元自行创建动画逻辑

本 skill 不提供：
- 固定的动画模板或预制函数
- Emoji 字体渲染（跨平台表现不可靠）
- 内置于 skill 中的预制图形资源库

**关于用户上传**：本 skill 不包含预置图形，但如果用户上传了图片，可以使用 PIL 加载和处理——根据用户的请求判断他们是想直接使用还是仅作为灵感参考。

发挥创意！组合多种概念（弹跳 + 旋转、脉冲 + 滑动等），充分利用 PIL 的全部能力。

## 依赖

```bash
pip install pillow imageio numpy
```
