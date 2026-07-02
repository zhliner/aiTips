---
name: frontend-ui-engineering
description: 构建生产级 UI。在构建或修改面向用户的界面时使用。在创建组件、实现布局、管理状态时使用，或当输出需要达到生产质量而非 AI 生成效果时使用。
---

# 前端 UI 工程

## 概述

构建具有可访问性、高性能、视觉精良的生产级用户界面。目标是让 UI 看起来像出自顶级公司中具有设计意识的工程师之手——而非由 AI 生成。这意味着真正的设计系统遵循、恰当的可访问性、深思熟虑的交互模式，杜绝通用的"AI 美学"。

## 适用场景

- 构建新的 UI 组件或页面
- 修改现有的面向用户界面
- 实现响应式布局
- 添加交互或状态管理
- 修复视觉或 UX 问题

## 组件架构

### 文件结构

将与组件相关的所有内容放在一起：

```
src/components/
  TaskList/
    TaskList.tsx          # 组件实现
    TaskList.test.tsx     # 测试
    TaskList.stories.tsx  # Storybook 故事（如使用）
    use-task-list.ts      # 自定义 hook（如状态复杂）
    types.ts              # 组件特定类型（如需要）
```

### 组件模式

**优先使用组合而非配置：**

```tsx
// 好：可组合
<Card>
  <CardHeader>
    <CardTitle>任务</CardTitle>
  </CardHeader>
  <CardBody>
    <TaskList tasks={tasks} />
  </CardBody>
</Card>

// 避免：过度配置
<Card
  title="任务"
  headerVariant="large"
  bodyPadding="md"
  content={<TaskList tasks={tasks} />}
/>
```

**保持组件职责聚焦：**

```tsx
// 好：只做一件事
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
  if (error) return <ErrorState message="加载任务失败" retry={refetch} />;
  if (tasks.length === 0) return <EmptyState message="暂无任务" />;

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

**选择能完成任务的最简单方式：**

```
本地状态 (useState)                → 组件特定的 UI 状态
状态提升                            → 2-3 个兄弟组件之间共享
Context                            → 主题、认证、语言（读多写少）
URL 状态 (searchParams)            → 筛选、分页、可分享的 UI 状态
服务端状态 (React Query, SWR)       → 带缓存的远程数据
全局存储 (Zustand, Redux)           → 应用全局共享的复杂客户端状态
```

**避免超过 3 层的 prop drilling。** 如果你在向不直接使用 props 的组件传递 props，请引入 context 或重构组件树。

## 设计系统遵循

### 避免 AI 美学

AI 生成的 UI 有可识别的模式。全部避免：

| AI 默认做法 | 问题所在 | 生产级质量 |
|---|---|---|
| 无处不在的紫色/靛蓝色 | 模型默认使用视觉上"安全"的调色板，让每个应用看起来一模一样 | 使用项目实际的色彩调色板 |
| 过度的渐变 | 渐变增加视觉噪音，与大多数设计系统冲突 | 扁平或与设计系统匹配的微妙渐变 |
| 到处使用大圆角 (rounded-2xl) | 最大化圆角传递"友好"感，但忽略了真实设计中圆角半径的层级关系 | 统一使用设计系统的 border-radius |
| 通用的 hero 区域 | 模板驱动的布局，与实际内容或用户需求无关 | 内容优先的布局 |
| Lorem ipsum 风格的文案 | 占位文本掩盖了真实内容才会暴露的布局问题（长度、换行、溢出） | 贴近实际的占位内容 |
| 处处使用超大内边距 | 均等的慷慨内边距破坏了视觉层级，浪费屏幕空间 | 一致的空间尺寸 |
| 千篇一律的卡片网格 | 统一网格是一种布局捷径，忽略了信息优先级和浏览模式 | 目的驱动的布局 |
| 重阴影设计 | 多层阴影增加了与内容竞争的深度感，并在低端设备上拖慢渲染速度 | 微妙或不用阴影，除非设计系统明确要求 |

### 间距与布局

使用一致的间距尺寸体系。不要随意创造值：

```css
/* 使用尺度体系：0.25rem 递增（或项目使用的任何体系） */
/* 好 */  padding: 1rem;      /* 16px */
/* 好 */  gap: 0.75rem;       /* 12px */
/* 不好 */ padding: 13px;      /* 不在任何尺度上 */
/* 不好 */ margin-top: 2.3rem; /* 不在任何尺度上 */
```

### 排版

遵循排版层级：

```
h1 → 页面标题（每页一个）
h2 → 段落标题
h3 → 子段落标题
body → 默认正文
small → 辅助/说明文本
```

不要跳过标题层级。不要将标题样式用于非标题内容。

### 颜色

- 使用语义化颜色 token：`text-primary`、`bg-surface`、`border-default`——而非原始 hex 值
- 确保足够的对比度（普通文本 4.5:1，大文本 3:1）
- 不要仅依赖颜色传达信息（同时使用图标、文本或图案）

## 可访问性（WCAG 2.1 AA）

每个组件必须满足以下标准：

### 键盘导航

```tsx
// 每个可交互元素必须可通过键盘访问
<button onClick={handleClick}>点我</button>                      // ✓ 默认可获得焦点
<div onClick={handleClick}>点我</div>                             // ✗ 无法获得焦点
<div role="button" tabIndex={0} onClick={handleClick}             // ✓ 但更推荐 <button>
     onKeyDown={e => {
       if (e.key === 'Enter') handleClick();
       if (e.key === ' ') e.preventDefault();
     }}
     onKeyUp={e => {
       if (e.key === ' ') handleClick();
     }}>
  点我
</div>
```

### ARIA 标签

```tsx
// 为缺少可见文本的可交互元素添加标签
<button aria-label="关闭对话框"><XIcon /></button>

// 为表单输入添加标签
<label htmlFor="email">邮箱</label>
<input id="email" type="email" />

// 或在没有可见标签时使用 aria-label
<input aria-label="搜索任务" type="search" />
```

### 焦点管理

```tsx
// 内容变化时移动焦点
function Dialog({ isOpen, onClose }: DialogProps) {
  const closeRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    if (isOpen) closeRef.current?.focus();
  }, [isOpen]);

  // 对话框打开时将焦点锁定在其内部
  return (
    <dialog open={isOpen}>
      <button ref={closeRef} onClick={onClose}>关闭</button>
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
        <h3 className="mt-2 text-sm font-medium">暂无任务</h3>
        <p className="mt-1 text-sm text-muted">创建新任务以开始使用。</p>
        <Button className="mt-4" onClick={onCreateTask}>创建任务</Button>
      </div>
    );
  }

  return <ul role="list">...</ul>;
}
```

## 响应式设计

优先针对移动端设计，然后扩展：

```tsx
// Tailwind：移动优先的响应式
<div className="
  grid grid-cols-1      /* 移动端：单列 */
  sm:grid-cols-2        /* 小屏：2 列 */
  lg:grid-cols-3        /* 大屏：3 列 */
  gap-4
">
```

在以下断点测试：320px、768px、1024px、1440px。

## 加载与过渡

```tsx
// 骨架屏加载（对内容区域不要使用 loading 动画）
function TaskListSkeleton() {
  return (
    <div className="space-y-3" aria-busy="true" aria-label="正在加载任务">
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

## 参见

有关详细的可访问性要求和测试工具，请参阅 `references/accessibility-checklist.md`。

## 常见合理化借口

| 借口 | 现实 |
|---|---|
| "可访问性是锦上添花" | 在许多司法管辖区这是法律要求，也是工程质量标准。 |
| "我们后面再做响应式" | 后期改造响应式设计比从一开始就构建难 3 倍。 |
| "设计还没定稿，所以先不管样式" | 使用设计系统默认值。无样式 UI 会给评审者留下糟糕的第一印象。 |
| "这只是个原型" | 原型会变成生产代码。从一开始就把基础打好。 |
| "AI 美学暂时没问题" | 它传达出低质量的信号。从一开始就使用项目实际的设计系统。 |

## 危险信号

- 组件超过 200 行（拆分它们）
- 内联样式或任意的像素值
- 缺少错误状态、加载状态或空状态
- 没有进行键盘导航测试
- 仅用颜色作为状态的唯一指示（没有文本或图标就使用红/绿色）
- 通用的"AI 风格"（紫色渐变、超大卡片、千篇一律的布局）

## 验证

构建 UI 之后：

- [ ] 组件渲染时无控制台错误
- [ ] 所有可交互元素均可通过键盘访问（通过 Tab 键浏览页面）
- [ ] 屏幕阅读器能够传达页面的内容和结构
- [ ] 响应式：在 320px、768px、1024px、1440px 下均正常工作
- [ ] 加载、错误和空状态全部得到处理
- [ ] 遵循项目设计系统（间距、颜色、排版）
- [ ] 开发工具或 axe-core 中没有可访问性警告
