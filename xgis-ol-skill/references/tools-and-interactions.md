# 工具、交互与控件

`xgis-ol` 的 `controls` 命名空间导出：`BasicTool`、`DrawTool` + `EnumDrawType`、
`MeasureTool`、`RollSwipe` + `EnumSwipeType`、`PrintTool`。这些是**编程式**工具类；
若要现成 UI 工具条，用 components.md 里的 Vue 组件（`DrawToolBar` 等）。

## BasicTool

```ts
import { BasicTool } from 'xgis-ol';
const t = new BasicTool(xmap);
t.zoomin();
t.zoomout();
```

## DrawTool（矢量绘制）

```ts
import { DrawTool, EnumDrawType } from 'xgis-ol';
const draw = new DrawTool(xmap);
draw.setDrawLayer(0);                              // 指定绘制图层（索引或 VectorLayer）
draw.changeDrawType(EnumDrawType.Polygon);        // 切换绘制类型，可传第二参 ol Style
// draw.DrawSource → 绘制结果 VectorSource
draw.clearLastDraw();                              // 撤销上一次
draw.clearboard();                                 // 清空
draw.clearInteraction();                           // 结束交互
```

`EnumDrawType`：`Point | LineString | LinearRing | Polygon | MultiPoint |
MultiLineString | MultiPolygon | GeometryCollection | Circle | Hand`。

## MeasureTool（测量）

```ts
import { MeasureTool } from 'xgis-ol';
const measure = new MeasureTool(xmap.map);         // 注意：传 ol/Map，不是 XMap
measure.addInteraction('LineString');              // 或 'Polygon'（长度/面积）
measure.removeInteraction();
```

内部处理提示 tooltip、格式化长度/面积、临时禁用双击缩放。

## RollSwipe（卷帘 / 鱼眼）

`xmap.RollSwipe`（`get RollSwipe(): RollSwipe`）或 `new RollSwipe(container, xmap)`。

```ts
import { EnumSwipeType } from 'xgis-ol';
const swipe = xmap.RollSwipe;
swipe.setSwipeLayer(1);                            // 卷帘图层（索引或 Layer）
swipe.changeSwipeType(EnumSwipeType.RollHorizonl); // 水平/垂直/鱼眼
swipe.load();
// swipe.unload();
```

`EnumSwipeType`：`RollHorizonl | RollVertical | FishEye`。也可结合
`xmap.LayerManager.changeSwipeLayerByID(id)` 指定卷帘图层。

## PrintTool（打印，基于 ol-ext）

`xmap.PrintTool`（`get PrintTool(): PrintTool`）。

```ts
xmap.PrintTool.addPrintControl();          // 简单打印控件
xmap.PrintTool.addPrintDialogControl(opts);// 打印对话框（ol-ext PrintDialog）
// xmap.PrintTool.removePrintControl();
// xmap.PrintTool.removePrintDialogControl();
```

## 自定义交互/控件的注册

用 XMap 的扩展字典按 key 管理（避免重复添加、便于卸载）：

```ts
xmap.addInteractionExt('myClip', interaction);
if (xmap.containsInteractionExt('myClip')) { ... }
xmap.removeInteractionExt('myClip');

xmap.addControlExt('myOverview', control);
xmap.removeControlExt('myOverview');
```

内置 key：`xmap.DefaultControlKeys = { overview, target, scale, scaleline }`，
`xmap.DefaultInteractionKeys = { FishEyeClip }`。

## ol-ext 直接使用

`ol-ext` 是本项目依赖，很多高级能力（透视图、裁剪 mask、遮罩、特效控件）直接用 ol-ext 原生 API，
配合 `xmap.map`。例如项目里的透视地图 `MapExt` 继承自 `ol/Map` 并用 `ol-ext/util/matrix3D`、
`ol-ext/util/element`。需要裁剪/遮罩/打印对话框等时，优先查 ol-ext 对应控件
（`ol-ext/control/PrintDialog`、`ol-ext/filter/*`、`ol-ext/interaction/*`）。

## 事件总线（MapEventBus）

`import { MapEvent, MapEventBus } from 'xgis-ol'`；`xmap.mapEventBus` 是实例。

```ts
xmap.mapEventBus.eventOn(MapEvent.LAYER_VISIBLE_CHANGED, (eArgs) => { ... });
xmap.mapEventBus.eventEmit(MapEvent.MAP_MESSAGE_INFO, xmap.newEvtArgs({ msg: 'hi' }));
xmap.mapEventBus.eventOff(MapEvent.LAYER_VISIBLE_CHANGED, handler);
```

`MapEventArgs`：`{ mapID, mapGroup, data?, target? }`，用 `xmap.newEvtArgs(data?, eventObject?)` 创建。

`MapEvent` 常量（图层与地图消息）：
`LAYER_ORDER_CHANGED`、`LAYER_LOCATE`、`LAYER_OPACITY_CHANGED`、`LAYER_VISIBLE_CHANGED`、
`LAYER_ADDED`、`LAYER_REMOVED`、`LAYER_COPY`、`LAYER_ITEMS_CHANGED`、`LAYER_ITEM_UPDATED`、
`LAYER_SWIPE_CHANGED`、`LAYER_CHANGED`、`MAP_SYNC_VIEW`、`MAP_SYNC_VIEW_END`、
`MAP_ZOOM_CHANGED`、`MAP_MESSAGE_INFO`、`LAYER_TREE_CONTEXT_MENU`、
`PASTE_LAYER_LOAD_WMTS`、`PASTE_LAYER_LOAD_MVT`。

## 通用工具函数（utils）

`import { uuid, deepMerge, requestFullScreen, exitFullScreen, dispatchWindowResize,
createElement, SaveFile, getRandomNum } from 'xgis-ol'`。还导出 mitt 事件与类型判断工具。
