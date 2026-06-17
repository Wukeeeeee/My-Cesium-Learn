# My Cesium Learn

CesiumJS 3D 地球学习项目，从零开始的练习记录。

---

## 项目列表

### 01-hello-cesium

入门第一个项目，创建 3D 地球。

**文件：**
- `01-hello-globe.html` — 创建第一个 3D 地球，飞往旧金山（带 3D 建筑）
- `02-shenzhen-practice.html` — 第二个练习，飞往深圳

**学到什么：**
- `Cesium.Viewer` 创建 3D 地球
- `flyTo` 飞到指定经纬度
- `Cartesian3.fromDegrees()` 经纬度转坐标
- `fetch` 读取本地 `apikey.txt` 管理 token
- `http-server` 启动本地服务器
- Debug：`flyto` 大小写错误、`getElementById` 找不到元素

---

### 02-province-markers

在地球上批量标注全国省会/直辖市/特区。

**文件：**
- `01-load-cities.html` — 基础版：加载全国省会红点+标签
- `02-label-distance-limit.html` — 防止标签穿透地球（`disableDepthTestDistance`）
- `03-distance-display.html` — 距离控制显示标签（`distanceDisplayCondition`），最完整版
- `china-cities.json` — 34个省级行政中心经纬度数据

**学到什么：**
- `fetch('china-cities.json')` + `forEach` 批量加点
- `verticalOrigin: TOP` + `pixelOffset` 让标签显示在点下方
- `disableDepthTestDistance: 0` 标签不穿透地球背面
- `distanceDisplayCondition` 控制标签在特定距离显示，解决港澳重叠问题
- `setView` 瞬间定位 vs `flyTo` 动画定位
- 地形瓦片国内访问慢的问题

---

### 03-gdp-visualization

各省会/直辖市/特区 GDP 柱状图可视化。

**文件：**
- `01-gdp-chart.html` — GDP 柱状图（带地形，最完整版）
- `02-gdp-simple.html` — GDP 柱状图（无地形，简单版）
- `china-gdp.json` — 城市GDP数据（含经纬度、海拔），共34条

**学到什么：**
- `box` 创建3D柱子，`position` 高度 = 柱高/2 让柱子贴地
- `verticalOrigin: BOTTOM` 标签在柱子顶往上长
- `disableDepthTestDistance: 0` 标签不被柱子挡住
- 高海拔城市（拉萨3650m）地形导致柱子埋地里的问题

---

### 04-performance-optimization

Cesium 性能优化实验，解决集成显卡拖拽卡顿问题。

**文件：**
- `00-original-entity-box.html` — 原版 Entity Box（baseline）
- `01-billboard.html` — Canvas 画柱子 + Billboard 贴图
- `02-points.html` — Point 散点图
- `03-instancing.html` — Primitive Instancing 合并 draw call
- `99-ultimate-optimization.html` — 终极优化（全关 + requestRenderMode）

**学到什么：**
- `requestRenderMode: true` 不动时不渲染，最关键的优化
- `Primitive` + `GeometryInstance` 合并多个 Box 为 1 个 draw call
- Billboard 用 Canvas 画图代替 3D 几何体，零 GPU 负担
- 关掉光照/雾/HDR/抗锯齿，不影响数据展示但省 GPU

> 📖 **深度阅读：** [Primitive 定位原理详解](notes/cesium/Primitive-Positioning.md) — 为什么 Entity 只需要 `position`，Primitive 却要 `modelMatrix` + `eastNorthUpToFixedFrame`，看完就懂。

---

## 启动方式

```bash
cd 项目目录
npx http-server -p 8080 -c-1
```

浏览器打开 `http://localhost:8080`。

> 必须用 http-server 启动，不支持 `file://` 双击打开。

## 笔记

详细概念参考和函数速查见 [notes/cesium/](notes/cesium/) 目录。
