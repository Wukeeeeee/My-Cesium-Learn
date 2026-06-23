# My Cesium Learn

CesiumJS 3D 地球学习项目，从零开始的练习记录。

---

## 项目列表

### 01-create-globe

入门第一个项目，创建 3D 地球。

**文件：**
- `01-first-globe.html` — 创建第一个 3D 地球，飞往旧金山
- `02-fly-to-location.html` — 飞往深圳，练习相机角度控制

**学到什么：**
- `Cesium.Viewer` 创建 3D 地球
- `flyTo` 飞到指定经纬度
- `Cartesian3.fromDegrees()` 经纬度转坐标
- `heading` + `pitch` 控制相机角度
- `fetch` 读取本地 `apikey.txt` 管理 token

---

### 02-add-markers

在地球上批量标注全国省会/直辖市/特区。

**文件：**
- `01-add-markers.html` — 从JSON读取34个城市数据，批量标红点+名称标签
- `02-label-fix.html` — 解决标签穿透地球背面问题
- `03-label-distance.html` — 相机距离控制标签显隐

**学到什么：**
- `fetch('china-cities.json')` + `forEach` 批量加点
- `verticalOrigin: TOP` + `pixelOffset` 控制标签位置
- `disableDepthTestDistance: 0` 标签不穿透地球背面
- `distanceDisplayCondition` 控制标签在特定距离显示
- `setView` 瞬间定位 vs `flyTo` 动画定位

---

### 03-make-chart

GDP 柱状图可视化。

**文件：**
- `01-chart-entity.html` — 用 Entity Box 画柱子（带地形）
- `02-chart-simple.html` — 简化版，无地形，柱子贴地
- `03-chart-fast.html` — Primitive Instancing 性能优化（34根柱子→1次draw call）
- `04-test.html` — 练习文件

**学到什么：**
- `box` 创建3D柱子，`position = h/2` 让柱子贴地
- `verticalOrigin: BOTTOM` 控制标签位置
- Primitive Instancing：`GeometryInstance` + `Primitive` 合并draw call
- 性能优化：`requestRenderMode` + 关光照 + 关抗锯齿

---

### 04-load-geojson

加载 GeoJSON 数据。

**文件：**
- `01-load-province.html` — 加载中国省界 + 南海九段线
- `china-province.geojson` — 中国34个省级行政区边界数据
- `southsea.geojson` — 南海九段线数据

**学到什么：**
- `GeoJsonDataSource.load()` 加载 GeoJSON
- `stroke` / `strokeWidth` / `fill` 控制边界线样式
- `clampToGround` 控制是否贴地
- `delete data.crs` 解决CRS不兼容问题
- 南海九段线用 `MultiLineString` 类型

---

### 05-3d-tiles

OSM 全球建筑白膜加载 + CustomShader 发光效果。

**文件：**
- `01-osm-buildings-nyc.html` — 加载 OSM 全球建筑，飞到纽约
- `02-osm-buildings-style-test.html` — 飞广州，测试 Cesium3DTileStyle
- `glow.html` — CustomShader 建筑发光 + 夜景氛围
- `cesium-building-glow-ref.md` — 掘金文章参考

**学到什么：**
- `createOsmBuildingsAsync()` 加载 OSM 全球建筑（无需资产ID）
- `Cesium3DTileStyle` 不支持 `emissive`，要用 `CustomShader` ✅
- `CustomShader` + `material.emissive` 实现建筑自发光
- 夜景氛围：关太阳 + 深色背景 + 关光照
- `Cesium3DTileset.fromUrl()` 加载指定资产ID

---

## 启动方式

```bash
cd 项目目录
npx http-server -p 8080 -c-1
```

浏览器打开 `http://localhost:8080`。

> 必须用 http-server 启动，不支持 `file://` 双击打开。

## 笔记

详细概念参考见 [notes/cesium/](notes/cesium/) 目录。

---

*CesiumJS 版本: 1.137 · 学习时间: 2026.06*
