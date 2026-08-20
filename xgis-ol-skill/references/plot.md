# xgis-plot 态势标绘

`xgis-plot` 基于 OpenLayers 实现军事态势标绘（箭头、旗帜、集结地等）。
入口默认导出 `PlotOL`，并具名导出 `PlotTypes`、`PlotGeometry`。使用前引入样式。

```ts
import PlotOL, { PlotTypes } from 'xgis-plot';
import 'xgis-plot/dist/index.css';
```

## 初始化

`PlotOL` 接收底层 `ol/Map`（即 `xmap.map`）：

```ts
import { XMap } from 'xgis-ol';
import { Vector as VectorLayer } from 'ol/layer';

const xMap = Global.XMap as XMap;
const plotHelper = new PlotOL(xMap.map, { layerName: 'plotDrawLayer' });

// 标绘绘制图层（VectorLayer），可注册进图层管理以便在图层树中显隐
const thisDrawLayer: VectorLayer = plotHelper.plotDraw.drawLayer;
if (thisDrawLayer) {
  xMap.registerLayer(thisDrawLayer, {
    id: 'plotDrawLayer', name: 'plotDrawLayer', alias: '态势标绘',
    group: 'plotDrawLayer', minZoom: 1, maxZoom: 24,
    visible: true, opacity: 1, type: 'vector',
  });
}
```

构造第二参 options 里 `layerName` 指定图层名（类型定义写作 `{ zoomToExtent }`，实际用 `layerName`）。

## 绘制与编辑

`PlotOL` 内部聚合三个子对象：`plotDraw`（绘制）、`plotEdit`（编辑控制点/拖动）、`plotUtils`。

核心 API：
- `activate(type: string)` → 激活某种标绘类型开始绘制，`type` 取自 `PlotTypes`。
- `getCurrentFeature()` → 当前选中要素。
- `getCurrentFeaGeoJSON()` → 当前要素 GeoJSON。
- `getAllFeatureGeoJSONs()` → 图层所有要素 GeoJSON 数组（用于导出）。
- `removeFeature(fea?)` → 删除指定/当前要素。
- `removeAllFeatures()` → 清空。
- `addFeatures(features)` → 导入 GeoJSON 要素（用于加载已保存标绘）。
- 属性字段：`getAttributeFields()`、`getAttributeField(key)`、`setAttributeField(key, value)`。

```ts
plotHelper.activate(PlotTypes.ATTACK_ARROW);   // 开始画进攻方向箭头
plotHelper.removeFeature();                     // 删当前
plotHelper.removeAllFeatures();                 // 清空
const json = plotHelper.getAllFeatureGeoJSONs();// 导出
plotHelper.addFeatures(JSON.parse(text));       // 导入
```

## PlotTypes（标绘类型常量）

- 点线：`POINT`、`POLYLINE`、`CURVE`、`ARC`、`FREEHANDLINE`
- 面：`POLYGON`、`FREE_POLYGON`、`CLOSED_CURVE`、`RECTANGLE`、`CIRCLE`、`ELLIPSE`、
  `LUNE`（弓形）、`SECTOR`（扇形）、`GATHERING_PLACE`（集结地）
- 箭头：`STRAIGHT_ARROW`、`FINE_ARROW`、`DOUBLE_ARROW`、`ATTACK_ARROW`（进攻方向）、
  `TAILED_ATTACK_ARROW`、`ASSAULT_DIRECTION`（突击方向）、`SQUAD_COMBAT`（分队作战）、`TAILED_SQUAD_COMBAT`
- 旗帜：`RECTFLAG`、`TRIANGLEFLAG`、`CURVEFLAG`、`PENNANT`
- 文本：`TEXTAREA`

`PlotGeometry` 命名空间导出对应几何类（`AttackArrow`、`DoubleArrow`、`GatheringPlace` 等），
一般无需直接使用——通过 `activate(type)` 即可。

## 导入/导出模式（配合 xframelib）

```ts
import { Global, SaveAs, H5Tool } from 'xframelib';

// 导出
const json = plotHelper.getAllFeatureGeoJSONs();
SaveAs(JSON.stringify(json), 'plotResult.json');

// 导入（读文件 → 解析 → addFeatures）
const data = await H5Tool.readFilePromise(file, 'Text') as string;
plotHelper.addFeatures(JSON.parse(data));
```

## 事件

`xgis-plot` 也带 `events`（`PlotEvent`、`MapEventBus`、mitt），可监听绘制开始/结束等，
用法与 xgis-ol 的事件总线一致。

## 注意
- `PlotOL` 用 `xmap.map`（ol/Map），不是 `XMap`。
- 绘制图层建议 `registerLayer` 注册，才能在 `<LayerTree>` 中控制显隐/透明度。
- 标绘图标资源（`img/drawImg/*.png`）需自备并放到 public 目录，UI 面板据此渲染缩略图。
