---
name: vue-widget-quasar-xframelib
description: Use this skill when working in the vue-widget-quasar-clivite-template repository or a similar Vue 3 + Quasar + Vite + xframelib Widget micro-frontend project. Trigger for adding or modifying pages, widgets, layout containers, widget settings, widget menus, modal settings, route modules, Pinia stores, global events, Cesium/OpenLayers/MapLibre GIS widgets, Quasar/Vite build issues, UTF-8 encoded text files, or code review in this architecture, even if the user only says "add a feature", "fix this widget", "register a page", "map widget", or "xframelib".
---

# Vue Widget Quasar xframelib Skill

Use this skill to modify or review this project without losing the thread of its Widget architecture. The main risk in this repository is not writing Vue code; it is forgetting the registration chain, placing a Widget in the wrong layout container, or leaving map/event resources alive after route changes.

## First Steps

1. Read the repository `AGENTS.md` and respect any local delegation, download mirror, or CodeGraph rules.
2. If `.codegraph/` exists at the repo root, use CodeGraph before `rg`/file reads for code-location questions. If it does not exist, skip CodeGraph.
3. Inspect the current implementation before editing. Prefer existing project patterns over generic Vue or Quasar examples.
4. Check `git status --short` before edits and do not revert unrelated user changes.
5. Keep all edited `*.md`, `*.vue`, `*.ts`, `*.js`, `*.json`, `*.scss`, and config text files encoded as UTF-8. In Windows PowerShell, read Chinese source notes with `Get-Content -Encoding UTF8` and write generated text with an explicit UTF-8 encoding.
6. On Windows PowerShell, prefer `pnpm.cmd` over `pnpm` when running scripts.

## Project Shape

Core stack:

- Vue 3 with `<script setup lang="ts">` and Composition API.
- Quasar Framework using `@quasar/app-vite`.
- Vite with project-specific plugins: `vite-plugin-xframelib`, `vite-plugin-xgis-cesium`, `vite-plugin-wasm`, `vite-plugin-commonjs`, `vite-plugin-node-polyfills`, and development-only helpers.
- xframelib provides `Global`, `LayoutContainer`, `LayoutManager`, Widget config types, menu/modal types, event bus, storage helpers, and layout container enums.
- Runtime configuration is loaded from `public/SysConfig.js` into `Global.Config`; service URLs, map keys, theme flags, lock-screen time, and log switches are environment-sensitive.
- Use `Global.Message` for user-facing feedback and `Global.Logger()` for diagnostics instead of leaving permanent `console.log`.
- GIS surface includes Cesium/xgis-cesium, OpenLayers/xgis-ol, and MapLibre.

Important directories:

- `src/layouts/**`: route-level layout shells. Layouts mount `LayoutContainer` and receive `containerLoaded`.
- `src/pages/**`: route pages. Pages often orchestrate which Widgets should load for the current route.
- `src/widgets/**`: Widget micro-frontend units. Use these for reusable business features, GIS controls, map containers, and floating panels.
- `src/settings/widgetSetting/*.ts`: Widget registration arrays. Aggregated by `src/settings/widgetSetting/index.ts` via `import.meta.glob("./*.ts")`.
- `src/settings/widgetMenuSetting/*.ts`: Widget menu definitions. Aggregated by `src/settings/widgetMenuSetting/index.ts`.
- `src/settings/modalSetting/*.ts`: Modal component registration. Aggregated by `src/settings/modalSetting/index.ts`.
- `src/settings/functionSetting.ts`: role/function permission points when a feature needs access control.
- `src/settings/projectSetting.ts`: product/system metadata used by registration/help flows.
- `src/router/<layout>/modules/*.ts`: modular child routes for a layout. Most `modules/index.ts` files aggregate siblings with `recursiveRoutes(import.meta.glob("./*.ts"))`.
- `src/permission/index.ts`: filters routes, Widget menus, and Widget config by function/role settings; also writes `Global.RightWidgetConfigMap`.
- `src/events/modules/*.ts`: string event constants grouped by domain.
- `src/stores/modules/*.ts`: Pinia modules, re-exported from `src/stores/index.ts`.
- `src/workers/*.service.ts`: async worker-service entry files that may be compiled into WebWorker output by the Vite worker plugin.
- `src/api/**` and `src/service/**`: generated API/RPC code. Avoid hand-editing unless the task explicitly targets generated artifacts.
- `public/SysConfig.js`: runtime global config. Treat as environment/sensitive config and change only with clear intent.
- `public/assets/**`, `.quasar/**`, `dist/**`, `node_modules/**`: generated or vendor output. Do not use as business edit targets.

## Decision Rules

Choose the edit surface by responsibility:

- New route screen: add a Page under `src/pages/<layout>/...` and a route module under `src/router/<layout>/modules/...`.
- Reusable or dynamically loaded business UI: add a Widget under `src/widgets/<domain>/...` and register it in `src/settings/widgetSetting/*.ts`.
- Menu item that opens/toggles a Widget: update `src/settings/widgetMenuSetting/*.ts`; `path` should match the Widget `id`.
- Modal component: add/register in `src/settings/modalSetting/*.ts`; open it through the `ModalContainerWidget` event flow rather than direct component ownership.
- Permission-gated feature: update `src/settings/functionSetting.ts` and any related route/menu metadata.
- Product metadata or registration/help output: inspect `projectSetting.ts`, the route/menu/widget settings, and any `MenuRoutes.json` generation behavior before changing generated metadata.
- Cross-Widget transient notification: use the event bus through `@/events`.
- Shared durable state: use a Pinia store under `src/stores/modules/**`.
- GIS map instance, basemap, layer, entity, or renderer container: prefer Widget form and pay extra attention to initialization order and cleanup.

## Widget Mechanism Rules

Treat Widget as the main reuse and runtime composition unit, not just a Vue component folder. The template's Widget mechanism depends on asynchronous `import()`, config-driven registration, menu recursion, permission filtering, event-driven load/unload, and optional dependency ordering.

Core rules:

- Prefer Widget form for reusable business units, GIS engines/tools, floating panels, menus, modal containers, and feature modules that must be dynamically loaded or unloaded.
- Keep the full chain consistent: Widget component under `src/widgets/**`, Widget config under `src/settings/widgetSetting/*.ts`, optional trigger menu under `src/settings/widgetMenuSetting/*.ts`, optional permission point under `src/settings/functionSetting.ts`, and optional metadata/regeneration through `/help/register` or `public/MenuRoutes.json`.
- Use native dynamic imports in Widget config; do not eagerly import heavy Widget components in settings files.
- Model dependencies with `afterid` or explicit page/layout orchestration. Do not use timers to guess that a parent Widget, map container, or proxy Widget has loaded.
- Remember Widget menus can be recursive. For Widget menu entries, `path` points to a Widget `id`; for route entries, `path` points to a router path. Keep those meanings separate.
- Understand permission flow before debugging a "missing Widget": route permissions, Widget menu permissions, and Widget config permissions are filtered separately in `src/permission/index.ts`.
- For anonymous/white-listed layouts such as portal, product, or big-screen pages, update `layoutIDwhiteList` when needed; otherwise the route can render while `getRightWidgetConfig()` returns no Widget configs for that layout.
- Keep Widget ids stable across settings, menu paths, load/unload calls, `afterid`, generated `MenuRoutes.json`, and any external registration system.
- Use `preload: true` only for shell Widgets that must exist with the layout, such as headers, footers, side menus, `ModalContainerWidget`, and always-on map containers. Keep business panels and heavy GIS tools lazy.
- When creating a new project/fork, align `package.json` `name` and `version` because the registration metadata uses them as the product identity.

## Layout and Widget Registration

Widget settings are flat files under `src/settings/widgetSetting/*.ts`; they default-export `Array<IWidgetConfig>`. The project index gathers these with `globFilterLayoutWidgetConfig`, so do not create nested settings directories unless you also update the aggregator.

Typical registration:

```ts
import type { IWidgetConfig } from "xframelib";
import { LayoutContainerEnum } from "xframelib";

const widgets: Array<IWidgetConfig> = [
  {
    layoutID: "bigScreenLayout",
    id: "MyBusinessWidget",
    label: "Business Panel",
    container: LayoutContainerEnum.centerFront,
    component: () => import("@/widgets/demo/MyBusinessWidget.vue"),
    preload: false
  }
];

export default widgets;
```

Registration rules:

- Keep Widget `id` stable and unique. Pages, menus, load/unload calls, and `afterid` depend on it.
- Use `layoutID` that matches the target `LayoutContainer`, such as `backLayout`, `bigScreenLayout`, `productLayout`, `portalLayout`, or `mapLayout`.
- Use `afterid` when a Widget depends on a map/proxy/container Widget being loaded first.
- Use `preload: true` only for layout chrome or always-needed controls. Keep heavier panels and GIS tools lazy when possible.
- Menu `path` values in `IWidgetMenu` should reference Widget `id` values, not component paths.
- For `MenuItemEnum.Widget`, `path` is a Widget `id`. For route menu items, `path` is a route path. Do not blur those two meanings.
- `MenuBarWidget` is designed to reuse a group of Widget menu entries across layouts. Style different menu bars by their configured `id` and shared menu-bar styles rather than copying a bespoke menu Widget for every group.
- If a Widget must support hide/show without unmounting, expose `isShow` and `changeVisible` with `defineExpose`, and use the existing `changeWidgetVisible`/component lookup pattern.
- Use `componentProps` when the current project version supports passing external props into a Widget. Remember that runtime may inject `id` and `layoutID` into Widget options.
- For XWindow-style Widgets, close/minimize behavior should still route through the owning `LayoutManager` with `unloadWidget()` or `changeWidgetVisible()`; do not remove DOM nodes directly.

## Container Selection

Respect container responsibility instead of solving layering problems with random `z-index` values:

- `LayoutContainerEnum.centerBack`: map engines, basemaps, render surfaces, background content, map proxy/container Widgets.
- `LayoutContainerEnum.centerFront`: floating toolbars, query panels, analysis panels, overlays that sit above maps.
- `LayoutContainerEnum.left` / `right`: side business panels, layer trees, properties, details, task lists.
- `LayoutContainerEnum.top` / `bottom`: global headers, status bars, fixed tool areas.

For full-screen overlays above GIS content, preserve event pass-through on the large shell and re-enable events only on real controls:

```vue
<template>
  <div class="my-widget">
    <aside class="panel">...</aside>
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

## Page Orchestration

Pages can load and unload Widgets for their route. Use `Global.LayoutMap.get(layoutID)` when the layout is ready; otherwise listen for `SysEvents.LayoutContainerLoaded` and remove the listener after initialization.

```ts
import { OffEventHandler, OnEventHandler } from "@/events";
import { onMounted, onUnmounted } from "vue";
import { Global, LayoutManager, SysEvents } from "xframelib";

let layoutManager: LayoutManager | undefined;

function layoutLoaded(data: { layoutID?: string }) {
  if (data?.layoutID === "bigScreenLayout") init();
}

function init() {
  layoutManager = Global.LayoutMap.get("bigScreenLayout");
  if (!layoutManager) {
    OnEventHandler(SysEvents.LayoutContainerLoaded, layoutLoaded);
    return;
  }

  layoutManager.loadWidget("MyBusinessWidget");
  OffEventHandler(SysEvents.LayoutContainerLoaded, layoutLoaded);
}

onMounted(init);

onUnmounted(() => {
  OffEventHandler(SysEvents.LayoutContainerLoaded, layoutLoaded);
  layoutManager?.unloadWidget("MyBusinessWidget");
});
```

When changing a map route, unload mutually exclusive 2D/3D map containers and proxy Widgets that the previous route loaded.

When editing a `LayoutContainer` shell:

- Pass filtered Widget config with `getRightWidgetConfig()` unless the local layout already provides a narrower config.
- Use `@containerLoaded` to capture the `LayoutManager` for that layout; this event also backs `SysEvents.LayoutContainerLoaded`.
- Preserve named slots `main`, `back`, `front`, `left`, `right`, and `bottom` when the layout uses slot content.
- Call `Global.Loading("end")` only where the layout/page is responsible for ending the startup loading state.

## Routes

Layout routes live in `src/router/<layout>/index.ts` and child routes live in `src/router/<layout>/modules/*.ts`.

Pattern:

```ts
import type { RouteRecordRaw } from "vue-router";

const routeName = "feature";

const routes: Array<RouteRecordRaw> = [
  {
    path: "feature",
    name: routeName,
    meta: {
      index: 10,
      title: "Feature",
      icon: "material-icons-outlined:extension"
    },
    component: () => import("@/pages/back/Feature/index.vue")
  }
];

export default routes;
```

If adding a new top-level layout, also register it in `src/router/index.ts` and provide matching Widget settings for its `layoutID`.

For normal page additions, add or update a module file under `src/router/<layout>/modules/*.ts`; do not manually import every route in the layout root unless you are changing the route aggregation mechanism itself.

## Modals

Modal settings are flat files under `src/settings/modalSetting/*.ts`; they default-export `Array<IModalConfig>`. Runtime loading is handled by `src/components/ModalContainer/index.ts`, while `src/widgets/layouts/ModalContainerWidget.vue` listens for `WidgetsEvent.ModalContainerWidget_LoadModal`.

Typical registration:

```ts
import type { IModalConfig } from "xframelib";

const modals: Array<IModalConfig> = [
  {
    id: "MyEditModal",
    label: "Edit",
    component: () => import("@/pages/back/MyFeature/modals/MyEditModal.vue")
  }
];

export default modals;
```

Typical open flow:

```ts
import { EmitMsg } from "@/events";
import WidgetsEvent from "@/events/modules/WidgetsEvent";

EmitMsg(WidgetsEvent.ModalContainerWidget_LoadModal, {
  modalID: "MyEditModal",
  rowData,
  extraData: { mode: "edit" },
  width: 720
});
```

Ensure the layout has loaded `ModalContainerWidget` before relying on this flow.

Modal component rules:

- Keep the modal component `name` aligned with the `modalSetting` id/name convention used nearby.
- Dynamic modal content should accept the established `data` and `extra` props when it needs row payloads or mode/options.
- Preload or otherwise load `ModalContainerWidget` in the relevant layout before emitting modal-open events.
- Modal content can listen for its own name/id event for OK/Cancel handling; always unregister that listener in `onUnmounted`.
- Use the project modal open shape consistently: `modalID`, `rowData`, `extraData`, and optional `width`. Keep `rowData` for business data and `extraData` for title, mode, footer, and display options.

## Events and Cleanup

Use the wrappers in `@/events`:

```ts
import { EmitMsg, OffEventHandler, OnEventHandler } from "@/events";
import WidgetsEvent from "@/events/modules/WidgetsEvent";

function handler(data: unknown) {
  // ...
}

OnEventHandler(WidgetsEvent.WidgetLoaded, handler);
EmitMsg(WidgetsEvent.WidgetLoaded, { id: "MyBusinessWidget" });
OffEventHandler(WidgetsEvent.WidgetLoaded, handler);
```

Cleanup checklist for every Page and Widget:

- `OnEventHandler` has a matching `OffEventHandler`.
- `window`/`document.addEventListener` has a matching `removeEventListener`.
- `setInterval` and long-lived `setTimeout` are cleared.
- Vue `watch` stop handles are called if they are not automatically scoped.
- ECharts, Monaco, ViewerJS, Cesium handlers, OpenLayers interactions/layers/sources, MapLibre events/layers/sources, WebGL renderers, and custom workers are disposed or removed.
- Widget close/unmount paths unload any child Widgets loaded by the Widget itself.
- App-level Widget closing is centralized through `WidgetsEvent.WidgetClosed`; avoid scattering direct DOM removal or ad hoc close logic through business code.

For high-frequency map or resize events, use throttling/debouncing rather than emitting on every frame.

## GIS Rules

GIS work has stricter ordering:

- Put map/render containers in `centerBack`.
- Put interactive panels and toolbars in `centerFront`, `left`, or `right`.
- Do not assume `Global.CesiumViewer`, `Global.Mars3dMap`, `Global.MapLayoutManager`, or map proxy instances exist synchronously. Wait for the relevant loaded event or layout loaded event.
- When a Widget depends on a map container, set `afterid` or load it after the map Widget resolves.
- On unmount, remove map events, draw interactions, entities, layers, sources, post-render callbacks, and custom animation loops.
- Do not allow transparent full-screen shells to block map drag, zoom, timeline, or context menu interactions.

## State

Use Pinia for shared, durable, or cross-route state. Add modules under `src/stores/modules/**` and export them from `src/stores/index.ts`.

Project patterns:

- Use `defineStore` from `pinia`.
- Use `xframelib` `Storage` where the existing module does.
- Persist through actions such as `saveCacheStore()` rather than mutating nested state and forgetting persistence.
- Keep temporary component-only UI state local with `ref`/`reactive`.
- Avoid circular Store imports; access another store inside an action if needed.

## Built-In UI Helpers

Before building custom Quasar UI, check for reusable project components:

- `src/components/Quasar/QLayoutMainContainer/index.vue`: bridge `LayoutContainer` with Quasar `QLayout`/`QPage`/`QFooter` layouts.
- `src/components/Menu/SideMenuBar/index.vue`: floating left/right route menus, commonly inside `q-drawer`; accepts route/menu children.
- `src/components/Menu/ContextMenu.vue`: common right-click menu; `menuList` uses `IContextMenuItem[]`, supports up to two menu levels, and `{}` can represent a separator in existing examples.
- `src/components/Quasar/**` and `src/components/QuasarExtensions/**`: existing Quasar wrappers such as base content, Markdown viewer/TOC, PDF/image viewers, FlashCard, and related extensions.
- `MenuBarWidget`: reusable Widget menu bar for horizontal or vertical groups, often used on big-screen pages with `widgetMenuSetting`; use one reusable menu-bar Widget across layouts and specialize appearance by configured `id` and `menuBarStyle.scss` rather than copying one menu Widget per group.

## Permissions

When a route, Widget, or Widget menu should be controlled by permissions:

- Add the function point to `src/settings/functionSetting.ts`.
- Add matching metadata/config expected by xframelib permission helpers.
- Re-check `src/permission/index.ts`, because `getRightRoutes()`, `getRightWidgetMenus()`, and `getRightWidgetConfig()` are separate filtering paths.
- Remember the whitelist behavior for unauthenticated or low-role states: `portalLayout`, `bigScreenLayout`, and `productLayout` Widget configs may still be allowed by `getWhiteListWidgetConfig()`.
- If the project uses `layoutIDwhiteList`, add new anonymous/white-listed layouts there; otherwise the route may work while its `widgetConfig` is filtered out.

## Quasar and TypeScript

- Prefer `<script setup lang="ts">` for new Vue files unless neighboring code clearly uses `defineComponent`.
- Reuse Quasar components and existing local components before inventing new UI primitives.
- Preserve UTF-8 encoding for Chinese menu labels, route titles, Widget labels, logs, docs, and comments.
- Explicitly type empty arrays in refs, table columns, menu items, and config arrays to avoid `never[]`.
- Narrow optional config before use; do not use non-null assertions to hide real missing config.
- Do not leave permanent business `console.log`; production config drops console output, so use project logging or guarded diagnostics when needed.
- Keep file naming consistent: components and Widgets use PascalCase; business Widgets usually end with `Widget.vue`.

## Quasar/Vite Notes

- `@quasar/app-vite` drives scripts and build modes; inspect the current `package.json` and `quasar.config.ts` before changing CLI, Vite plugins, aliases, or build targets.
- New project forks should update `package.json` `name` and `version` together when they are used as the system's unique product identity.
- User-defined environment variables should use the `VITE_` prefix and uppercase names; do not assume runtime `SysConfig.js` values and build-time `.env` values are interchangeable.
- Keep path aliases centralized in `tsconfig.json` and enabled through the project's Vite/Quasar config pattern. If import warnings appear, check `tsconfig.json` `include`, such as `src/**/*`.
- For subdirectory deployment, preserve `publicPath: "./"` and relative asset references unless the deployment target is intentionally changing.
- For Cesium and other GIS vendor assets, do not delete required public assets such as Cesium `Assets`; when upgrading Cesium, align package versions with `public/Cesium` resources.
- For top-level-await dependency errors, inspect `quasar.config.ts` `viteConf.esbuild.supported` and related target settings before patching libraries.
- Worker-service files belong under `src/workers/*.service.ts`; prefer the existing worker-service plugin pattern over ad hoc workerpool code when external imports are needed inside workers.

## Build and Verification

Scripts from current `package.json`:

```powershell
pnpm.cmd dev
pnpm.cmd typecheck
pnpm.cmd lint:check
pnpm.cmd lint:fix
pnpm.cmd format
pnpm.cmd test
pnpm.cmd build
pnpm.cmd build:pwa
pnpm.cmd op-images
```

Verification guidance:

- For narrow edits, run the smallest relevant check first, such as `node_modules\.bin\oxfmt.CMD --check <changed-files>` plus targeted tests if available.
- For TypeScript or shared contract edits, run `pnpm.cmd typecheck`.
- For routing/settings/build config changes, run `pnpm.cmd build` when feasible.
- `pnpm.cmd lint:check` and `pnpm.cmd typecheck` may expose pre-existing repository issues. Report whether failures are related to the current change.
- Avoid `pnpm.cmd clean` unless explicitly requested; it removes lockfile/dependency/cache artifacts.
- If dependencies or tools must be downloaded, use China-accessible mirrors when possible and follow the environment approval flow.

## Do Not Hand Edit

Avoid modifying these unless the user explicitly asks and the intent is clear:

- `src/api/**`
- `src/service/**`
- `public/assets/**`
- `public/SysConfig.js` without confirming runtime config impact
- `.quasar/**`
- `dist/**`
- `node_modules/**`
- generated lock/cache files except as part of an intentional dependency operation

## Feature Checklist

Before finishing a feature or fix:

- The file location matches the responsibility: Page, Widget, component, Store, setting, or route.
- Text files remain UTF-8 encoded; Chinese labels and docs render correctly after edits.
- Widget registration exists in `src/settings/widgetSetting/*.ts` with the right `layoutID`, `id`, `container`, `component`, and `preload`.
- Menu, modal, and function permission settings are updated when needed.
- Widget menu `path` values, `afterid`, page load/unload calls, generated metadata, and permission configs reference the same stable Widget ids.
- Route modules are registered under the correct layout and use dynamic imports.
- Full-screen Page/Widget shells preserve GIS pointer-event pass-through when applicable.
- Event listeners, timers, map resources, chart/editor instances, workers, and child Widgets are cleaned up.
- No generated/vendor/runtime-sensitive files were changed casually.
- Formatting and feasible checks were run, or the reason they were not run is documented.
