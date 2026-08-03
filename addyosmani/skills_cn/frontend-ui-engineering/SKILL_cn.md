---
name: frontend-ui-engineering
description: 构建生产级 UI。在构建或修改面向用户的界面时使用。在创建组件、实现布局、管理状态，或需要输出看起来具有生产质量而非 AI 生成的效果时使用。
---

# 前端 UI 工程

## 概述

构建可访问、高性能且视觉精良的生产级用户界面。目标是让 UI 看起来像是由顶级公司中具备设计意识的工程师构建的——而不是 AI 生成的产物。这意味着要真正遵循设计系统、正确实现无障碍访问、精心设计交互模式，以及杜绝千篇一律的"AI 美学"。

## 使用场景

- 构建新的 UI 组件或页面
- 修改现有的面向用户的界面
- 实现响应式布局
- 添加交互功能或状态管理
- 修复视觉或 UX 问题

## 组件架构

### 文件结构

将与组件相关的所有文件放在一起：

```
src/components/
  TaskList/
    TaskList.tsx          # 组件实现
    TaskList.test.tsx     # 测试
    TaskList.stories.tsx  # Storybook stories（如使用）
    use-task-list.ts      # 自定义 hook（如状态较复杂）
    types.ts              # 组件专属类型（如需要）
```

### 组件模式

**优先使用组合而非配置：**

```tsx
// 好的做法：可组合
<Card>
  <CardHeader>
    <CardTitle>Tasks</CardTitle>
  </CardHeader>
  <CardBody>
    <TaskList tasks={tasks} />
  </CardBody>
</Card>

// 应避免：过度配置
<Card
  title="Tasks"
  headerVariant="large"
  bodyPadding="md"
  content={<TaskList tasks={tasks} />}
/>
```

**保持组件职责单一：**

```tsx
// 好的做法：只做一件事
export function TaskItem({ task, onToggle, onDelete }: TaskItemProps) {
  return (
    <li className="flex items-center gap-3 p-3">
      <Checkbox checked={task.done} onChange={() => onToggle(task.id)} />
      <span className={task.done ? 'line-through text-muted' : ''}>{task.title}</span>
      <Button variant="ghost" size="sm" onClick={() => onDelete(task.id)}>
        <TrashIcon />
      </Button>
    </li>
  );
}
```

**将数据获取与展示分离：**

```tsx
// 容器组件：处理数据
export function TaskListContainer() {
  const { tasks, isLoading, error } = useTasks();

  if (isLoading) return <TaskListSkeleton />;
  if (error) return <ErrorState message="Failed to load tasks" retry={refetch} />;
  if (tasks.length === 0) return <EmptyState message="No tasks yet" />;

  return <TaskList tasks={tasks} />;
}

// 展示组件：处理渲染
export function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <ul role="list" className="divide-y">
      {tasks.map(task => <TaskItem key={task.id} task={task} />)}
    </ul>
  );
}
```

## 状态管理

**选择能解决问题的最简方案：**

```
Local state (useState)           → 组件专属的 UI 状态
Lifted state                     → 2-3 个兄弟组件间共享的状态
Context                          → 主题、认证、语言环境（读多写少）
URL state (searchParams)         → 筛选、分页、可分享的 UI 状态
Server state (React Query, SWR)  → 带缓存的远程数据
Global store (Zustand, Redux)    → 全应用共享的复杂客户端状态
```

**避免 prop drilling 超过 3 层。** 如果你需要将 props 穿过不使用它们的组件传递，请引入 context 或重构组件树。

## 遵循设计系统

### 避免 AI 美学

AI 生成的 UI 有一些可辨识的模式。全部避免它们：

| AI 默认风格 | 问题所在 | 生产级做法 |
|---|---|---|
| 到处使用紫色/靛蓝色 | 模型默认选择视觉上"安全"的配色，导致每个应用看起来都一样 | 使用项目实际的调色板 |
| 过度使用渐变 | 渐变增加视觉噪音，与大多数设计系统冲突 | 使用与设计系统匹配的纯色或微妙渐变 |
| 到处大圆角（rounded-2xl） | 最大圆角传达"友好"感，但忽略了真实设计中圆角半径的层级关系 | 使用设计系统中一致的 border-radius |
| 通用 hero 区块 | 模板驱动的布局，与实际内容或用户需求无关 | 以内容为先的布局 |
| Lorem ipsum 式文案 | 占位文本掩盖了真实内容才会暴露的布局问题（长度、换行、溢出） | 使用逼真的占位内容 |
| 到处过大的内边距 | 均匀的宽大内边距破坏了视觉层级，浪费屏幕空间 | 使用一致的间距比例 |
| 千篇一律的卡片网格 | 统一网格是一种布局捷径，忽略了信息优先级和浏览模式 | 以目的为导向的布局 |
| 重度阴影设计 | 层叠阴影增加的深度感与内容竞争，并拖慢低端设备的渲染速度 | 除非设计系统要求，否则使用微妙阴影或不用阴影 |

### 间距与布局

使用一致的间距比例。不要随意发明数值：

```css
/* 使用比例：0.25rem 递增（或项目使用的任何比例） */
/* 好的做法 */  padding: 1rem;      /* 16px */
/* 好的做法 */  gap: 0.75rem;       /* 12px */
/* 不好的做法 */ padding: 13px;      /* 不在任何比例上 */
/* 不好的做法 */ margin-top: 2.3rem; /* 不在任何比例上 */
```

### 排版

遵循标题层级：

```
h1 → 页面标题（每页仅一个）
h2 → 章节标题
h3 → 子章节标题
body → 默认文本
small → 辅助/帮助文本
```

不要跳过标题层级。不要将标题样式用于非标题内容。

### 颜色

- 使用语义化颜色 token：`text-primary`、`bg-surface`、`border-default`——而非原始十六进制值
- 确保足够的对比度（普通文本 4.5:1，大号文本 3:1）
- 不要仅依赖颜色传达信息（同时使用图标、文本或图案）

## 无障碍访问（WCAG 2.1 AA）

每个组件都必须满足以下标准：

### 键盘导航

```tsx
// 每个可交互元素都必须可通过键盘访问
<button onClick={handleClick}>Click me</button>        // ✓ 默认可获焦
<div onClick={handleClick}>Click me</div>               // ✗ 不可获焦
<div role="button" tabIndex={0} onClick={handleClick}    // ✓ 但优先使用 <button>
     onKeyDown={e => {
       if (e.key === 'Enter') handleClick();
       if (e.key === ' ') e.preventDefault();
     }}
     onKeyUp={e => {
       if (e.key === ' ') handleClick();
     }}>
  Click me
</div>
```

### ARIA 标签

```tsx
// 为缺少可见文本的可交互元素添加标签
<button aria-label="Close dialog"><XIcon /></button>

// 为表单输入添加标签
<label htmlFor="email">Email</label>
<input id="email" type="email" />

// 或在没有可见标签时使用 aria-label
<input aria-label="Search tasks" type="search" />
```

### 焦点管理

```tsx
// 内容变化时移动焦点
function Dialog({ isOpen, onClose }: DialogProps) {
  const closeRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    if (isOpen) closeRef.current?.focus();
  }, [isOpen]);

  // 对话框打开时将焦点限制在其内部
  return (
    <dialog open={isOpen}>
      <button ref={closeRef} onClick={onClose}>Close</button>
      {/* 对话框内容 */}
    </dialog>
  );
}
```

### 有意义的空状态和错误状态

```tsx
// 不要显示空白屏幕
function TaskList({ tasks }: { tasks: Task[] }) {
  if (tasks.length === 0) {
    return (
      <div role="status" className="text-center py-12">
        <TasksEmptyIcon className="mx-auto h-12 w-12 text-muted" />
        <h3 className="mt-2 text-sm font-medium">No tasks</h3>
        <p className="mt-1 text-sm text-muted">Get started by creating a new task.</p>
        <Button className="mt-4" onClick={onCreateTask}>Create Task</Button>
      </div>
    );
  }

  return <ul role="list">...</ul>;
}
```

## 响应式设计

先为移动端设计，再逐步扩展：

```tsx
// Tailwind：移动优先的响应式
<div className="
  grid grid-cols-1      /* 移动端：单列 */
  sm:grid-cols-2        /* 小屏：2 列 */
  lg:grid-cols-3        /* 大屏：3 列 */
  gap-4
">
```

在以下断点进行测试：320px、768px、1024px、1440px。

## 加载与过渡

```tsx
// 骨架屏加载（内容区域不要用 spinner）
function TaskListSkeleton() {
  return (
    <div className="space-y-3" aria-busy="true" aria-label="Loading tasks">
      {Array.from({ length: 3 }).map((_, i) => (
        <div key={i} className="h-12 bg-muted animate-pulse rounded" />
      ))}
    </div>
  );
}

// 乐观更新以提升感知速度
function useToggleTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: toggleTask,
    onMutate: async (taskId) => {
      await queryClient.cancelQueries({ queryKey: ['tasks'] });
      const previous = queryClient.getQueryData(['tasks']);

      queryClient.setQueryData(['tasks'], (old: Task[]) =>
        old.map(t => t.id === taskId ? { ...t, done: !t.done } : t)
      );

      return { previous };
    },
    onError: (_err, _taskId, context) => {
      queryClient.setQueryData(['tasks'], context?.previous);
    },
  });
}
```

## 另请参阅

有关详细的无障碍要求和测试工具，请参阅 `references/accessibility-checklist.md`。

## 常见的自我合理化

| 合理化说法 | 现实 |
|---|---|
| "无障碍是锦上添花" | 在许多司法管辖区这是法律要求，也是工程质量标准。 |
| "以后再适配响应式" | 事后改造响应式设计的难度是从一开始就构建的 3 倍。 |
| "设计还没定稿，先跳过样式" | 使用设计系统默认值。未样式化的 UI 会给审查者留下糟糕的第一印象。 |
| "这只是个原型" | 原型会变成生产代码。从一开始就把基础打好。 |
| "AI 美学暂时可以接受" | 它传达出低质量感。从一开始就使用项目实际的设计系统。 |

## 危险信号

- 组件超过 200 行（拆分它们）
- 行内样式或随意的像素值
- 缺少错误状态、加载状态或空状态
- 未进行键盘导航测试
- 仅用颜色作为状态指示（红/绿而没有文本或图标）
- 千篇一律的"AI 外观"（紫色渐变、超大卡片、模板布局）

## 验证

构建 UI 后：

- [ ] 组件渲染无控制台错误
- [ ] 所有可交互元素可通过键盘访问（用 Tab 键遍历页面）
- [ ] 屏幕阅读器能传达页面的内容和结构
- [ ] 响应式：在 320px、768px、1024px、1440px 下正常工作
- [ ] 加载状态、错误状态和空状态均已处理
- [ ] 遵循项目的设计系统（间距、颜色、排版）
- [ ] 开发工具或 axe-core 中无无障碍警告
