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

## 启动方式

```bash
cd 项目目录
npx http-server -p 8080 -c-1
```

浏览器打开 `http://localhost:8080`。

> 必须用 http-server 启动，不支持 `file://` 双击打开。

## 笔记

详细概念参考和函数速查见 [notes/cesium/](notes/cesium/) 目录。
