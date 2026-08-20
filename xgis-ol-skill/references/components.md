# xgis-ol Vue 组件

`xgis-ol` 的 `components` 命名空间提供开箱即用的 Vue3 组件。使用前引入样式
`import 'xgis-ol/dist/index.css'`。图标离线注册用 `registerMapIconCollection`。

导出：`OLXMap`、`XMapView`、`DrawToolBar`、`MeasureToolBar`、`MenuToolBar`、
`SwipeToolBar`、`ZoomFullBar`、`ContextMenu`、`LayerTree`、`registerMapIconCollection`。

## OLXMap（地图容器组件）

自带初始化，`@mapInited` 回调拿 `xmap`。Props：
- `mapid: string`（默认 `'map'`）
- `mapgroup: string`
- `hasLayerManager: boolean`
- `initTDTLayers: string[]`（初始天地图图层，如 `['vec_c','cva_c']`）
- `viewProjection: string | object`（默认 `'EPSG:3857'`）
- `defaultCenter: number[]`
- `viewOptions: ViewOptions`
- `multiWorld: boolean`
- `enableContextMenu: boolean`

事件：`@mapInited`（`res.xmap`）。具名插槽：`#mapLeftPanel` 等扩展面板。

```vue
<template>
  <OLXMap mapid="map" :hasLayerManager="false" :defaultCenter="[108.95, 34.5]"
    :initTDTLayers="['vec_c', 'cva_c']" viewProjection="EPSG:4326"
    @mapInited="onMapInited">
    <template #mapLeftPanel><div class="leftPanel">左侧面板</div></template>
  </OLXMap>
</template>
<script setup lang="ts">
import { OLXMap, XMap } from 'xgis-ol';
import 'xgis-ol/dist/index.css';
let xMap: XMap;
function onMapInited(res) { xMap = res.xmap; }
</script>
```

## XMapView（带布局的地图视图）

在 `OLXMap` 基础上封装了图层树/工具条显隐、尺寸控制。额外 Props：
`hasLayerTree`、`viewHeight`、`viewWidth`、`initTDTLayers`、`viewProjection`、`viewOptions`、`multiWorld`。
暴露 `doLocation(x, y, z?)` 等方法，`@mapInited` 回调。适合快速搭一个完整地图页。

## LayerTree（图层树）

与 `xmap.LayerManager` 联动。Props：`xmap: XMap`、`moreContextMenu`（右键菜单项数组）。

```vue
<LayerTree :xmap="mapRef" :moreContextMenu="contextMenuList" />
```

`moreContextMenu` 项为 `ILayerContextItem`：`{ name?, value?, icon?, tag? }`。

## 工具条组件（DrawToolBar / MeasureToolBar / SwipeToolBar / MenuToolBar / ZoomFullBar）

统一模式：传 `:xmap="mapRef"`，其余可选。

```vue
<DrawToolBar :xmap="mapRef" />
<MeasureToolBar :xmap="mapRef" />
<SwipeToolBar :xmap="mapRef" v-if="mapRef" />
```

`ZoomFullBar` 额外 Props：`hasLayerTree`、`isInternet`、`hasFullScreen`；事件 `@locate`。
封装了缩放、全屏、回到初始视图、定位、图层树开关。

```vue
<ZoomFullBar :xmap="mapRef" :hasLayerTree="hasLayerTree" />
```

## ContextMenu（地图右键菜单）

Props：`xmap: XMap`（必填）、`target: string|Element`（地图容器 id）、
`moreMenuList: IMapContextItem[]`、`replace: boolean`。事件 `@itemClicked`。

```vue
<ContextMenu :xmap="mapRef" target="map" :moreMenuList="menuList" @itemClicked="doItemClick" />
```

`IMapContextItem`：`{ id?, label?, icon?, tag?, children? }`。空对象 `{}` 表示分隔线。

## 典型组合页（配置驱动 + 组件）

```vue
<template>
  <div class="MainMapWidget">
    <div id="map" class="mapstyle">
      <ZoomFullBar :xmap="mapRef" :hasLayerTree="hasLayerTree" class="xmap-zoombar" />
      <ContextMenu :xmap="mapRef" target="map" :moreMenuList="menuList" @itemClicked="doItemClick" />
    </div>
    <div class="layerTreeContainer" v-show="isLayerTreeShow">
      <LayerTree :xmap="mapRef" :moreContextMenu="contextMenuList" />
    </div>
  </div>
</template>
<script setup lang="ts">
import { computed, onMounted, ref } from 'vue';
import { Global, requestGet } from 'xframelib';
import { PrjGridTool, XMap, ZoomFullBar, ContextMenu, IMapContextItem, LayerTree } from 'xgis-ol';
import 'xgis-ol/dist/index.css';

const mapRef = ref<XMap>();
const hasLayerTree = ref(false);
const menuList: Array<IMapContextItem> = [{}, { id: 'other', label: '更多功能', icon: 'ic:round-other-houses' }];
const contextMenuList = [{}, { name: '上移', icon: 'gis:layer-up', value: 'upMove' }];
const isLayerTreeShow = computed(() => mapRef.value && mapRef.value.mapMenuState.layerTree);
function doItemClick(item) { Global.Message.info('点击：' + item.label); }

onMounted(async () => {
  const res = await requestGet('', 'DefaultMapConfig.json');
  const mapConfig = res.data;
  if (mapConfig.projInfo) mapConfig.viewOptions.projection = PrjGridTool.getProjection(mapConfig.projInfo);
  const xmap = XMap.initByMapConfig(mapConfig);
  mapRef.value = xmap;
  hasLayerTree.value = !!xmap.LayerManager;
  Global.XMap = xmap;
});
</script>
```

`#map` 容器需 `position: relative` 且有宽高；工具条用 `position: absolute` 定位于其中。

## 与 xframelib 的配合

本项目地图 widget 常与 `xframelib` 搭配：`Global.XMap` 全局挂地图、`Global.Message` 提示、
`requestGet`/`get` 拉配置、`H5Tool`（`windowResizeHandler`、`bindDropFileHanlder`、
`onPasteHandler`、`readFilePromise`）、`XWindow` 浮动面板、`SaveAs` 导出。
详见 xframelib-skill（若已安装）。
