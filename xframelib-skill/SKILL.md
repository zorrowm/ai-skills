---
name: xframelib-skill
description: 适用于当前 xframelib + Vue 3 + Quasar + Vite 的 Widget 微前端开发、页面/Widget 注册、GIS 地图容器、事件总线、布局容器、生命周期清理、UTF-8 编码约束和工程化验证。Use when modifying this repository's pages, widgets, settings, routing, xframelib layout integration, Cesium/OpenLayers/MapLibre widgets, UTF-8 text files, or when reviewing/fixing project-specific frontend issues.
---

# xframelib + Vue + Quasar Widget 开发规则

本技能用于 `vue-widget-quasar-clivite-template` 这类 xframelib Widget 微前端项目。行动前先读仓库根目录的 `AGENTS.md`，再按现有代码风格实现；不要把生成代码、静态 worker 和运行时配置当作普通业务文件随手改。

## 快速流程

1. 先定位目标所属布局：`back`、`adefault`、`bigScreen`、`portal`、`product`、`sideBar` 或地图相关 Widget。
2. 决定承载位置：页面级生命周期放 `src/pages/**`，可复用业务功能优先放 `src/widgets/**`，公共 UI 放 `src/components/**`，跨组件状态放 `src/stores/modules/**`。
3. 新增 Widget 时同时检查并更新：
   - `src/settings/widgetSetting/**`：Widget 注册与容器选择。
   - `src/settings/widgetMenuSetting/**`：菜单项触发 Widget 时更新。
   - `src/settings/modalSetting/**`：Modal 类组件需要注册。
   - `src/settings/functionSetting.ts`：需要权限/功能点控制时更新。
4. 新增页面时按布局注册路由：`src/router/<layout>/modules/**`，页面组件放到对应 `src/pages/<layout>/**`。
5. 读写 `*.md`、`*.vue`、`*.ts`、`*.js`、`*.json`、`*.scss` 等文本文件时显式保持 UTF-8 编码；在 PowerShell 中读取中文文档优先使用 `Get-Content -Encoding UTF8`，写入或生成文件时也指定 UTF-8。
6. 完成后优先运行与改动范围匹配的验证：`pnpm.cmd typecheck`、`pnpm.cmd lint:check`、`pnpm.cmd build`。在 Windows PowerShell 中优先使用 `pnpm.cmd`，避免 `pnpm.ps1` 执行策略问题。

## xframelib 核心能力边界

xframelib 是来源于业务项目的前端基础库，不绑定具体 UI 库。修改项目时优先复用它已有的能力：

- `Global.Config` 对应 `public/SysConfig.js` 的运行时系统配置；新增服务地址、地图 Key、主题、锁屏、日志开关等配置前先确认部署环境影响。
- `Global.EventBus` 是底层事件总线；业务代码优先通过项目封装的 `@/events` 使用 `EmitMsg`、`OnEventHandler`、`OffEventHandler`。
- `Global.Logger()` 替代长期裸 `console.log`；发布日志开关通常受 `SysConfig.UI.ProductLog` 控制。
- `Global.Axios`、`AxiosHelper` 用于普通 HTTP 请求；Hprose Proxy/RPC 代码用于后台服务代理调用，生成服务代码不要随手改。
- 文件下载/上传、`StorageHelper`、`H5Tool`、`IsTool`、`ValidateTool`、XXTEA 等工具优先复用库能力，不要在业务组件里重复造通用实现。

## 不要手改的区域

- `src/api/**`、`src/service/**` 是接口生成代码；类型问题优先改生成模板或补声明，不要在生成产物里做长期手工修补。
- `public/assets/**` 多为基础依赖库 worker 文件；除非任务明确要求，不要删除、重命名或手改。
- `public/SysConfig.js` 是运行时全局配置，可能含服务地址、密钥和部署差异；改动前确认用途，避免提交敏感信息。
- `.quasar/**`、`dist/**`、`node_modules/**` 是生成/依赖目录，不作为业务改动目标。

## LayoutContainer 容器职责

`LayoutContainer` 是 Page 和 Widget 的挂载层，不是普通业务 DOM。

| 容器 | 职责 | 常见内容 |
| --- | --- | --- |
| `topContainer` | 顶部全局区域 | 标题、顶栏、导航 |
| `bottomContainer` | 底部全局区域 | 状态条、底部菜单、时间轴 |
| `leftContainer` | 左侧业务区域 | 菜单、图层树、任务列表 |
| `rightContainer` | 右侧业务区域 | 属性、详情、分析结果 |
| `mainContainer` | 路由页面区域 | `router-view` 渲染的 Page |
| `backContainer` | 中心后景层 | 地图实例、底图、图层、三维实体 |
| `centerFrontContainer` | 中心前景层 | 浮动工具条、查询面板、覆盖地图的 UI |
| `centerdiv` | 中心层公共类 | `main/back/front` 中心容器都可能带此类 |

容器选择原则：地图实例和图层放 `centerBack`；浮动可交互 UI 放 `centerFront` 或左右侧栏；顶部/底部只放真正全局固定信息。不要用提高 `z-index` 代替容器职责判断。

Layout 使用规则：

- Layout 根组件通常通过 `:widgetConfig="getRightWidgetConfig()"` 和 `:layoutID="layoutID"` 挂载 `LayoutContainer`。
- `@containerLoaded` 会返回当前 `layoutID` 和 `layoutManager`，并触发 `SysEvents.LayoutContainerLoaded`；首次进入页面时如果 `Global.LayoutMap.get(layoutID)` 还没有值，就监听该事件后再加载 Widget。
- 进入布局后按需调用 `Global.Loading("end")` 关闭加载动画，避免页面已渲染但全局 loading 残留。
- `LayoutContainer` 支持 `main`、`back`、`front`、`left`、`right`、`bottom` 具名插槽和 `default` 默认插槽；用插槽做布局固定内容时仍要尊重对应容器职责。

## 事件穿透规则

xframelib 中心层通常有：

```css
.centerdiv {
  pointer-events: none;
}

.centerdiv > * {
  pointer-events: all !important;
}
```

这会让中心层本身穿透，但直接子节点恢复点击。GIS 页面和全屏透明 Widget 如果根节点覆盖整屏，必须继续穿透，否则会挡住 Cesium、OpenLayers、MapLibre 的拖拽、缩放、timeline 或工具条。

Page 根节点模式：

```vue
<template>
  <div class="my-page">
    <div class="my-page-panel">...</div>
  </div>
</template>

<style scoped>
.my-page {
  width: 100%;
  height: 100%;
  pointer-events: none !important;
}

.my-page > * {
  pointer-events: all !important;
}
</style>
```

全屏 Widget 根节点模式：

```vue
<template>
  <div class="my-widget">
    <aside class="left-panel">...</aside>
    <aside class="right-panel">...</aside>
  </div>
</template>

<style scoped>
.my-widget {
  position: absolute;
  inset: 0;
  pointer-events: none !important;
}

.my-widget > * {
  pointer-events: all !important;
}
</style>
```

原则：大容器穿透，真实按钮、面板、表格、工具条接收事件。

## Page 编写规则

Page 是路由入口，适合管理生命周期、当前页面 Widget 编排、少量页面级 UI 和跨 Widget 状态。Page 不必把所有逻辑都拆成 Widget，但地图业务和可复用业务功能优先 Widget 化。

Page 常见职责：

- 获取当前布局的 `LayoutManager`。
- 在 `onMounted` 中加载本页需要的 Widget。
- 在 `onUnmounted` 中卸载本页加载的 Widget。
- 等待地图初始化完成后加载依赖地图的 Widget。
- 清理同一布局下互斥的地图、图层或业务 Widget。

示例：

```ts
import { Global } from "xframelib";
import { onMounted, onUnmounted } from "vue";

const layoutID = "bigScreen";
const widgets = ["cesiumWidget", "menuBarWidget"];

onMounted(() => {
  const layoutManager = Global.getLayoutManager(layoutID);
  widgets.forEach(id => layoutManager?.loadWidget(id));
});

onUnmounted(() => {
  const layoutManager = Global.getLayoutManager(layoutID);
  widgets.forEach(id => layoutManager?.unloadWidget(id));
});
```

## Widget 编写规则

Widget 是主要业务单元。新增功能优先考虑 Widget，再在 `src/settings/widgetSetting/**` 注册。Widget 文件命名使用大驼峰，业务 Widget 以 `Widget.vue` 结尾。

### Widget 机制规则

Widget 是 xframelib 的运行时组合单元，不只是普通 Vue 组件。机制依赖 `import()` 异步加载、配置驱动注册、菜单递归关联、权限过滤、事件调度加载卸载，以及可选的依赖顺序控制。

- 可复用业务单元、GIS 引擎/工具、浮动面板、菜单、弹框容器、需要动态加载卸载的功能模块优先做成 Widget。
- 保持完整链路一致：组件放 `src/widgets/**`，注册放 `src/settings/widgetSetting/**`，菜单触发放 `src/settings/widgetMenuSetting/**`，权限点放 `src/settings/functionSetting.ts`，注册元数据通过 `/help/register` 或 `public/MenuRoutes.json` 生成/更新。
- Widget 配置里使用原生动态 `import()`，不要在 settings 文件里静态导入重型 Widget。
- 依赖顺序用 `afterid`、`SysEvents.WidgetLoadedEvent` 或 Page/Layout 编排表达，不要用定时器猜父 Widget、地图容器或代理 Widget 已加载。
- Widget 菜单可递归；Widget 菜单项的 `path` 指向 Widget `id`，路由菜单项的 `path` 指向路由路径，不要混用。
- 排查“Widget 不见了”时同时看三条过滤链：路由权限、Widget 菜单权限、Widget 配置权限；免登录布局还要检查 `src/permission/index.ts` 的 `layoutIDwhiteList`。
- `id` 要在 settings、菜单 `path`、load/unload、`afterid`、生成元数据和外部注册系统中保持稳定一致。
- `preload: true` 只给随布局必须存在的壳层 Widget，例如头部、底部、侧边菜单、`ModalContainerWidget`、常驻地图容器；普通业务面板和重型 GIS 工具保持懒加载。
- 新项目或 fork 要同步确认 `package.json` 的 `name` 和 `version`，它们常作为系统注册元数据的产品身份。

注册时使用项目已有模式：

```ts
import type { IWidgetConfig } from "xframelib";
import { LayoutContainerEnum } from "xframelib";

const widgets: Array<IWidgetConfig> = [
  {
    id: "MyBusinessWidget",
    label: "业务面板",
    container: LayoutContainerEnum.centerFront,
    component: () => import("@/widgets/demo/MyBusinessWidget.vue")
  }
];

export default widgets;
```

注意：

- `id` 必须稳定且唯一；菜单、页面加载和卸载都依赖它。
- `layoutID` 要与目标 `LayoutContainer` 匹配；权限过滤、菜单和 `Global.LayoutMap` 都依赖布局维度。
- 有加载顺序依赖时优先使用 `afterid` 或等待 `SysEvents.WidgetLoadedEvent`，不要靠任意延时。
- 需要给 Widget 传入外部参数时，优先检查当前版本是否支持 `componentProps`，并沿用项目已有配置写法。
- Widget 配置可能包含 `layout`、`cssClass`、`componentProps` 等扩展能力；改配置前先查当前项目的 `IWidgetConfig` 实际字段。
- 运行时可用 `Global.getLayoutManager(widgetID)` 按 Widget id 反查所属 `LayoutManager`，适合 Widget 内部自卸载或显隐控制。
- Widget 若要支持隐藏/打开而不是销毁/重建，应通过 `defineExpose` 暴露 `isShow` 和 `changeVisible`，并沿用已有 `changeWidgetVisible`/`getWidgetComponent` 模式。
- Widget 内如果再挂载子 `LayoutContainer`，要用独立 `layoutID` 管理子容器，并监听 `SysEvents.LayoutContainerLoaded` 获取子 `LayoutManager`。
- 地图底图/地图实例类 Widget 放 `centerBack`。
- 业务面板优先左右侧栏或 `centerFront`，全屏壳必须穿透事件。
- 不额外包无意义的全屏 `task-widget` 外壳。
- Widget 自己加载其他 Widget 时，也要在关闭或卸载路径中清理。

## 地图与 GIS 规则

地图相关功能要特别重视初始化顺序和卸载清理。

- Cesium、OpenLayers、MapLibre 容器类 Widget 放后景层。
- 依赖 `Global.CesiumViewer`、`Global.Mars3dMap`、`Global.MapLayoutManager` 等全局对象前，处理尚未初始化的状态。
- 地图事件、postRender/camera.changed、绘制工具、图层、实体、定时器必须在 `onUnmounted` 或 Widget 关闭逻辑中清理。
- 页面切换时卸载互斥地图 Widget，避免 2D/3D 容器和代理 Widget 残留。

初始化等待示例：

```ts
if (!Global.CesiumViewer) {
  OnEventHandler(SystemsEvent.CesiumWidgetLoaded, init);
  return;
}

init();
```

卸载时必须成对 `OffEventHandler`，避免重复初始化和内存泄漏。

## 事件总线和生命周期清理

事件统一从 `@/events` 使用：

```ts
import { OffEventHandler, OnEventHandler } from "@/events";

function handler(data: unknown) {
  // ...
}

onMounted(() => {
  OnEventHandler(WidgetsEvent.WidgetClosed, handler);
});

onUnmounted(() => {
  OffEventHandler(WidgetsEvent.WidgetClosed, handler);
});
```

清理清单：

- `OnEventHandler` 对应 `OffEventHandler`。
- `window/document/addEventListener` 对应 `removeEventListener`。
- `setInterval` 对应 `clearInterval`。
- 长时间 `setTimeout` 在卸载时按需 `clearTimeout`。
- ECharts、Monaco、ViewerJS、Cesium handler、OpenLayers interaction/source/layer 等实例要 dispose/destroy/remove。
- Vue `watch` 若不在当前 effect scope 自动释放，保存 stop handle 并在卸载调用。
- 一次性卸载多个 Widget 时优先用项目已有 `layoutManager.unloadWidgets(ids)`；离开布局或销毁容器时检查是否需要 `unloadAllWidgets()`。
- Widget 需要知道自身 `id/layoutID` 时，先沿用现有 runtime 注入方式；必要时再用 `getCurrentInstance()?.proxy?.$options` 读取，不要硬编码多份 id。

## Quasar/Vue/TypeScript 约定

- 使用 Vue 3 `<script setup lang="ts">` 和 Composition API。
- 所有新增或修改的源码、Markdown、JSON、SCSS 配置文件保持 UTF-8 编码，避免中文菜单、日志、注释、文档和 `SysConfig.js` 配置出现乱码。
- Pinia Store 放 `src/stores/modules/**`，命名优先以 `Store` 结尾。
- 表格列、菜单项、空数组 ref 要显式标注类型，避免 `never[]` 推断。
- 可选配置值在使用前收窄，不用非空断言掩盖真实缺失。
- UI 控件优先复用 Quasar 组件和项目已有 `src/components/Quasar/**`、菜单、表格部件。
- 不把业务调试信息长期留成裸 `console.log`；需要时使用统一 logger 或开发环境保护。

## 响应式自适应样式规则

Quasar 官方统称为响应式设计（Responsive Design）和断点（Breakpoints）。默认断点按 Quasar v2 记忆：`xs < 600px`、`sm >= 600px`、`md >= 1024px`、`lg >= 1440px`、`xl >= 1920px`。结构级响应式优先用 Quasar Flex Grid/Responsive classes：`row`、`col-12`、`col-sm-6`、`col-md-4`、`offset-lg-*`；显隐优先用 Visibility classes：`xs`、`sm`、`md`、`lg`、`xl`、`lt-md`、`gt-sm`；行为和组件参数再用 Screen Plugin：`$q.screen.lt.md`、`$q.screen.gt.sm`、`Screen.name`。不要把 Tailwind/UnoCSS 的 `md:text-h4` 或 Bootstrap 的 576/768/992/1200 阈值误当成 Quasar 默认规则。

项目自定义 `v-media` 只用于同一元素按断点切换多个样式 class，如标题、正文、装饰类。`v-media` 语法是 `"类名.断点条件"`，断点条件直接映射 Quasar `Screen`，例如 `v-media="'text-h2.gt.lg text-h3.lg text-h4.md text-h5.sm text-h6.xs'"` 或 `v-media="'text-body2.lt.md text-body1.gt.md'"`；使用前先确认 `src/directives/index.ts` 已注册 `setupmediaDirective(app)`。结构切换、组件显隐和参数优先用 `$q.screen`，例如 `v-if="$q.screen.gt.sm"` 隐藏桌面按钮、`v-if="$q.screen.lt.sm"` 切换移动端 `q-drawer`、`:dense="$q.screen.lt.md"` 压缩表格。地图或全屏 Widget 做响应式布局时，仍要遵守中心层 `pointer-events` 穿透规则，不要让移动端抽屉、透明壳或自适应外层挡住 GIS 拖拽缩放。

## 工程化与验证边界

当前项目存在历史技术债：

- `pnpm typecheck` 能启动，但仍有大量既有类型错误，首批集中在生成 API、旧 GIS API、未使用变量和 `never[]` 推断。
- `pnpm lint:check` 可能因全仓历史格式问题失败。若只改少量文件，可先用 `node_modules/.bin/oxfmt.CMD --check <files>` 验证本次改动文件。
- `quasar.config.ts` 对生产构建、分包和 worker 有项目特定配置；改动前先理解 `manualChunks`、`publicPath: "./"`、`cssCodeSplit: false` 的部署影响。
- `pnpm clean` 会删除 lockfile 和依赖缓存相关文件，除非任务明确要求，避免随意执行。

验证建议：

```powershell
pnpm.cmd typecheck
pnpm.cmd lint:check
pnpm.cmd build
node_modules\.bin\oxfmt.CMD --check <changed-files>
```

若 PowerShell 禁止 `pnpm.ps1`，改用 `pnpm.cmd`。若依赖安装需要联网或重建 `node_modules`，按 Codex 权限流程请求批准。

## 新增功能检查表

- Page/Widget 文件位置符合布局和职责。
- 文本文件读写保持 UTF-8，中文内容没有被 PowerShell 默认编码写坏。
- Widget 已在对应 `src/settings/widgetSetting/**` 注册，`id` 唯一稳定。
- 菜单触发、Modal、功能权限按需更新对应 settings 文件。
- Widget 菜单 `path`、`afterid`、Page load/unload、权限配置和生成元数据引用同一组稳定 Widget id。
- 全屏 Page/Widget 已处理 `pointer-events` 穿透。
- 需要保留实例的 Widget 已正确实现 `isShow`/`changeVisible`，不把显隐误做成重复加载。
- 地图/GIS 资源、事件、定时器、图层、实体在卸载时清理。
- 没有手改生成代码、public worker 或敏感运行时配置。
- 本次改动文件已格式化；能跑的检查已跑，并记录剩余既有失败原因。
