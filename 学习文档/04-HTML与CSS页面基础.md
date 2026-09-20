# 04｜HTML 与 CSS 页面基础

> 目标：能读懂 Vue Template 和组件样式，能让 AI 写出结构清楚、响应式且可访问的页面。

[← JSON](./03-JSON数据格式.md) · [学习首页](../README.md) · [TypeScript →](./05-TypeScript基础与类型系统.md)

## 1. HTML 与 CSS 的分工

```text
HTML   页面有什么、结构和语义
CSS    页面怎样布局和呈现
Vue    状态、交互和组件化
```

Vue 不会替代 HTML 和 CSS。`.vue` 文件的 template 最终生成 HTML，style 最终生成 CSS。

## 2. 基础页面结构

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>TodoLab</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

Vite 从 `index.html` 加载 `src/main.ts`，Vue 再挂载到 `#app`。

## 3. 语义化标签

```html
<header>站点标题和主导航</header>
<nav>导航链接</nav>
<main>当前页面主要内容</main>
<section>一个有标题的内容区域</section>
<article>可独立理解的内容</article>
<aside>补充内容</aside>
<footer>页脚信息</footer>
```

语义化能改善：

- 屏幕阅读器理解；
- 键盘导航；
- 搜索引擎理解；
- 代码维护。

不要把所有元素都写成 div，也不要只为了样式选择标签。

## 4. 标题、文本与链接

```html
<main>
  <h1>任务列表</h1>
  <section aria-labelledby="active-title">
    <h2 id="active-title">未完成任务</h2>
    <p>今天还有 3 项任务。</p>
    <a href="/todos/42">查看任务详情</a>
  </section>
</main>
```

页面通常有一个清晰 h1，后续标题按层级组织。不要因为字号需要而跳级，字号交给 CSS。

链接用于导航，按钮用于执行动作。

## 5. 表单

```html
<form>
  <label for="todo-title">任务标题</label>
  <input
    id="todo-title"
    name="title"
    type="text"
    maxlength="100"
    required
  />
  <button type="submit">创建任务</button>
</form>
```

重点：

- input 有关联 label；
- button 明确 type；
- name 对传统表单提交很重要；
- placeholder 不能代替 label；
- 前端 required 只改善体验，后端仍要校验。

## 6. 图片与可访问文本

```html
<img src="/todo-board.png" alt="任务看板截图" />
```

装饰图片使用空 alt：

```html
<img src="/divider.svg" alt="" />
```

有含义的图标按钮必须有可访问名称：

```html
<button type="button" aria-label="删除任务">
  <svg aria-hidden="true">...</svg>
</button>
```

## 7. CSS 选择器和层叠

```css
.todo-card {
  color: #0f172a;
}

.todo-card.completed {
  color: #64748b;
}
```

样式结果由来源、重要性、选择器优先级和出现顺序共同决定。

业务项目优先使用低复杂度 class，不要依赖很深的 DOM 层级选择器或大量 `!important`。

## 8. 盒模型

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

```text
content → padding → border → margin
```

`box-sizing: border-box` 让声明的 width 包含 padding 和 border，更容易控制布局。

## 9. Flexbox

适合一维排列：

```css
.toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 0.75rem;
  flex-wrap: wrap;
}
```

```text
justify-content   主轴对齐
align-items       交叉轴对齐
gap               项目间距
flex-wrap         是否换行
```

## 10. Grid

适合二维区域：

```css
.todo-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
  gap: 1rem;
}
```

卡片列表、仪表板和复杂页面区域通常适合 Grid；导航和按钮组通常适合 Flexbox。

## 11. 响应式布局

移动优先：

```css
.page {
  width: min(100% - 2rem, 72rem);
  margin-inline: auto;
}

@media (min-width: 48rem) {
  .layout {
    grid-template-columns: 16rem 1fr;
  }
}
```

避免固定宽度造成横向滚动。图片通常使用：

```css
img {
  max-width: 100%;
  height: auto;
}
```

## 12. CSS 变量

```css
:root {
  --color-primary: #2563eb;
  --color-text: #0f172a;
  --color-muted: #64748b;
  --radius-md: 0.75rem;
  --space-md: 1rem;
}

.button-primary {
  color: white;
  background: var(--color-primary);
  border-radius: var(--radius-md);
}
```

变量适合设计令牌和主题，不要为每一个偶然值创建变量。

## 13. 显示与隐藏

| 方法 | 可见 | 占据布局 | 可能接收交互 |
| --- | :---: | :---: | :---: |
| `opacity: 0` | 否 | 是 | 是 |
| `visibility: hidden` | 否 | 是 | 否 |
| `display: none` | 否 | 否 | 否 |

`opacity` 会让元素及子内容一起透明，不等于元素不存在。

只想让背景半透明时使用带 alpha 的颜色：

```css
background: rgb(15 23 42 / 0.75);
```

## 14. 定位与层叠

```css
.dialog-backdrop {
  position: fixed;
  inset: 0;
  display: grid;
  place-items: center;
  background: rgb(15 23 42 / 0.5);
  z-index: 1000;
}
```

`z-index` 受堆叠上下文影响。`opacity < 1`、transform 等属性可能创建新堆叠上下文，不能只靠无限增大 z-index 排错。

## 15. 键盘和焦点

```css
button:focus-visible,
a:focus-visible,
input:focus-visible {
  outline: 3px solid #60a5fa;
  outline-offset: 2px;
}
```

不要无替代地移除 outline。交互控件要能使用 Tab 到达，并有明显焦点状态。

优先使用原生 button、a、input，而不是给 div 模拟全部交互。

## 16. Vue 单文件组件中的样式

```vue
<template>
  <article class="todo-card">
    <h2>{{ todo.title }}</h2>
  </article>
</template>

<style scoped>
.todo-card {
  padding: 1rem;
  border: 1px solid #cbd5e1;
  border-radius: 0.75rem;
}
</style>
```

`scoped` 通过编译属性限制样式范围，不等于原生 Shadow DOM。全局基础样式和设计令牌仍应放在统一文件。

## 17. 常见错误

- 所有结构都使用 div；
- 链接和按钮职责混用；
- input 没有关联 label；
- 使用 placeholder 代替字段名称；
- 固定宽度导致移动端溢出；
- 大量 absolute 定位拼页面；
- 用 `opacity: 0` 隐藏仍可点击的控件；
- 无替代地删除焦点样式；
- 选择器过深并依赖大量 `!important`。

## 18. 给 AI 的开发指令

```text
请在现有 Vue 3 项目中实现响应式 Todo 页面。
使用 main、section、form、label、button 等语义化元素。
移动优先，列表用 Grid，工具栏用 Flexbox，使用项目现有设计令牌。
所有输入有 label，图标按钮有可访问名称，键盘焦点清晰。
不要用固定像素拼接整页，不要增加无必要依赖。
完成后检查窄屏、键盘操作、颜色对比和 type-check/build。
```

## 19. 面试表达

> HTML 负责结构和语义，CSS 负责布局和视觉，Vue 在二者之上提供状态和组件化。

> Flexbox 适合一维排列，Grid 适合二维布局。实际页面通常组合使用。

> 盒模型由 content、padding、border 和 margin 组成，border-box 让声明尺寸包含 padding 与 border。

> `opacity: 0` 仍占布局并可能接收点击，不能等同于 `display: none`。

> 可访问性首先使用正确原生元素、label、键盘焦点和可访问名称，而不是事后大量补 ARIA。

[进入下一课：TypeScript →](./05-TypeScript基础与类型系统.md)
