# Cesium Learning Projects

## 01-hello-cesium

第一个 Cesium 入门项目。

**项目位置：** `E:/my_repo/MyCesium/01-hello-cesium/`
`
### 文件说明
- `01-hello-globe.html` — 创建第一个 3D 地球，飞往旧金山（带 3D 建筑）
- `02-shenzhen-practice.html` — 第二个练习，飞往深圳（不带 3D 建筑）
- `apikey.txt` — Cesium Ion token，两个 html 共用

### 在这个项目中学到了什么
- 用 `Cesium.Viewer` 创建 3D 地球
- 用 `flyTo` 飞到指定经纬度
- `Cartesian3.fromDegrees()` 把经纬度转成三维坐标
- API Key 的作用与配置
- `fetch` 读取本地 `apikey.txt` 管理 token
- `http-server` 启动本地服务器（不能双击用 `file://` 打开）
- 常见 debug：`flyto` 大小写错误、`getElementById` 找不到元素

## 02-province-markers

第二个项目：利用 JSON 数据在地球上批量标注全国省会/直辖市/特区。

**项目位置：** `E:/my_repo/MyCesium/02-province-markers/`

### 文件说明
- `01-load-cities.html` — 基础版：加载全国省会红点+标签，默认飞中国
- `02-label-distance-limit.html` — 加了 disableDepthTestDistance 防止标签穿透地球
- `03-distance-display.html` — 加了 distanceDisplayCondition 控制标签远近显示，最完整版
- `china-cities.json` — 中国34个省级行政中心经纬度数据（直辖市4 + 省会28 + 特区2）
- `apikey.txt` — Cesium Ion token

### 在这个项目中学到了什么
- **JSON 加载**：通过 `fetch('china-cities.json')` 读取外部数据，循环 `forEach` 批量加点
- **标签偏移**：`verticalOrigin: Cesium.VerticalOrigin.TOP` 让标签显示在点下方，配合 `pixelOffset: new Cesium.Cartesian2(0, 10)` 微调位置
- **深度测试**：`disableDepthTestDistance: 0` 让背面的标签被地球挡住，不会穿透显示
- **距离显示控制**：`distanceDisplayCondition: new Cesium.DistanceDisplayCondition(0, 500000)` 控制标签在特定距离范围内才显示，解决港澳等近距离城市拉远时重叠的问题
- **视角切换**：`viewer.camera.setView()` 瞬间定位到亚洲（无动画，比 `flyTo` 更快）
- **地形 vs 无地形**：`Cesium.Terrain.fromWorldTerrain()` 增加了地球真实起伏，但地形瓦片从美国服务器加载，国内访问慢容易超时失败

## 03-gdp-visualization

第三个项目：各省会/直辖市/特区 GDP 柱状图可视化。

**项目位置：** `E:/my_repo/MyCesium/03-gdp-visualization/`

### 文件说明
- `01-gdp-chart.html` — GDP 柱状图（带地形，最完整版）
- `02-gdp-simple.html` — GDP 柱状图（无地形，简单版）
- `china-gdp.json` — 城市GDP数据（含经纬度、海拔），共34条

### 在这个项目中学到了什么
- **Box 柱子**：用 `viewer.entities.add({ box: { dimensions } })` 创建3D柱子
- **柱子贴地**：`position` 高度 = `dimensions.z / 2`，中心抬一半，底部贴地面
- **标签订位**：`verticalOrigin: BOTTOM` 让标签在柱子顶往上长
- **标签穿透**：`disableDepthTestDistance: 0` 防止标签被柱子挡住
- **地形问题**：地形瓦片从美国加载慢，高海拔城市（拉萨3650m）柱子会埋地里
- **外部数据加载**：`fetch('china-gdp.json')` 批量创建柱子+标签组合

---

## 04-performance-optimization

第四个项目：Cesium 性能优化实验。

**项目位置：** `E:/my_repo/MyCesium/04-performance-optimization/`

### 文件说明
- `00-original-entity-box.html` — 原版 Entity Box（未优化，baseline）
- `01-billboard.html` — Canvas 画柱子图片，Billboard 贴图，最轻量
- `02-points.html` — Point 散点图，用点大小表示 GDP
- `03-instancing.html` — Primitive Instancing 合并 34 个 Box 为 1 个 draw call

### 在这个项目中学到了什么
- **性能瓶颈诊断**：Cesium 卡顿不一定是电脑问题，而是 Entity Box 的 34 个 draw call 太重
- **Billboard 替代 3D**：用 Canvas 画 2D 图片代替 3D Box，GPU 零负担，帧率翻 3-4 倍
- **Primitive Instancing**：`GeometryInstance` + `Primitive` 合并 draw call，保留 3D 效果但性能明显提升
- **Point 散点图**：最简单轻量的数据可视化方式，适合 Iris Xe 等集成显卡
- **优化思路**：先关特效（光照/雾/HDR/抗锯齿），再换渲染方案（Billboard > Point > Instancing > Entity）

---

## 注意

- `cesium.com` **主站**（ion 控制台、文档）在国内访问慢/可能被墙
- 但 `cesium.com/downloads/cesiumjs/...` 的 CDN 地址在国内可以正常访问，**不需要换源**
- 如果以后遇到 Cesium 加载 render 失败，先检查网络和 token，再考虑换国内 CDN

## API Key

Token 存在 `apikey.txt`，所有 html 通过 `fetch('apikey.txt')` 读取，换 token 只改这一个文件即可。

```js
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();
```

> 注意：必须用 `http-server` 启动（`file://` 直接打开不支持 fetch）。
