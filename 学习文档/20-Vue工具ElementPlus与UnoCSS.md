# 20｜Vue 官方工具、Element Plus 与 UnoCSS

> 所属阶段：前端界面工程化与主线收束
>
> 本课合并知识点：Vue - Official、Vue DevTools、vue-tsc、Element Plus、UnoCSS、后台页面结构、主题与可访问性
>
> 前置知识：HTML/CSS、TypeScript、Vite、Vue 3、Router、Pinia、Axios
>
> 学习目标：理解前端开发工具、组件库和原子化 CSS 分别负责什么，能指挥 AI 组合出一致、可维护的管理页面。

[← pytest、httpx、Apifox 与 OpenAPI](./19-接口测试与OpenAPI协作.md)

## 1. 三类工具不要混成一个概念

```text
Vue - Official / Vue DevTools / vue-tsc → 写代码、检查类型、观察运行状态
Element Plus                        → 表单、表格、弹窗等成品交互组件
UnoCSS                              → 根据类名生成布局、间距、颜色等 CSS
```

TodoLab 管理后台的“待办列表页”可以用 Vue 负责状态和组件，用 Element Plus 提供表格/表单，用 UnoCSS 布局和间距。它们都不替代 API 契约、业务权限或后端验证。

## 2. Vue - Official 扩展解决什么

在 VS Code 中，Vue - Official 为 `.vue` 单文件组件提供语法高亮、组件/属性类型提示、模板表达式检查和跳转。它是 Vue 3 项目的官方推荐 IDE 扩展，旧的 Vetur 面向 Vue 2，通常不应在同一 Vue 3 项目中同时启用造成冲突。

若编辑器报错但命令行正常，先确认：打开的是项目根目录、依赖已安装、`tsconfig` 正确、扩展启用且工作区使用预期的 TypeScript 版本。IDE 报错是线索，最终仍要以项目构建和类型检查结果核对。

## 3. Vue DevTools 观察运行时

Vue DevTools 可查看组件树、props、响应式状态、Pinia Store、Router 状态以及组件更新情况。适合定位“接口已返回数据但页面没变”的问题：

```text
浏览器 Network：请求/响应有没有问题？
Vue DevTools：数据进入组件或 Store 了吗？
组件模板：v-if、v-for、key、计算属性是否正确？
```

DevTools 是开发排错工具，不是生产监控。可用浏览器扩展或 Vite 插件，具体版本与当前 Vite 版本的兼容要求要看官方文档，不应为了安装最新版插件盲目升级整个工程。

## 4. `vue-tsc` 与 `tsc` 的区别

`tsc` 处理普通 TypeScript 文件；Vue SFC 的 `<script setup>` 与模板还需要 Vue 相关的类型分析。`vue-tsc` 可在命令行对 Vue 单文件组件进行类型检查。

```json
{
  "scripts": {
    "typecheck": "vue-tsc --noEmit",
    "build": "vue-tsc --noEmit && vite build"
  }
}
```

实际脚本要与项目现有 `tsconfig`、`vite.config.ts` 和工具版本匹配。类型检查能发现类型不匹配，却不能保证后端实际 JSON 一定符合类型；API 响应仍需契约测试和必要的运行时校验。

## 5. Element Plus 的定位

Element Plus 是 Vue 3 组件库，提供按钮、输入框、表单、表格、分页、弹窗、抽屉、消息提示等常见后台组件。它提供交互和视觉基础，但业务数据加载、字段映射、权限、错误处理仍由应用负责。

引入组件库能加快后台界面开发，也会带来主题、包体积、升级兼容性和设计一致性问题。不要把每个简单元素都替换为组件库组件；先看页面需要的交互复杂度。

## 6. 一个直观的安装方式

```bash
pnpm add element-plus
```

教学阶段可用全量注册理解组件：

```ts
// src/main.ts
import { createApp } from 'vue'
import ElementPlus from 'element-plus'
import 'element-plus/dist/index.css'
import App from './App.vue'

createApp(App).use(ElementPlus).mount('#app')
```

大型项目通常评估按需引入和自动导入插件以减小打包体积；插件配置随版本变化，应以当前官方文档及项目锁文件为准。图标包需按项目需要单独安装，不能假定 Element Plus 安装后所有图标都已可用。

## 7. 表单：前端校验和后端校验的分工

```vue
<script setup lang="ts">
import { reactive, ref } from 'vue'
import type { FormInstance, FormRules } from 'element-plus'

const formRef = ref<FormInstance>()
const form = reactive({ title: '' })
const rules: FormRules<typeof form> = {
  title: [
    { required: true, message: '请输入标题', trigger: 'blur' },
    { max: 120, message: '最多 120 个字符', trigger: 'blur' },
  ],
}

async function submit() {
  const valid = await formRef.value?.validate().catch(() => false)
  if (!valid) return
  // 调用 API；根据状态处理成功和错误。
}
</script>

<template>
  <el-form ref="formRef" :model="form" :rules="rules" label-width="80px">
    <el-form-item label="标题" prop="title">
      <el-input v-model="form.title" maxlength="120" show-word-limit />
    </el-form-item>
    <el-button type="primary" @click="submit">创建</el-button>
  </el-form>
</template>
```

前端规则改善输入体验，但请求可以绕过浏览器，后端仍需 Pydantic 校验。`validate()` 结果、错误提示和请求中状态需要处理；不要把“按钮可点击”误认为提交一定成功。

## 8. 表格、分页与服务端数据

```vue
<template>
  <el-table :data="todos" v-loading="loading" row-key="id">
    <el-table-column prop="title" label="标题" min-width="220" />
    <el-table-column prop="completed" label="状态" width="100">
      <template #default="{ row }">
        <el-tag :type="row.completed ? 'success' : 'info'">
          {{ row.completed ? '完成' : '待办' }}
        </el-tag>
      </template>
    </el-table-column>
  </el-table>
  <el-pagination
    v-model:current-page="page"
    :page-size="pageSize"
    :total="total"
    layout="prev, pager, next"
  />
</template>
```

上面只展示模板，`todos`、`loading`、`page`、`pageSize`、`total` 仍需在脚本中声明并接 API。分页应明确后端返回的是总数还是游标，页码从 0 还是 1 开始。排序/筛选参数变化时通常要重置页码并重新请求，避免当前页越界。

表格还要有空状态、错误状态和重新加载入口；不能只显示旋转图标。大量行数据应使用服务端分页或适合的虚拟列表，而不是一次取全量再让浏览器硬渲染。

## 9. 对话框与删除确认

删除 Todo 时，弹窗只是一层防误触体验。服务端仍要校验用户身份、资源所有权和业务规则。异步删除过程中应禁用重复提交；失败时保留或恢复界面状态，不能先在页面删掉一行就默认数据库已成功删除。

组件库消息提示适合短反馈，复杂错误应在页面相应位置显示。无论用对话框还是抽屉，都要考虑键盘焦点、关闭时未保存的数据和移动端布局。

## 10. UnoCSS 是如何工作的

UnoCSS 是按需生成 CSS 的工具。它扫描源码中的类名，按配置的预设/规则生成对应样式。例如：

```html
<section class="mx-auto max-w-6xl p-4 md:p-8">
  <h1 class="text-xl font-semibold">我的待办</h1>
</section>
```

`p-4`、`text-xl` 等类名需要有对应预设或自定义规则。UnoCSS 不负责表单状态、分页逻辑、权限和响应式数据；它只是样式生成层。

## 11. 在 Vite + Vue 中接入 UnoCSS

```bash
pnpm add -D unocss
```

```ts
// vite.config.ts：保留项目原有插件，只加入 UnoCSS()
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import UnoCSS from 'unocss/vite'

export default defineConfig({
  plugins: [vue(), UnoCSS()],
})
```

```ts
// uno.config.ts
import { defineConfig, presetWind3 } from 'unocss'

export default defineConfig({
  presets: [presetWind3()],
})
```

```ts
// src/main.ts
import 'virtual:uno.css'
```

这三个位置分别是 Vite 插件、样式规则、运行入口引入生成的 CSS。若项目已有 UnoCSS 配置，先合并而不是覆盖。`presetWind3` 是一个预设选择；版本升级前查兼容性与弃用说明。

## 12. 动态类名为什么可能失效

UnoCSS 依赖静态扫描识别类名。如果写：

```ts
const className = `bg-${color}-500`
```

构建工具未必能看见完整的 `bg-blue-500`，生成的 CSS 可能缺失。更可靠的是明确映射：

```ts
const colorClass = {
  success: 'bg-green-100 text-green-800',
  danger: 'bg-red-100 text-red-800',
} as const
```

确需动态组合时使用 UnoCSS 的 safelist 或规则，并检查生产构建产物。不要为了“写得短”牺牲可预测性。

## 13. 组件库样式与 UnoCSS 如何分工

建议边界：Element Plus 管复杂交互组件的基础视觉和行为；UnoCSS 管页面布局、间距、宽度、响应式和少量常规文本样式；全局 CSS/主题变量管品牌颜色、字体和跨组件规则。

```text
Element Plus：el-form、el-table、el-dialog
UnoCSS：grid、gap、padding、max-width、responsive
主题变量：主色、语义色、圆角和文字规范
```

避免在每个页面用大量高优先级 CSS 覆盖 Element Plus 内部结构；升级组件库时这种覆盖容易失效。先查官方主题定制方式，必要时封装项目自己的业务组件。

## 14. 一个完整后台页面应有的状态

待办列表页面至少要区分：

```text
首次加载 → 有数据 / 空数据 / 加载失败
操作中   → 禁止重复提交、显示反馈
操作成功 → 更新列表或失效缓存
操作失败 → 保留用户输入、可重试
未登录   → 重新认证或跳转
无权限   → 明确提示，不无限重试
```

只画“正常有数据”的截图很容易遗漏真实工作中最常见的状态。UI 也要处理长标题、窄屏和网络慢；这些不靠组件库自动完成。

## 15. 设计系统和可访问性

组件库提供一致的默认视觉，但项目仍需统一间距、字号、颜色、交互状态和语言。不要混用多个组件库的默认外观；将常用业务组合封装为 `TodoForm`、`TodoTable` 等组件。

可访问性包括：表单标签与错误信息、键盘可操作、焦点可见、足够的颜色对比、状态不只靠颜色表达。比如完成状态除了绿色，还应有“完成”文字。适配移动端时可以考虑表格折叠为卡片，而不是只让页面横向滚动。

## 16. 图标、国际化与日期格式

图标要有含义且风格一致；纯图标按钮应提供可访问名称。中文后台若使用组件库日期选择器、分页等，注意语言包和日期格式。后端日期通常传带时区的 ISO 字符串；界面展示的本地化格式由前端统一处理，不能直接拿浏览器本地时区当数据库真相。

## 17. 性能与包体积

对常用后台项目，先保证正确性和可维护性，再针对实际测量结果优化。可关注：首屏 JS/CSS 体积、路由懒加载、大型表格渲染、重复请求、图标引入方式和组件库按需引入。

UnoCSS 按需生成样式不意味着整个前端一定更小；Element Plus 全量注册也不一定立即造成不可接受的体积。用 Vite 构建报告和浏览器性能工具比较，再决定是否引入自动导入、拆包或虚拟列表。

## 18. 一次具体排错：页面有请求但没有表格行

1. 浏览器 Network 看响应状态和 JSON 字段；
2. Axios 层是否只返回 `response.data`，类型是否与真实响应一致；
3. 组件里 `todos` 是数组还是包了一层 `data.items`；
4. DevTools 看响应式状态是否更新；
5. `el-table :data` 是否绑定正确、`row-key` 是否稳定；
6. 空态、错误态是否被 `v-if` 错误条件覆盖。

不要一开始就改 CSS；先判断“没数据”还是“数据被隐藏”。

## 19. 一次具体排错：UnoCSS 类名开发有效、构建后失效

检查 `uno.config.ts` 是否被加载、`virtual:uno.css` 是否在入口引入、类名是否由字符串拼接生成、构建扫描范围是否包含该文件，以及样式是否被更高优先级规则覆盖。开发与生产构建都要看最终元素 class 和 computed style。

## 20. 指挥 AI 做可维护的后台页面

```text
请在现有 Vue 3 + TypeScript + Vite 项目中实现 Todo 管理页。
先读取项目已有 Router、Pinia、Axios、主题和组件约定；
使用 Element Plus 表单/表格/弹窗，UnoCSS 只负责布局与间距。
覆盖加载、空列表、请求失败、无权限、提交中和删除失败状态；
前端验证与后端契约保持一致，但不把前端校验当安全边界。
保持组件职责清晰，避免把 API 调用、表单和表格都塞进一个巨大组件。
完成后运行项目现有 typecheck、build 与测试，报告真实结果。
```

```text
请审查这个管理页的可访问性和一致性：
检查表单标签、错误提示、键盘焦点、图标按钮名称、颜色对比、
窄屏布局、长文本和空/错/加载状态；只提出有依据的最小改动。
```

## 21. 面试表达速记

> Vue - Official 改善 `.vue` 文件的编辑体验，Vue DevTools 看运行时组件/状态，`vue-tsc` 在 CI 中做单文件组件的类型检查；三者分别解决不同层面的问题。

> Element Plus 提供复杂交互组件，UnoCSS 根据类名生成布局和基础样式。组件库不负责业务逻辑，原子化 CSS 也不负责状态管理，二者要按职责分工。

> 前端表单校验是体验，后端校验是数据与安全边界。后台页面不能只处理成功状态，还要处理加载、空态、错误、无权限和重复提交。

> UnoCSS 对动态拼接类名可能无法静态识别；应使用明确映射或 safelist，并验证生产构建后的样式。

## 22. 技术地图与官方资料

```text
IDE：Vue - Official → 写组件、看类型
运行：Vue DevTools  → 看组件树/Store/Router
检查：vue-tsc       → CI 类型检查
界面：Element Plus  → 表单/表格/弹窗
样式：UnoCSS        → 布局/间距/响应式
数据：Axios + API   → 请求与真实业务状态
```

- [Vue：工具链](https://vuejs.org/guide/scaling-up/tooling)
- [Vue DevTools](https://devtools.vuejs.org/)
- [Element Plus：安装](https://element-plus.org/en-US/guide/installation.html)
- [Element Plus：表单](https://element-plus.org/en-US/component/form.html)
- [UnoCSS：Vite 插件](https://unocss.dev/integrations/vite)
- [UnoCSS：Wind3 预设](https://unocss.dev/presets/wind3)

## 23. 主线完成后怎么使用这些知识

至此，按当前大纲合并的 20 节主线已经完成。实际工作中可以从一个需求出发，沿这条链检查：

```text
需求 → Vue 页面/状态 → Axios/HTTP/JSON → FastAPI/MVC 分层
    → 认证/权限 → SQLAlchemy/PostgreSQL → pytest/契约
    → Git/CI → Docker/Linux 运行
```

你不必一次背完所有参数。看到新任务时，先定位它属于哪一层、数据从哪里来、边界在哪里、如何验证；再让 AI 根据项目现状实现，并用本系列文档审查它有没有遗漏。
