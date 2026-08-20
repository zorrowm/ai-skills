---
name: xgis-ol-skill
description: "基于 OpenLayers(ol) + ol-ext + xgis-ol + xgis-data2d + xgis-plot 开发二维 GIS 地图（Vue3 + Quasar + TypeScript）。适用于任务提到：ol/OpenLayers 二维地图、xgis-ol、XMap、initByMapConfig、IMapConfig、LayerManager 图层管理/图层树 LayerTree、PrjGridTool 投影与坐标转换、WMTSTool 天地图/WMTS、VTLayerTool 矢量切片 MVT、在线底图(天地图/高德/谷歌/Bing/Mapbox/百度/腾讯/Arcgis)、DrawTool/MeasureTool/RollSwipe 卷帘/PrintTool 打印、OLXMap/XMapView/ZoomFullBar/ContextMenu 等 Vue 组件、xgis-plot 态势标绘(PlotOL/PlotTypes 箭头/旗帜/集结地)、xgis-data2d 矢量数据导入导出(Shapefile/GeoJSON/TopoJSON 转换/投影变换)、ol-ext 透视图/裁剪/遮罩，或地图初始化、图层加载、绘制测量、右键菜单、多地图联动等二维 GIS 开发与集成问题。"
---

# xgis-ol 二维地图开发

本 skill 指导基于 **OpenLayers 生态 + xgis 系列封装库** 的二维 GIS 开发。技术栈：
Vue3 `<script setup>` + Quasar + TypeScript + Vite，配合 `xframelib`（`Global.XMap` 全局挂载）。

## 库分工

| 库 | 作用 |
|----|------|
| `ol` (OpenLayers) | 底层地图引擎：Map/View/Layer/Source/Feature/Style/Interaction |
| `ol-ext` | ol 扩展：透视图、裁剪 mask、遮罩、打印对话框、特效控件等 |
| `xgis-ol` | **核心封装**：`XMap` 门面类、图层管理、投影工具、WMTS/MVT、绘制测量卷帘打印、Vue 组件 |
| `xgis-plot` | 态势标绘：军事箭头/旗帜/集结地等（`PlotOL` + `PlotTypes`） |
| `xgis-data2d` | 矢量数据导入导出与格式转换（Shapefile/GeoJSON/TopoJSON/CSV、投影变换） |

分层原则：**`xgis-ol` 是入口**，能用 `XMap`/`xgis-ol` 封装就不要直接写裸 ol；
但矢量加载、GeoJSON、样式等 ol 原生能力仍直接用（通过 `xmap.map`）。data2d 只做数据转换，
渲染回到 ol。ol-ext 用于 xgis-ol 未覆盖的高级视觉能力。

## 工作流程

1. **先判断任务类型**，据此加载对应 reference（不要一次全读）：
   - 地图初始化、XMap API、视图/定位、生命周期 → `references/core-xmap.md`
   - 图层管理(LayerManager)、投影(PrjGridTool)、在线底图、WMTS、矢量切片 MVT、GeoJSON → `references/layers-and-projection.md`
   - 绘制/测量/卷帘/打印/自定义交互控件、事件总线、ol-ext → `references/tools-and-interactions.md`
   - Vue 组件（OLXMap/XMapView/LayerTree/工具条/ContextMenu） → `references/components.md`
   - 态势标绘 → `references/plot.md`
   - 矢量数据导入导出/格式转换 → `references/data2d.md`
2. **优先参考本地已安装库的类型声明**（`node_modules/xgis-ol/dist/**/*.d.ts`、
   `node_modules/xgis-plot/dist/**/*.d.ts`）和**现有项目代码**（`src/widgets/**`、`src/map/`），
   它们是最权威的 API 来源。API 随版本变化，拿不准时读 `.d.ts` 确认。
3. **仿照项目既有 widget 编写**：本项目地图功能都放在 `src/widgets/`（`olwidgets/`、`examples/`），
   新功能应遵循同样的 `<script setup>` + `Global.XMap` 模式。

## 来源优先级

1. 本地 `node_modules/{xgis-ol,xgis-plot,xgis-data2d}/dist/**/*.d.ts` 类型声明。
2. 现有项目代码：`src/widgets/**`、`src/map/MapExt.ts`、`src/models/`、`public/*.json` 配置。
3. 本 skill 的 references。
4. OpenLayers 官方文档 `https://openlayers.org/en/latest/apidoc/`、ol-ext `http://viglino.github.io/ol-ext/`。

## 三种地图初始化方式（速查，细节见 core-xmap.md / components.md）

1. **纯手动**：`new XMap('map', group)` → `initMapView({...})` → `xmap.map.addLayer(...)`。最小示例。
2. **配置驱动**：`XMap.initByMapConfig(mapConfig: IMapConfig)`，一次配置图层/控件/交互。推荐业务系统。
3. **Vue 组件**：`<OLXMap @mapInited="e => xmap = e.xmap">` 或 `<XMapView>`。最省事。

均需引入 `import 'xgis-ol/dist/index.css'`。

## 关键实现规则

- **样式必引**：`import 'xgis-ol/dist/index.css'`；用到标绘再引 `xgis-plot/dist/index.css`。
- **时机**：`XMap` 在 `onMounted` 之后创建（DOM 容器要存在）；容器需 `position:relative` 且有真实宽高，否则地图空白。
- **全局挂载**：项目约定 `Global.XMap = xmap`（来自 `xframelib`），其它 widget 用 `Global.XMap as XMap` 取用。
- **不要深响应式化地图对象**：用 `ref<XMap>()` 只存引用，或挂 `Global`；勿放进 `reactive`。
- **裸 ol 走 `xmap.map`**：加原生图层/交互用 `xmap.map.addLayer(layer as any)`；`xmap.MapView` 是视图。
- **手动图层要注册**：想在图层树 `<LayerTree>` 里管理显隐/透明度，调用 `xmap.registerLayer(layer, layerItem)`。
- **投影要对齐**：地图常为 `EPSG:3857` 或自定义投影；加载 4326 数据用 `featureProjection`，坐标用 `PrjGridTool` 转换。自定义投影用 `PrjGridTool.getProjection(projInfo)` 并放入 `viewOptions.projection`。
- **工具类的入参**：`DrawTool`/`RollSwipe`/`BasicTool`/`PrintTool` 等构造传 `XMap`；但 `MeasureTool` 和 `PlotOL` 传 `xmap.map`（ol/Map）。注意区分。
- **命名**：不要把变量/类命名为 `ol`、`XMap`、`PlotOL` 等库保留名。
- **清理**：`onUnmounted` 中 `xmap.dispose()`、移除自加图层与事件监听、`plotHelper.removeAllFeatures()` 等。

## 验证清单

- 初始化：容器 id 存在且有宽高；`onMounted` 后创建；样式已引入；控制台无 CORS/token 报错。
- 图层：服务 URL 能在浏览器 network 加载；投影/切片方案正确；`zoomToExtent`/`zoomToCenter` 能定位可见。
- 交互：绘制/测量/卷帘/标绘激活后能操作，切换/清除正常；自定义交互用 `addInteractionExt` 按 key 管理避免重复。
- 数据：data2d 导入产物投影与地图对齐；导出前 `writeFeatures` 并按需 `transformDataset`。
- 生命周期：卸载时地图、图层、事件、标绘、定时器均已释放。
