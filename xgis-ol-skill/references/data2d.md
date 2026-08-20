# xgis-data2d 矢量数据导入导出

`xgis-data2d` 基于 MapShaper 提取，用于**矢量数据格式转换**：Shapefile / GeoJSON /
TopoJSON / CSV 的导入导出、投影变换、拓扑简化。它本身不操作地图，产物需转成 ol 图层再加入地图。

```ts
import {
  import2DFiles, internal, saveDataset, saveDatasetFromGeojson,
  getProj, transformDataset,
} from 'xgis-data2d';

// internal 里含更细粒度 API
const {
  importContent, exportDatasetAsGeoJSON, setDatasetCrsInfo,
  exportFileContent, importGeoJSON,
} = internal;
```

## 导入文件 → 加到地图

`import2DFiles(files: File[])` 解析拖拽/选择的文件（支持 shp 的一组文件），返回
`{ dataset, layername }[]`。再用 `exportDatasetAsGeoJSON` 转 GeoJSON，交给 ol 渲染：

```ts
import GeoJSON from 'ol/format/GeoJSON.js';
import { Vector as VectorLayer } from 'ol/layer.js';
import { Vector as VectorSource } from 'ol/source.js';
import { XMap } from 'xgis-ol';

async function dragFileHandler(fileList: FileList) {
  const files = Array.from(fileList);
  const result = await import2DFiles(files);
  const xmap = Global.XMap as XMap;
  const prj = xmap.MapView.getProjection();
  result.forEach(it => {
    const content = exportDatasetAsGeoJSON(it.dataset, {});
    // data2d 产物默认 EPSG:4326，按地图投影重投影
    const feats = new GeoJSON().readFeatures(content, {
      dataProjection: 'EPSG:4326', featureProjection: prj,
    });
    const layer = new VectorLayer({ source: new VectorSource({ features: feats }) });
    xmap.map.addLayer(layer as any);
  });
}
```

配合 `xframelib` 的 `H5Tool.bindDropFileHanlder('map', dragFileHandler)` 实现拖拽入图，
`H5Tool.onPasteHandler(handler)` 实现粘贴入图。

## 导出（地图矢量 → Shapefile / GeoJSON）

```ts
// ol 要素 → GeoJSON 文本 → dataset → 转投影 → 导出
const features = geojsonLayer.getSource()?.getFeatures();
const geoInfo = new GeoJSON().writeFeatures(features);   // 默认地图投影，如 3857
const dataset = importGeoJSON(geoInfo, {});
const ds = await transformDataset(dataset, 'EPSG:3857', 'EPSG:4326'); // 转经纬度
saveDataset(dataset, 'shapefile', true, '4326.zip');     // 导出 shp（zip）
```

从静态 GeoJSON 直接导出：

```ts
saveDatasetFromGeojson(geoInfoText, 'shapefile');
```

## 设置坐标系（CRS）

导入无投影信息的数据时，需手动设 CRS（esri wkt）：

```ts
const dataset = importContent({ json: { filename: 'china.json', content: JSON.stringify(obj) } });
setDatasetCrsInfo(dataset, {
  prj: 'GEOGCS["GCS_WGS_1984",DATUM["D_WGS_1984",SPHEROID["WGS_1984",6378137,298.257223563]],PRIMEM["Greenwich",0],UNIT["Degree",0.017453292519943295]]',
});
saveDataset(dataset, 'shapefile');
```

## 常用 API 速览

- `import2DFiles(files)` → 批量导入文件为 datasets。
- `internal.importContent({ json | ... })` → 从内容对象导入。
- `internal.importGeoJSON(text, opts)` → GeoJSON 文本 → dataset。
- `internal.exportDatasetAsGeoJSON(dataset, opts)` → dataset → GeoJSON。
- `internal.exportFileContent(...)` → 导出为指定格式内容。
- `internal.setDatasetCrsInfo(dataset, { prj })` → 设投影（esri wkt）。
- `transformDataset(dataset, fromEPSG, toEPSG)` → 数据集重投影（异步）。
- `getProj(...)` → 投影相关辅助。
- `saveDataset(dataset, format, zipped?, filename?)` → 导出并下载（`format` 如 `'shapefile'`、`'geojson'`、`'topojson'`）。
- `saveDatasetFromGeojson(geojsonText, format)` → 直接从 GeoJSON 导出下载。

## 注意
- data2d 是数据层工具，**渲染交给 ol / xgis-ol**：导入后转 GeoJSON 建 `VectorLayer`，导出前把 ol 要素 `writeFeatures`。
- 注意投影：data2d 默认按 EPSG:4326 处理；地图多为 EPSG:3857 或自定义投影，导入用 `featureProjection`、导出用 `transformDataset` 对齐。
- Shapefile 是多文件（.shp/.dbf/.shx/.prj），导入时把这一组文件一起传给 `import2DFiles`。
