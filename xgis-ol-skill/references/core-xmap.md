# 核心：XMap 与地图初始化

`xgis-ol` 的入口是 `XMap` 类（`import { XMap } from 'xgis-ol'`）。它封装了 OpenLayers 的
`ol/Map`，并集成图层管理、投影工具、WMTS、事件总线等。使用前务必引入样式：

```ts
import { XMap } from 'xgis-ol';
import 'xgis-ol/dist/index.css';
```

## 三种初始化方式

### 1. 纯手动（最基础，适合演示/最小示例）

```ts
import { onMounted } from 'vue';
import { XMap } from 'xgis-ol';
import 'xgis-ol/dist/index.css';
import { Tile } from 'ol/layer';
import { OSM } from 'ol/source';

onMounted(() => {
  // 参数：target 容器id, mapGroup 组名, hasLayerManager 是否启用图层管理
  const xmap = new XMap('map', 'mapGroupName');
  xmap.initMapView({
    zoom: 5,
    center: [116.462, 40.248],
    minZoom: 1,
    maxZoom: 22,
    projection: 'EPSG:4326',
  });
  // 原生 ol 图层通过 xmap.map 添加
  xmap.map.addLayer(new Tile({ source: new OSM() }));
});
```

关键点：
- `new XMap(target, mapGroup?, hasLayerManager?)`：`target` 是 DOM 容器 id 字符串。
- **必须在 `onMounted` 后调用**（DOM 容器需已存在），且容器需有真实宽高，否则地图空白。
- `initMapView(opt_options?: ViewOptions)` 创建并返回 `ol/View`；`projection` 缺省 `EPSG:3857`。
- `xmap.map` 是底层 `ol/Map`；任何原生 ol 操作都走 `xmap.map`。

### 2. 配置驱动（推荐用于业务系统）

通过 `IMapConfig` 配置对象一次性初始化图层、控件、交互：

```ts
import { XMap, PrjGridTool } from 'xgis-ol';
import { Global, requestGet } from 'xframelib';

const configResult = await requestGet('', 'DefaultMapConfig.json');
const mapConfig = configResult.data;           // 见 IMapConfig 结构
if (mapConfig.projInfo) {
  // 自定义投影：先构建 Projection 放入 viewOptions
  mapConfig.viewOptions.projection = PrjGridTool.getProjection(mapConfig.projInfo);
}
const xmap = XMap.initByMapConfig(mapConfig);  // 静态工厂方法
Global.XMap = xmap;                            // 全局挂载，供其它 widget 取用
```

`IMapConfig` 主要字段（详见 `xgis-ol` 的 `core/Models.d.ts`）：

```ts
interface IMapConfig {
  id: string;                     // 地图容器 id
  group?: string;                 // 地图组
  hasLayerManager?: boolean;      // 是否启用图层管理
  isInternet?: boolean;           // 是否在线（决定天地图等在线图层）
  tdtXYZLocalURL?: string;        // 离线天地图 XYZ 模板地址
  projInfo?: IProjInfo;           // 自定义投影信息
  viewOptions: ViewOptions;       // ol 视图参数
  layers?: Array<string>;         // 初始在线图层 id（如 'vec_c','cva_c'）
  controls?: Array<IControlOption>;      // 控件
  interactions?: Array<IControlOption>;  // 交互
}
```

`IProjInfo`：`{ epsg: string; prjExtent?: [minx,miny,maxx,maxy]; proj4?: string }`。
自定义投影必须提供 `prjExtent`。

配置里的 `controls` / `interactions` 是 `IControlOption`：`{ key: string; type?: string; options?: any }`。
框架内置了 `defaultMapConfig`（Models 中导出）可作为模板参考。

### 3. Vue 组件驱动（最省事，见 components.md）

用 `<OLXMap>` 或 `<XMapView>` 组件，`@mapInited` 回调里拿到 `xmap`。

## XMap 常用 API

### 视图与定位
- `initMapView(viewOptions?)` → 创建视图。
- `get MapView(): View` / `setView(view)` → 视图存取。
- `setProjection(prj, newViewOptions?)` → 运行时切换投影。
- `zoomToCenter(center84, level?)` → 按 WGS84 中心点定位。
- `zoomToExtent(extent84)` → 按 WGS84 范围定位。
- `zoomTo(targetExtent, proj?)` → 按目标投影范围定位。
- `goHomeView()` → 回到初始视图。
- `get CurrentPosition(): Coordinate` → 当前鼠标位置坐标。

### 图层
- `get LayerManager(): LayerManager` → 图层管理对象（见 layers-and-projection.md）。
- `registerLayer(layer, layerItem)` → 把手动创建的 ol 图层注册进图层管理/图层树。
- `removeLayer(layer)` → 移除图层。
- `changeLayerOrder(oldIndex, newIndex)` → 调整顺序。
- `addOnlineLayer(layerID, tdtLocalURL?)` → 添加在线底图（天地图/高德/谷歌/Bing/Mapbox/百度/腾讯/Arcgis）。

### 工具对象（惰性获取）
- `get WMTSTool(): WMTSTool` → 加载 WMTS / 天地图。
- `get PrintTool(): PrintTool` → 打印。
- `get RollSwipe(): RollSwipe` → 卷帘。
- `set TDTKeys(value: string[])` / `get TDTKey` → 天地图密钥（数组，内部随机取一个）。

### 交互 / 控件扩展字典
XMap 维护 `InteractionsDiction` 和 `ControlsDiction` 两个 Map，用于按 key 管理自定义交互/控件：
- `addInteractionExt(key, interaction)` / `getInteractionExt(key)` / `removeInteractionExt(key)` / `containsInteractionExt(key)` / `clearInteractionExt()`
- `addControlExt(key, control)` / `getControlExt(key)` / `removeControlExt(key)` / `containsControlExt(key)` / `clearControlExt()`
- `addDefaultControl(item: IControlOption)` / `addDefaultInteraction(item)` → 按配置项添加内置控件/交互。

内置 key 常量：`DefaultControlKeys = { overview, target, scale, scaleline }`，
`DefaultInteractionKeys = { FishEyeClip }`。

### 视图同步（多地图联动）
- `enableMapSyncView()` / `disableMapSyncView()` → 开启/取消与同组其它地图的视图同步。

### 事件总线
- `mapEventBus: MapEventBusClass` → 见 events。`newEvtArgs(data?, eventObject?)` 生成 `MapEventArgs`。

### 响应式菜单状态
`xmap.mapMenuState` 是响应式对象，控制各功能面板显隐：
`{ layerTree, dataPanel, location, drawTool, measureTool, swipeTool, otherTool, printTool, selectTool, popupPanel, tagState }`。
Vue 里可 `computed(() => xmap.mapMenuState.layerTree)` 驱动面板显示。

### 销毁
- `dispose()` → 注销地图，释放资源（组件卸载时调用）。

## 生命周期约定
- 创建：`onMounted` 之后、容器存在时。
- 卸载：`onUnmounted` 中 `xmap.dispose()`，并移除自行添加的图层/事件。
- 不要把 `XMap` 实例放进 Vue 的 `reactive`/`ref` 深层响应式（用 `ref<XMap>()` 存引用即可，或挂 `Global.XMap`）。
