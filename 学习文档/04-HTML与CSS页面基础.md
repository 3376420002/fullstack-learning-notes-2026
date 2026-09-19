# 04｜HTML 与 CSS 页面基础

> 所属阶段：View 层基础  
> 本课合并知识点：HTML、CSS、表单、语义化、无障碍、响应式布局、浏览器调试  
> 前置知识：MVC、HTTP / HTTPS、JSON  
> 本课目标：能够从零编写结构合理、样式清晰、可响应、可访问的静态页面，并理解它如何演进为 Vue 组件。

[← JSON 数据格式](./03-JSON数据格式.md) · [学习首页](../README.md) · [TypeScript →](./05-TypeScript基础与类型系统.md)

## 1. 为什么把 HTML 和 CSS 放在一起学习

HTML 和 CSS 职责不同，但在页面开发中几乎总是一起工作：

```text
HTML：页面有什么、它们是什么
CSS：这些内容如何排列、如何呈现
JavaScript / TypeScript：页面如何响应行为和数据变化
```

例如一个“创建任务”按钮：

```html
<button type="submit" class="todo-form__submit">创建任务</button>
```

- `<button>` 告诉浏览器和辅助技术“这是一个可操作按钮”；
- `type="submit"` 说明它会提交表单；
- CSS 类决定按钮的颜色、间距、大小和交互状态；
- 将来的 TypeScript / Vue 决定点击后发送什么请求以及如何更新数据。

如果只学习视觉样式而忽略 HTML 语义，页面可能“看起来像按钮”，但键盘无法操作、表单无法提交、屏幕阅读器也无法理解。

## 2. 浏览器如何把代码变成页面

浏览器渲染页面的简化过程：

```text
读取 HTML → 构建 DOM 树
读取 CSS  → 构建样式规则
DOM + CSS → 计算布局和绘制
JavaScript → 修改 DOM、状态和交互
```

DOM（Document Object Model）是 HTML 在浏览器内存中的树状表示：

```html
<main>
  <h1>Todo 列表</h1>
  <ul>
    <li>学习 HTML</li>
  </ul>
</main>
```

对应的结构可以理解为：

```text
main
├─ h1
│  └─ "Todo 列表"
└─ ul
   └─ li
      └─ "学习 HTML"
```

Vue 的模板最终也会转化为浏览器能够渲染和更新的 DOM。

## 3. HTML 文档的基本骨架

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta
      name="description"
      content="用于学习 HTML 与 CSS 的 Todo 页面"
    />
    <title>Todo 学习项目</title>
    <link rel="stylesheet" href="./styles.css" />
  </head>
  <body>
    <main>
      <h1>Todo 学习项目</h1>
    </main>
  </body>
</html>
```

关键部分：

| 代码 | 作用 |
| --- | --- |
| `<!doctype html>` | 声明使用现代 HTML 标准模式 |
| `<html lang="zh-CN">` | 告知浏览器和辅助技术页面主要语言 |
| `<meta charset="UTF-8">` | 指定字符编码，避免中文乱码 |
| viewport meta | 让移动端按设备宽度正确布局 |
| `<title>` | 浏览器标签、收藏和搜索结果标题 |
| stylesheet link | 引入外部 CSS 文件 |
| `<body>` | 用户实际看到的页面内容 |

`<head>` 中主要放文档元数据和资源引用，不是页面正文。

## 4. 元素、属性与内容

```html
<a href="/todos/42" class="todo-link">查看任务</a>
```

- 元素：`a`；
- 属性：`href` 和 `class`；
- 内容：`查看任务`；
- 开始标签：`<a ...>`；
- 结束标签：`</a>`。

有些元素没有结束标签，例如：

```html
<img src="todo.png" alt="Todo 页面截图" />
<input type="text" name="title" />
<meta charset="UTF-8" />
```

HTML 属性值建议统一使用双引号，缩进保持一致。

## 5. 语义化 HTML

语义化的含义是：根据内容的真实角色选择元素，而不是只看默认外观。

### 5.1 页面结构元素

| 元素 | 适用场景 |
| --- | --- |
| `<header>` | 页面或区域的头部 |
| `<nav>` | 主要导航链接区域 |
| `<main>` | 页面唯一的主要内容 |
| `<section>` | 有主题的一组内容，通常有标题 |
| `<article>` | 可独立理解或分发的内容 |
| `<aside>` | 补充信息、侧栏 |
| `<footer>` | 页面或区域的尾部 |

示例：

```html
<body>
  <header class="site-header">
    <a href="/" class="site-logo">TodoLab</a>
    <nav aria-label="主导航">
      <a href="/todos">任务</a>
      <a href="/profile">个人中心</a>
    </nav>
  </header>

  <main>
    <section aria-labelledby="todo-heading">
      <h1 id="todo-heading">我的任务</h1>
    </section>
  </main>

  <footer>© 2026 TodoLab</footer>
</body>
```

一个页面通常只有一个主要 `<main>`。`section` 最好有能描述其主题的标题。

### 5.2 div 和 span

`div` 和 `span` 没有额外语义：

- `div` 常作为块级分组容器；
- `span` 常用于行内文本片段。

它们不是错误元素，但不应该替代已经存在的语义元素。

不推荐：

```html
<div class="button">保存</div>
```

推荐：

```html
<button type="button">保存</button>
```

真正的 button 天然支持键盘焦点、Enter / Space 操作和禁用状态。

## 6. 标题与文本

```html
<h1>Todo 管理</h1>
<h2>今日任务</h2>
<h3>高优先级</h3>

<p>你今天还有 3 个任务未完成。</p>
<p><strong>注意：</strong>过期任务会显示为红色。</p>
```

标题层级表达文档结构，不是字号工具。不要因为想要更小的字就从 `h1` 直接跳到 `h4`，字号应交给 CSS。

常用文本元素：

| 元素 | 语义 |
| --- | --- |
| `<p>` | 段落 |
| `<strong>` | 重要性 |
| `<em>` | 强调语气 |
| `<code>` | 行内代码 |
| `<pre>` | 保留空白的预格式文本 |
| `<time>` | 日期或时间 |
| `<small>` | 注释、版权等辅助文本 |

## 7. 链接与按钮的区别

判断原则：

- 跳转到另一个地址：使用 `<a>`；
- 在当前页面执行操作：使用 `<button>`。

```html
<a href="/todos/42">查看详情</a>
<button type="button">标记完成</button>
```

不要通过 CSS 把链接和按钮的语义互换。外观可以相似，但语义和键盘行为不同。

在新标签页打开外部链接时：

```html
<a href="https://example.com" target="_blank" rel="noopener noreferrer">
  查看外部资料
</a>
```

## 8. 图片与替代文本

```html
<img
  src="./images/todo-dashboard.png"
  alt="Todo 仪表盘，显示三个未完成任务"
  width="1200"
  height="675"
/>
```

`alt` 的目标是表达图片在当前上下文中的信息：

- 信息型图片：描述它传达的关键信息；
- 纯装饰图片：使用 `alt=""`；
- 不要写“这是一张图片”；
- 图片中的重要文字不能只存在于图片里；
- 提供 width 和 height 有助于浏览器提前保留空间，减少布局跳动。

## 9. 列表

无序列表适合没有先后关系的项目：

```html
<ul>
  <li>学习 HTML</li>
  <li>学习 CSS</li>
</ul>
```

有序列表适合步骤和排名：

```html
<ol>
  <li>填写标题</li>
  <li>选择优先级</li>
  <li>提交任务</li>
</ol>
```

导航菜单、Todo 集合等本质上也是列表，不要用连续的 `<br>` 手工换行模拟。

## 10. 表格

表格适合二维数据，不适合页面整体布局。

```html
<table>
  <caption>本周任务统计</caption>
  <thead>
    <tr>
      <th scope="col">日期</th>
      <th scope="col">新增</th>
      <th scope="col">完成</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th scope="row">星期一</th>
      <td>5</td>
      <td>3</td>
    </tr>
  </tbody>
</table>
```

- `caption` 描述表格用途；
- `thead` 和 `tbody` 表达区域；
- `th` 是表头；
- `scope` 帮助辅助技术理解表头与数据的关系。

移动端宽表格可以放入可横向滚动的容器，但不要直接隐藏重要列。

## 11. 表单基础

表单是页面和 HTTP 联系最紧密的 HTML 能力。

```html
<form action="/api/v1/todos" method="post">
  <div class="form-field">
    <label for="todo-title">任务标题</label>
    <input
      id="todo-title"
      name="title"
      type="text"
      minlength="1"
      maxlength="100"
      required
    />
  </div>

  <button type="submit">创建任务</button>
</form>
```

重要属性：

| 属性 | 作用 |
| --- | --- |
| `action` | 原生表单提交到的 URL |
| `method` | 原生表单使用 GET 或 POST |
| `name` | 表单提交时的字段名 |
| `id` | 元素唯一标识，也可与 label 关联 |
| `required` | 浏览器基础必填校验 |
| `minlength` / `maxlength` | 字符串长度限制 |
| `min` / `max` | 数值或日期范围 |
| `autocomplete` | 浏览器自动填写提示 |

`id` 和 `name` 不是同一件事：`id` 用于页面内标识，`name` 决定原生表单提交的字段名。

## 12. label、placeholder 与帮助文本

每个重要输入控件都应该有可见 label：

```html
<label for="email">邮箱</label>
<input
  id="email"
  name="email"
  type="email"
  autocomplete="email"
  aria-describedby="email-help"
/>
<p id="email-help" class="form-help">用于接收任务提醒。</p>
```

placeholder 不能替代 label，因为：

- 用户输入后 placeholder 会消失；
- 颜色通常较淡；
- 它不适合承载字段名称或重要要求；
- 部分辅助技术对它的呈现不一致。

## 13. 常见输入控件

```html
<input type="text" name="title" />
<input type="email" name="email" />
<input type="password" name="password" />
<input type="number" name="priority" min="1" max="5" />
<input type="date" name="due_date" />
<input type="checkbox" name="completed" />
<input type="radio" name="priority" value="high" />
<input type="file" name="attachment" />
<textarea name="description"></textarea>

<select name="status">
  <option value="todo">待办</option>
  <option value="done">完成</option>
</select>
```

不要只因为要限制输入就使用 `type="number"`。电话号码、邮政编码等虽然由数字字符组成，但并不参与数值计算，更适合文本输入配合 `inputmode`。

## 14. fieldset 与一组控件

单选框或复选框组应使用 fieldset：

```html
<fieldset>
  <legend>优先级</legend>

  <label>
    <input type="radio" name="priority" value="low" />
    低
  </label>

  <label>
    <input type="radio" name="priority" value="high" />
    高
  </label>
</fieldset>
```

`legend` 为整组控件提供名称。

## 15. 浏览器校验与服务端校验

```html
<input name="title" minlength="1" maxlength="100" required />
```

浏览器校验可以快速反馈，但它不能替代后端校验：

- 用户可以修改 HTML；
- 客户端可以绕过页面直接发送请求；
- 不同浏览器行为可能存在差异；
- 业务规则通常无法只靠 HTML 属性表达。

正确分工：

```text
HTML / Vue：改善填写体验和即时反馈
Pydantic：验证接口数据契约
Service：验证业务规则
Database：使用约束保护最终数据一致性
```

## 16. CSS 规则的结构

```css
.todo-card {
  padding: 1rem;
  border: 1px solid #dbe3ee;
  border-radius: 0.75rem;
  background-color: #ffffff;
}
```

- 选择器：`.todo-card`；
- 声明块：花括号内的内容；
- 属性：`padding`；
- 值：`1rem`；
- 一条声明以分号结尾。

## 17. 常用选择器

```css
/* 元素选择器 */
button { }

/* 类选择器 */
.todo-card { }

/* ID 选择器 */
#todo-title { }

/* 属性选择器 */
input[type="email"] { }

/* 后代选择器 */
.todo-card p { }

/* 直接子元素 */
.todo-list > li { }

/* 多个选择器 */
h1,
h2 { }
```

项目样式优先使用 class。ID 的优先级较高，不适合大量用于可复用组件样式。

## 18. 伪类与伪元素

伪类描述状态：

```css
.button:hover { }
.button:focus-visible { }
.button:disabled { }
.todo-item:first-child { }
.form-input:invalid { }
```

伪元素创建可样式化的虚拟部分：

```css
.required-label::after {
  content: " *";
  color: #dc2626;
}
```

重要信息不要只通过 `::before` 或 `::after` 的 content 提供，因为辅助技术对生成内容的处理可能不一致。

## 19. 层叠、优先级与源码顺序

当多个规则匹配同一个元素时，浏览器需要决定使用哪一个值。主要考虑：

1. 规则来源和重要性；
2. 选择器优先级；
3. 如果优先级相同，后出现的规则生效。

简化优先级认识：

```text
内联 style > ID > class / 属性 / 伪类 > 元素 / 伪元素
```

不推荐依赖越来越复杂的选择器或大量 `!important` 解决问题：

```css
#app .page .todo-list li.todo-item.active {
  color: red !important;
}
```

更易维护：

```css
.todo-item--active {
  color: #b91c1c;
}
```

## 20. 继承

部分属性会从父元素继承，例如字体和文字颜色：

```css
body {
  color: #172033;
  font-family: system-ui, sans-serif;
}
```

子元素通常会继承这些值。

尺寸、边框、背景、margin 等通常不会自动继承。浏览器开发者工具的 Computed 面板可以查看最终值及其来源。

## 21. 盒模型

每个普通元素都可以理解为一个盒子：

```text
margin  外边距
└─ border  边框
   └─ padding  内边距
      └─ content  内容区
```

推荐全局设置：

```css
*,
*::before,
*::after {
  box-sizing: border-box;
}
```

使用 `border-box` 后，设置的 width 和 height 会包含 padding 与 border，布局更容易计算。

示例：

```css
.todo-card {
  width: 20rem;
  padding: 1rem;
  border: 1px solid #dbe3ee;
  margin-block: 1rem;
}
```

## 22. display 与普通文档流

常见 display 值：

| 值 | 特点 |
| --- | --- |
| `block` | 通常独占一行，可设置宽高 |
| `inline` | 跟随文字排列，宽高行为受限 |
| `inline-block` | 行内排列，同时可设置宽高 |
| `flex` | 一维布局 |
| `grid` | 二维布局 |
| `none` | 不参与布局，也通常不会被辅助技术读取 |

页面应优先依赖普通文档流、Flexbox 和 Grid。不要一开始就用大量绝对定位拼页面。

## 23. Flexbox：一维布局

Flexbox 适合沿一个主轴排列项目，例如工具栏、按钮组和卡片行。

```css
.todo-toolbar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}
```

核心概念：

- `flex-direction`：主轴方向；
- `justify-content`：主轴对齐；
- `align-items`：交叉轴对齐；
- `gap`：项目间距；
- `flex-wrap`：空间不足时是否换行；
- `flex`：项目如何放大、缩小和设置基础尺寸。

响应式按钮组：

```css
.actions {
  display: flex;
  flex-wrap: wrap;
  gap: 0.75rem;
}
```

优先使用 `gap`，而不是给每个子元素手工添加不对称 margin。

## 24. Grid：二维布局

Grid 适合同时控制行和列，例如卡片网格、仪表盘和完整页面区域。

```css
.todo-grid {
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 1rem;
}
```

自动适应可用宽度：

```css
.todo-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(16rem, 1fr));
  gap: 1rem;
}
```

简单判断：

```text
主要控制一行或一列 → Flexbox
需要明确的行列网格   → Grid
```

两者可以在同一页面嵌套使用。

## 25. 定位 position

| 值 | 说明 |
| --- | --- |
| `static` | 默认，遵循普通文档流 |
| `relative` | 保留原位置，可作为绝对定位参照 |
| `absolute` | 脱离普通流，相对定位祖先排列 |
| `fixed` | 相对视口固定 |
| `sticky` | 达到阈值后在滚动容器中粘住 |

示例：卡片右上角状态标记：

```css
.todo-card {
  position: relative;
}

.todo-card__badge {
  position: absolute;
  inset-block-start: 0.75rem;
  inset-inline-end: 0.75rem;
}
```

绝对定位适合局部覆盖，不适合替代整个页面布局。脱离文档流的元素过多会导致内容变化时互相遮挡。

## 26. 尺寸单位

| 单位 | 适用场景 |
| --- | --- |
| `px` | 边框、图标等精确小尺寸 |
| `rem` | 字体、间距、圆角等与根字号相关的尺寸 |
| `em` | 相对当前元素字号 |
| `%` | 相对包含块的比例 |
| `vw` / `vh` | 相对视口尺寸，需注意移动端行为 |
| `fr` | Grid 中分配剩余空间 |
| `ch` | 与字符宽度相关，适合控制文本行长 |

示例：

```css
.page-shell {
  width: min(100% - 2rem, 72rem);
  margin-inline: auto;
}

.article {
  max-width: 68ch;
}
```

不要把所有尺寸都写死为 px。页面容器通常需要最大宽度和弹性宽度共同工作。

## 27. CSS 逻辑属性

```css
.todo-card {
  margin-inline: auto;
  padding-block: 1rem;
  padding-inline: 1.25rem;
  border-inline-start: 0.25rem solid #2563eb;
}
```

- `inline` 大致对应文字行方向；
- `block` 大致对应段落堆叠方向。

逻辑属性比固定的 left / right 更容易适配不同书写方向。

## 28. 颜色、字体与可读性

```css
:root {
  color-scheme: light;
  font-family: Inter, "PingFang SC", "Microsoft YaHei", system-ui, sans-serif;
  line-height: 1.6;
}

body {
  color: #172033;
  background: #f5f7fb;
}
```

可读性原则：

- 正文保持足够字号和行高；
- 控制每行文字长度；
- 文字与背景保持足够对比；
- 不只依靠颜色表达状态；
- 避免大段居中正文；
- 避免使用过多字体和字重。

错误状态可以同时使用颜色、图标和文字：

```html
<p class="form-error" role="alert">标题不能为空。</p>
```

## 29. CSS 自定义属性

CSS 自定义属性可以集中管理设计值：

```css
:root {
  --color-primary: #2563eb;
  --color-text: #172033;
  --color-muted: #637083;
  --color-border: #dbe3ee;
  --radius-md: 0.75rem;
  --shadow-card: 0 0.75rem 2rem rgb(15 23 42 / 0.08);
}

.button-primary {
  color: white;
  background: var(--color-primary);
  border-radius: var(--radius-md);
}
```

这些变量会为后面的 Vue 组件库、主题切换和设计系统打基础。

## 30. 响应式设计

响应式设计不是为某几款手机单独写页面，而是让布局根据可用空间合理变化。

移动优先示例：

```css
.page-layout {
  display: grid;
  gap: 1rem;
}

@media (min-width: 48rem) {
  .page-layout {
    grid-template-columns: 16rem minmax(0, 1fr);
  }
}
```

基础规则先适配窄屏；空间足够时再增加侧栏。

响应式检查清单：

- 页面是否出现非预期横向滚动；
- 按钮和输入框是否容易触控；
- 文本是否过小；
- 表格、代码块和长 URL 是否溢出；
- 网格是否能合理折行；
- 图片是否能缩放；
- 固定定位是否遮挡内容。

通用图片规则：

```css
img {
  display: block;
  max-width: 100%;
  height: auto;
}
```

## 31. 用户偏好媒体查询

减少动画偏好：

```css
@media (prefers-reduced-motion: reduce) {
  *,
  *::before,
  *::after {
    scroll-behavior: auto !important;
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

后续还可以使用 `prefers-color-scheme` 实现深浅主题，但主题不能只做颜色反转，还需要重新检查对比度、阴影、边框和图片。

## 32. 键盘焦点

不要移除焦点轮廓后什么都不补：

```css
:focus-visible {
  outline: 3px solid #93c5fd;
  outline-offset: 3px;
}
```

键盘用户需要知道当前焦点在哪里。一个基础页面应该能够使用 Tab 顺序访问链接、按钮和表单控件，并使用 Enter 或 Space 操作。

## 33. 隐藏内容的不同方式

| 方法 | 视觉呈现 | 是否占空间 | 辅助技术通常能否读取 |
| --- | --- | --- | --- |
| `display: none` | 不显示 | 否 | 否 |
| `visibility: hidden` | 不显示 | 是 | 通常否 |
| `opacity: 0` | 透明 | 是 | 可能仍可交互和读取 |
| 仅视觉隐藏工具类 | 不显示 | 几乎不占 | 是 |

不要只使用 `opacity: 0` 隐藏可交互元素，否则用户可能通过键盘聚焦到一个看不见的控件。

屏幕阅读器专用文本工具类：

```css
.sr-only {
  position: absolute;
  width: 1px;
  height: 1px;
  padding: 0;
  margin: -1px;
  overflow: hidden;
  clip: rect(0, 0, 0, 0);
  white-space: nowrap;
  border: 0;
}
```

## 34. HTML 与 CSS 命名和组织

一个简单、可持续的类名结构：

```html
<article class="todo-card todo-card--completed">
  <h2 class="todo-card__title">学习 HTML</h2>
  <p class="todo-card__meta">今天截止</p>
</article>
```

- `todo-card`：组件；
- `todo-card__title`：组件内部元素；
- `todo-card--completed`：组件变体。

不必机械使用某一种命名法，但类名应该表达角色，而不是临时外观。

不推荐：

```html
<div class="blue-box-left-20">...</div>
```

推荐：

```html
<aside class="todo-sidebar">...</aside>
```

## 35. 一个完整的 Todo 页面示例

下面的示例把本课知识整合在一起。它是静态页面，还没有接入 TypeScript 和 API。

示例中的原生 `<form method="post">` 默认按表单格式提交，而不是 JSON。学习 TypeScript 和 Vue 后，我们会拦截 submit 事件，把字段组成对象，再通过 Axios 以 `application/json` 发送给 FastAPI。原生 HTML 表单本身只直接支持 GET 和 POST，PUT、PATCH、DELETE 通常由 JavaScript API 请求完成。

### 35.1 HTML

```html
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta
      name="description"
      content="HTML 与 CSS 课程的响应式 Todo 页面"
    />
    <title>TodoLab</title>
    <link rel="stylesheet" href="./styles.css" />
  </head>

  <body>
    <a class="skip-link" href="#main-content">跳到主要内容</a>

    <header class="site-header">
      <div class="site-header__inner">
        <a class="brand" href="/" aria-label="TodoLab 首页">TodoLab</a>
        <nav class="main-nav" aria-label="主导航">
          <a aria-current="page" href="/todos">任务</a>
          <a href="/statistics">统计</a>
          <a href="/profile">我的账户</a>
        </nav>
      </div>
    </header>

    <main id="main-content" class="page-shell">
      <section class="hero" aria-labelledby="page-title">
        <div>
          <p class="eyebrow">2026 全栈学习项目</p>
          <h1 id="page-title">我的任务</h1>
          <p class="hero__description">
            使用语义化 HTML 描述内容，使用 CSS 完成布局和视觉呈现。
          </p>
        </div>

        <dl class="summary" aria-label="任务概览">
          <div>
            <dt>待完成</dt>
            <dd>3</dd>
          </div>
          <div>
            <dt>已完成</dt>
            <dd>5</dd>
          </div>
        </dl>
      </section>

      <div class="content-grid">
        <aside class="panel" aria-labelledby="create-heading">
          <h2 id="create-heading">创建任务</h2>

          <form class="todo-form" action="/api/v1/todos" method="post">
            <div class="form-field">
              <label for="todo-title">任务标题</label>
              <input
                id="todo-title"
                name="title"
                type="text"
                minlength="1"
                maxlength="100"
                placeholder="例如：完成 HTML 练习"
                required
              />
            </div>

            <div class="form-field">
              <label for="todo-description">任务说明</label>
              <textarea
                id="todo-description"
                name="description"
                rows="4"
                aria-describedby="description-help"
              ></textarea>
              <p id="description-help" class="form-help">
                可选，简单记录完成标准。
              </p>
            </div>

            <div class="form-field">
              <label for="todo-priority">优先级</label>
              <select id="todo-priority" name="priority">
                <option value="1">低</option>
                <option value="2" selected>中</option>
                <option value="3">高</option>
              </select>
            </div>

            <button class="button button--primary" type="submit">
              创建任务
            </button>
          </form>
        </aside>

        <section class="panel" aria-labelledby="list-heading">
          <div class="section-heading">
            <div>
              <p class="eyebrow">今日计划</p>
              <h2 id="list-heading">任务列表</h2>
            </div>
            <button class="button button--secondary" type="button">
              只看未完成
            </button>
          </div>

          <ul class="todo-list">
            <li>
              <article class="todo-card">
                <div class="todo-card__content">
                  <h3>学习语义化 HTML</h3>
                  <p>完成标题、导航、表单和列表练习。</p>
                  <p class="todo-card__meta">
                    <span class="status status--high">高优先级</span>
                    <time datetime="2026-09-18">9 月 18 日截止</time>
                  </p>
                </div>
                <button class="button button--secondary" type="button">
                  标记完成
                </button>
              </article>
            </li>

            <li>
              <article class="todo-card todo-card--completed">
                <div class="todo-card__content">
                  <h3>理解 CSS 盒模型</h3>
                  <p>使用开发者工具检查 content、padding 和 margin。</p>
                  <p class="todo-card__meta">
                    <span class="status status--done">已完成</span>
                  </p>
                </div>
                <button class="button button--secondary" type="button">
                  恢复任务
                </button>
              </article>
            </li>
          </ul>
        </section>
      </div>
    </main>

    <footer class="site-footer">
      <p>TodoLab · HTML 与 CSS 学习项目</p>
    </footer>
  </body>
</html>
```

### 35.2 CSS

```css
:root {
  color-scheme: light;
  font-family: Inter, "PingFang SC", "Microsoft YaHei", system-ui, sans-serif;
  line-height: 1.6;
  font-synthesis: none;

  --color-primary: #2563eb;
  --color-primary-dark: #1d4ed8;
  --color-text: #172033;
  --color-muted: #637083;
  --color-border: #dbe3ee;
  --color-surface: #ffffff;
  --color-background: #f4f7fb;
  --color-danger: #b91c1c;
  --color-success: #047857;
  --radius-sm: 0.5rem;
  --radius-md: 0.875rem;
  --shadow-card: 0 0.75rem 2rem rgb(15 23 42 / 0.08);
}

*,
*::before,
*::after {
  box-sizing: border-box;
}

html {
  scroll-behavior: smooth;
}

body {
  min-width: 20rem;
  min-height: 100vh;
  margin: 0;
  color: var(--color-text);
  background: var(--color-background);
}

button,
input,
select,
textarea {
  font: inherit;
}

a {
  color: inherit;
}

:focus-visible {
  outline: 3px solid #93c5fd;
  outline-offset: 3px;
}

.skip-link {
  position: fixed;
  inset-block-start: 0.5rem;
  inset-inline-start: 0.5rem;
  z-index: 10;
  padding: 0.625rem 0.875rem;
  color: white;
  background: #111827;
  border-radius: var(--radius-sm);
  transform: translateY(-150%);
}

.skip-link:focus {
  transform: translateY(0);
}

.site-header {
  background: rgb(255 255 255 / 0.92);
  border-block-end: 1px solid var(--color-border);
}

.site-header__inner,
.page-shell {
  width: min(100% - 2rem, 72rem);
  margin-inline: auto;
}

.site-header__inner {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  min-height: 4rem;
}

.brand {
  color: var(--color-primary-dark);
  font-size: 1.25rem;
  font-weight: 800;
  text-decoration: none;
}

.main-nav {
  display: flex;
  flex-wrap: wrap;
  gap: 0.25rem;
}

.main-nav a {
  padding: 0.5rem 0.75rem;
  color: var(--color-muted);
  text-decoration: none;
  border-radius: var(--radius-sm);
}

.main-nav a:hover,
.main-nav a[aria-current="page"] {
  color: var(--color-primary-dark);
  background: #eff6ff;
}

.page-shell {
  padding-block: 2rem 4rem;
}

.hero {
  display: flex;
  flex-wrap: wrap;
  align-items: end;
  justify-content: space-between;
  gap: 1.5rem;
  margin-block-end: 1.5rem;
}

h1,
h2,
h3,
p {
  margin-block-start: 0;
}

h1 {
  margin-block-end: 0.5rem;
  font-size: clamp(2rem, 5vw, 3.5rem);
  line-height: 1.1;
}

h2 {
  font-size: 1.35rem;
}

h3 {
  margin-block-end: 0.25rem;
  font-size: 1.05rem;
}

.eyebrow {
  margin-block-end: 0.25rem;
  color: var(--color-primary-dark);
  font-size: 0.8rem;
  font-weight: 800;
  letter-spacing: 0.08em;
  text-transform: uppercase;
}

.hero__description,
.form-help,
.todo-card__meta {
  color: var(--color-muted);
}

.summary {
  display: flex;
  gap: 0.75rem;
  margin: 0;
}

.summary div {
  min-width: 7rem;
  padding: 0.75rem 1rem;
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
}

.summary dt {
  color: var(--color-muted);
  font-size: 0.8rem;
}

.summary dd {
  margin: 0;
  font-size: 1.5rem;
  font-weight: 800;
}

.content-grid {
  display: grid;
  gap: 1rem;
}

.panel {
  padding: 1.25rem;
  background: var(--color-surface);
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
  box-shadow: var(--shadow-card);
}

.todo-form,
.form-field {
  display: grid;
  gap: 0.5rem;
}

.todo-form {
  gap: 1rem;
}

label {
  font-weight: 700;
}

input,
select,
textarea {
  width: 100%;
  padding: 0.75rem 0.875rem;
  color: var(--color-text);
  background: white;
  border: 1px solid #bac7d8;
  border-radius: var(--radius-sm);
}

textarea {
  resize: vertical;
}

input:focus,
select:focus,
textarea:focus {
  border-color: var(--color-primary);
}

.form-help {
  margin: 0;
  font-size: 0.85rem;
}

.button {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 2.75rem;
  padding: 0.625rem 1rem;
  font-weight: 750;
  border: 1px solid transparent;
  border-radius: var(--radius-sm);
  cursor: pointer;
}

.button--primary {
  color: white;
  background: var(--color-primary);
}

.button--primary:hover {
  background: var(--color-primary-dark);
}

.button--secondary {
  color: var(--color-primary-dark);
  background: white;
  border-color: var(--color-border);
}

.button:disabled {
  cursor: not-allowed;
  opacity: 0.55;
}

.section-heading,
.todo-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
}

.todo-list {
  display: grid;
  gap: 0.75rem;
  padding: 0;
  margin: 0;
  list-style: none;
}

.todo-card {
  padding: 1rem;
  border: 1px solid var(--color-border);
  border-radius: var(--radius-md);
}

.todo-card__content {
  min-width: 0;
}

.todo-card p {
  margin-block-end: 0.5rem;
}

.todo-card--completed h3 {
  color: var(--color-muted);
  text-decoration: line-through;
}

.todo-card__meta {
  display: flex;
  flex-wrap: wrap;
  gap: 0.625rem;
  font-size: 0.85rem;
}

.status {
  display: inline-flex;
  align-items: center;
  padding-inline: 0.5rem;
  border-radius: 999px;
  font-weight: 700;
}

.status--high {
  color: var(--color-danger);
  background: #fef2f2;
}

.status--done {
  color: var(--color-success);
  background: #ecfdf5;
}

.site-footer {
  padding: 1.5rem 1rem;
  color: var(--color-muted);
  text-align: center;
  border-block-start: 1px solid var(--color-border);
}

.site-footer p {
  margin: 0;
}

@media (min-width: 52rem) {
  .content-grid {
    grid-template-columns: minmax(16rem, 21rem) minmax(0, 1fr);
    align-items: start;
  }
}

@media (max-width: 38rem) {
  .section-heading,
  .todo-card {
    align-items: stretch;
    flex-direction: column;
  }
}

@media (prefers-reduced-motion: reduce) {
  html {
    scroll-behavior: auto;
  }

  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

## 36. 用浏览器开发者工具学习

打开页面后按 F12，重点练习以下面板。

### Elements

- 检查 DOM 层级；
- 临时修改 HTML；
- 查看匹配到的 CSS 规则；
- 查看被覆盖的声明；
- 切换伪类状态，如 hover 和 focus；
- 查看盒模型计算结果。

### Computed

- 查看元素最终采用的属性值；
- 追踪继承和值的来源；
- 检查实际宽高、字体和颜色。

### Device Toolbar

- 模拟不同视口宽度；
- 检查响应式断点；
- 检查触控区域和横向滚动；
- 模拟缩放后页面是否仍可使用。

开发者工具中的修改不会自动保存到源码。确定正确后，需要回到文件中修改。

## 37. HTML / CSS 与 Vue 的关系

Vue 单文件组件：

```vue
<script setup lang="ts">
// 状态和行为
</script>

<template>
  <!-- 类似 HTML，但支持 Vue 指令和组件 -->
</template>

<style scoped>
/* CSS；scoped 表示样式主要限制在当前组件 */
</style>
```

学习 Vue 之前必须掌握 HTML 和 CSS，因为：

- Vue template 不是替代 HTML，而是在 HTML 基础上增加模板能力；
- Vue 不会自动修复错误的语义和无障碍问题；
- 组件布局最终仍然依赖盒模型、Flexbox 和 Grid；
- scoped CSS 仍然遵循大部分 CSS 层叠和优先级规则；
- 表单的 label、button type、键盘焦点等基础规则仍然成立。

## 38. 常见错误

### 错误一：所有内容都使用 div

页面缺乏结构语义，键盘和辅助技术体验差。

### 错误二：用 `<br>` 和空格做布局

内容一变化或屏幕变窄，布局立即失效。应使用 margin、padding、Flexbox 或 Grid。

### 错误三：placeholder 替代 label

用户输入后字段含义消失，无障碍体验差。

### 错误四：点击元素都写成 div

缺少按钮默认的键盘、焦点和禁用行为。

### 错误五：一遇到冲突就写 `!important`

短期解决覆盖，长期让样式越来越难修改。应先检查优先级、源码顺序和类名设计。

### 错误六：使用绝对定位完成全部布局

内容高度和屏幕尺寸变化时容易重叠。主布局应使用文档流、Flexbox 和 Grid。

### 错误七：固定宽度导致手机横向滚动

应结合 max-width、百分比、弹性布局和媒体查询。

### 错误八：移除焦点轮廓

键盘用户无法知道焦点位置。需要提供清晰的 `:focus-visible` 样式。

### 错误九：只用颜色表达错误或完成状态

色觉差异用户可能无法区分。应同时使用文字、图标或形状。

### 错误十：认为前端 required 能保护数据

后端仍然必须校验请求和业务规则。

## 39. 本课实操

### 练习 A：语义化改造

把下面的结构改成合理的 HTML：

```html
<div class="header">
  <div class="logo">TodoLab</div>
  <div class="link">任务</div>
</div>
<div class="title">我的任务</div>
<div class="button">创建</div>
```

至少使用 `header`、`nav`、`a`、`main`、`h1` 和 `button` 中的合适元素。

### 练习 B：表单

创建一个 Todo 表单，包含：

- 标题：必填，1～100 字；
- 描述：可选；
- 优先级：低、中、高；
- 截止日期；
- 提交按钮；
- 所有输入都有正确 label；
- 至少一个字段包含帮助文本。

### 练习 C：盒模型

创建宽度为 20rem 的卡片，设置：

- 1rem 内边距；
- 1px 边框；
- 0.75rem 圆角；
- 卡片之间 1rem 间距；
- 全局使用 `box-sizing: border-box`。

在开发者工具中找到盒模型图，解释 content、padding、border 和 margin 的实际尺寸。

### 练习 D：响应式布局

实现以下变化：

```text
窄屏：创建表单在上，任务列表在下
宽屏：创建表单在左，任务列表在右
```

要求使用 Grid 或 Flexbox，不使用 absolute 定位。

### 练习 E：键盘与无障碍检查

只使用键盘完成：

1. 跳到主要内容；
2. 访问导航链接；
3. 填写表单；
4. 提交；
5. 操作任务按钮。

检查每一步是否有清晰焦点，并确认图片、表单、导航和标题拥有正确语义。

## 40. 语义化改造参考答案

```html
<header>
  <a href="/" aria-label="TodoLab 首页">TodoLab</a>
  <nav aria-label="主导航">
    <a href="/todos">任务</a>
  </nav>
</header>

<main>
  <h1>我的任务</h1>
  <button type="button">创建</button>
</main>
```

## 41. 自测问题

进入下一课前，确保你能回答：

- HTML、CSS 和 TypeScript 分别负责什么？
- 为什么应该使用语义化元素？
- 链接和按钮的使用边界是什么？
- label 为什么不能被 placeholder 替代？
- `id` 和 `name` 在表单中有什么区别？
- 浏览器校验为什么不能替代后端校验？
- CSS 层叠和选择器优先级如何影响最终样式？
- 盒模型包含哪四层？
- Flexbox 和 Grid 分别适合什么布局？
- 为什么不应该使用大量 absolute 定位？
- 移动优先响应式设计是什么？
- 为什么不能随意移除焦点轮廓？
- HTML 和 CSS 知识如何迁移到 Vue 单文件组件？

## 42. 验收标准

完成本课后，你应当能够：

- 编写完整、编码正确的 HTML 文档；
- 使用语义元素组织页面、导航、列表、表格和表单；
- 为表单建立 label、帮助文本和基础校验；
- 使用 CSS 选择器、层叠、继承和盒模型；
- 使用 Flexbox 与 Grid 完成常见布局；
- 使用相对单位和媒体查询实现响应式页面；
- 提供清晰的焦点、图片替代文本和键盘操作路径；
- 使用 CSS 自定义属性管理基础设计值；
- 使用浏览器开发者工具检查 DOM、样式和盒模型；
- 读懂完整 Todo 静态页面，并能独立修改其结构与样式。

## 43. 下一步

下一课建议学习 **TypeScript 基础**，并把本课静态 Todo 页面逐步变成有数据和交互的程序。将重点学习：

- 基础类型、数组、对象和函数；
- interface 与 type；
- 联合类型、可选字段和类型收窄；
- 泛型与 `ApiResult<T>`；
- DOM 事件和表单数据；
- 模块化与严格模式；
- JSON 数据契约在 TypeScript 中的表达。
