# 图层、投影与瓦片服务

## LayerManager（图层管理）

`xmap.LayerManager` 管理所有注册图层，与图层树 `<LayerTree>` 联动。仅当
`hasLayerManager=true` 时存在。

图层描述用 `ILayerItem`：

```ts
interface ILayerItem {
  id?: string; name: string; alias?: string; group?: string; type?: string;
  minZoom?: number; maxZoom?: number; visible?: boolean; opacity?: number;
  children?: ILayerItem[];
}
```

注册手动创建的 ol 图层：

```ts
const layerItem = {
  id: 'plotDrawLayer', name: 'plotDrawLayer', alias: '态势标绘',
  group: 'plotDrawLayer', minZoom: 1, maxZoom: 24,
  visible: true, opacity: 1, type: 'vector',
};
xmap.registerLayer(thisLayer, layerItem);      // 或 xmap.LayerManager.registerLayer(...)
```

常用方法：
- `getLayer(layerID)` / `hasLayer(layerID)` → 按 id 取图层/判断存在。
- `get Layers(): Map<string, Layer>` → 图层字典。
- `changeLayerVisible(layerID, visible?)` / `changeLayerOpacity(layerID, opacity?)` → 可见性/透明度。
- `deleteLayerByID(layerID)` / `deleteLayerByIndex(index)` / `deleteLayer(layer)` / `clearLayers()` → 删除。
- `layerLocateHandler(layerID)` → 图层定位。
- `changeSwipeLayerByID(layerID)` / `changeSwipeLayerIndex(idx)` / `get SwipeLayerIndex` → 卷帘图层控制。
- `LayerItemList: Array<ILayerItem>` → 图层描述列表（图层树数据源）。

`EnumLayerType`：`grid | mvt | symbol | vector | json`。

## 在线底图 Provider

`xmap.addOnlineLayer(layerID, tdtLocalURL?)` 添加在线底图。可用 provider：
`ArcgisLayerProvider`、`BaiduLayerProvider`、`BingLayerProvider`、`GaodeLayerProvider`、
`GoogleLayerProvider`、`MapboxLayerProvider`、`TencentLayerProvider`，以及 `createBaiduMap`。
内置底图清单 `BasicLayerList: IBasicLayer[]`（`{ id, label, type, layers, img }`），
可用于底图切换 UI。

百度地图坐标特殊，`xgis-ol` 内含 `BDProject` / `BaiDuSource` 处理。

## PrjGridTool（投影与切片方案，静态工具类）

`import { PrjGridTool } from 'xgis-ol'`，全部为静态方法。缓存 `prjMap` / `tileGridMap`。

### 投影构建与坐标转换
- `getProjection(prjInfo?: IProjInfo): Projection` → 由 `{ epsg, prjExtent?, proj4? }` 构建投影。
- `getProjectionOnline(epsgNum, epsgSearchURL?)` → 在线查询 EPSG 构建投影，返回 `{ projection, isGeographic }`。
- `toLonLat(coord, currentPrj)` → 当前投影 → 经纬度。
- `fromLonLat(center84, targetPrj)` / `fromLonLatCoordinate(center84, targetPrj)` → 经纬度 → 目标投影。
- `transformCoordinate(coord, fromPrj, toPrj)` → 任意投影间坐标转换。
- `getExtentFromTransform(extent, fromPrj, toPrj?)` → 范围转换。
- `transformGeometry(geom, fromPrj, toPrj?)` / `transformFeature(feature, ...)` / `transformFeatures(list, ...)` → 几何/要素重投影。
- `isGeographic(projection)` → 是否地理投影。
- 静态常量：`EPSG4326Extent`、`EPSG3857Extent`。

### 瓦片矩阵集（TileGrid）
- `getTDTTileGrid(isWebMercator?)` → 天地图瓦片矩阵集（默认墨卡托）。
- `createWMTSTileGrid(tileSchema: ITileGridSchema): WMTSTileGrid` → 自定义切片方案。
- `getExtentFromTransform`、比例尺换算：`getScaleToResolutionParam`、
  `computeScaleByResolution(res, isGeographic?, isChinaDPI?)`、`computeResolutionByScale(scale, ...)`。

`ITileGridSchema`：`{ rule, origin?, extent?, tileSize, resolutions }`。

### WMTS 元数据在线解析
- `getWMTSCapabilities(wmtsURL, layer)` → 请求并解析 GetCapabilities XML。
- `getXMLOptionsFromCapabilities(wmtsCap, isChinaDPI?, epsgSearchURL?)` → 解析为加载所需 options，
  可配合 `xmap.WMTSTool.addWMTSLayerByXMLOptions(xmlOptions)` 使用。

## WMTSTool（WMTS 与天地图）

`xmap.WMTSTool`（`get WMTSTool(): WMTSTool`）。

### 天地图
```ts
xmap.WMTSTool.addTDTLayer('vec_c', '天地图矢量');        // 注册进图层管理
xmap.WMTSTool.addTDTLayerOnly('cva_c', '注记');          // 仅加图层不注册
xmap.WMTSTool.addLocalTDTLayerXYZ(localURL, ['vec_c','cva_c']); // 离线 XYZ 天地图
```

### 通用 WMTS
```ts
const tileGrid = PrjGridTool.getTDTTileGrid(false); // 4326 矩阵
const prj = PrjGridTool.getProjection({ epsg: 'EPSG:4326', prjExtent: [-180,-90,180,90] });
const wmtsLayer = xmap.WMTSTool.addWMTSLayer(
  layerName, 'C', wmtsServiceURL, tileGrid, prj,
  'default', 'KVP', 'image/jpeg'
);
```

其它重载：
- `addWMTSLayer2(alias, layerName, tileMatrixSet, tileURL, tileGrid?, prj?, style?, minZoom?, maxZoom?)`
- `addWMTSLayerByOptions(layerOption, sourceOption, alias?)` / `addWMTSLayerByOptions2(targetXMap, ...)` → 专业版，直接给 ol WMTS options。
- `addWMTSLayerSelf(res: IWMTSLayerInfo, layerName)` → 按自定义 WMTS 元数据（多投影影像）加载。
- `addWMTSDebugLayer(wmtsLayer)` → 生成对应 TileGrid 的调试网格层。
- `addTileDebugLayer(tileGrid, prjLike)` → 任意矩阵调试层。
- `addWMTSLayerByXMLOptions(xmlOptions)` → 配合 `PrjGridTool.getXMLOptionsFromCapabilities`。

`IWMTSLayerInfo`：`{ name, bounds, minlevel, maxlevel, center?, level?, tileUrl?, prjInfo?, tileSchema?, style? }`。

## VTLayerTool（矢量切片 MVT）

```ts
import { VTLayerTool, PrjGridTool } from 'xgis-ol';
const vtTool = new VTLayerTool(xmap);
// 1) 通过 TileJson 加载（含投影/范围信息）
const mvtLayer = vtTool.addVTLayer(tileJson);          // 注册图层
vtTool.addVTLayerNoBounds(tileJson, alias?);           // 不按范围定位
// 2) 直接给 MVT 模板 URL（墨卡托）
vtTool.addVTLayerW('http://host/tiles/{z}/{x}/{y}.mvt', alias?, minZoom?, maxZoom?);
```

`tileJson` 结构 `ITileJson`（含 `prjInfo?`、`tileSchema?`、`center`、`bounds` 等）。
定位到 tileJson 中心：`PrjGridTool.fromLonLatCoordinate([tileJson.center[0], tileJson.center[1]], prjObj)`。
Mapbox 样式：`xgis-ol` re-export 了 `ol-mapbox-style`，可 `import { apply, ... } from 'xgis-ol'`。

## GeoJSON / 原生矢量图层

`xgis-ol` 不封装 GeoJSON 加载，直接用原生 ol：

```ts
import GeoJSON from 'ol/format/GeoJSON.js';
import { Vector as VectorLayer } from 'ol/layer.js';
import { Vector as VectorSource } from 'ol/source.js';

const source = new VectorSource({ url: './china.json', format: new GeoJSON() });
const layer = new VectorLayer({ source });
xmap.map.addLayer(layer as any);
// 卸载：xmap.map.removeLayer(layer)
```

跨投影读取时用 `readFeatures(content, { dataProjection:'EPSG:4326', featureProjection: prj })`。

## DrawFeatureTool（顶层绘制/定位框图层）

`import { DrawFeatureTool } from 'xgis-ol'`；`new DrawFeatureTool(xmap, drawStyle?)`。
用于在最上层画临时几何、定位范围框：
- `addFeature(feature)` / `addGeometry(geom)` / `addExtent84(extent)` → 添加。
- `clear()` / `removeVectorLayer()` / `reinitVectorLayer()` → 清理。
- `set FeatureStyle(style)` / `get TargetVectorLayer` → 样式与图层。
