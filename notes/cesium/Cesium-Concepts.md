# Cesium Concepts — 函数参考手册

> 常用函数的参数说明，忘了就翻这里。

---

## Viewer — 创建 3D 地球

```js
const viewer = new Cesium.Viewer(containerId, options)
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `containerId` | ✅ | HTML 中 div 的 id，如 `'cesiumContainer'` |
| `options.terrain` | ❌ | 地形，`Cesium.Terrain.fromWorldTerrain()` 加载全球地形（慢） |
| `options.imageryProvider` | ❌ | 影像源，默认用 Cesium Ion，可换 ArcGIS 等 |

**示例：**

```js
// 不带地形（加载快）
const viewer = new Cesium.Viewer('cesiumContainer');

// 带地形（加载慢，国内可能超时）
const viewer = new Cesium.Viewer('cesiumContainer', {
    terrain: Cesium.Terrain.fromWorldTerrain()
});
```

---

## Cartesian3.fromDegrees — 经纬度转坐标

```js
Cesium.Cartesian3.fromDegrees(经度, 纬度, 高度)
```

| 参数 | 类型 | 说明 |
|------|------|------|
| 经度 | 数字 | -180 ~ 180，东经为正（中国约 73~135） |
| 纬度 | 数字 | -90 ~ 90，北纬为正（中国约 18~54） |
| 高度 | 数字 | 海拔，单位**米**。0 为地面，越长越高 |

---

## flyTo — 飞行动画

```js
viewer.camera.flyTo(options)
```

| 参数 | 必填 | 说明 |
|------|------|------|
| `destination` | ✅ | 目标位置，`Cartesian3.fromDegrees(lon, lat, height)` |
| `orientation.heading` | ❌ | 朝向角度（弧度），`Cesium.Math.toRadians(度)`，0=正北 |
| `orientation.pitch` | ❌ | 俯仰角度（弧度），负值=向下看，-90=垂直看地面 |
| `orientation.roll` | ❌ | 旋转角度（弧度），一般不管 |
| `duration` | ❌ | 飞行时间（秒），默认 3 秒 |

**示例：**

```js
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(113.27, 23.13, 4000000),
    orientation: {
        heading: Cesium.Math.toRadians(0.0),
        pitch: Cesium.Math.toRadians(-30.0),
    }
});
```

高度参考：4000000（400万米）= 能看到整个中国

---

## setView — 瞬间定位（无动画）

```js
viewer.camera.setView(options)
```

参数跟 `flyTo` 一样，但没有飞行动画，瞬间切换。

---

## entities.add — 加点/线/面/模型

```js
viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(lon, lat, height),
    point: { ... },
    label: { ... },
    model: { uri: 'model.glb' }
})
```

### point（点）

| 属性 | 说明 |
|------|------|
| `pixelSize` | 点的大小（像素），默认 1 |
| `color` | 颜色，如 `Cesium.Color.RED` / `Cesium.Color.BLUE` |
| `outlineColor` | 边框颜色 |
| `outlineWidth` | 边框粗细 |

### label（文字标签）

| 属性 | 说明 |
|------|------|
| `text` | 显示的文本，如 `'广州'` |
| `font` | 字体大小+样式，如 `'20px sans-serif'` |
| `color` / `fillColor` | 文字颜色 |
| `outlineColor` | 文字边框颜色 |
| `outlineWidth` | 文字边框粗细 |
| `style` | 文字样式，`Cesium.LabelStyle.FILL_AND_OUTLINE` |
| `pixelOffset` | 偏移像素，`new Cesium.Cartesian2(x, y)`，正x=右，正y=上 |
| `verticalOrigin` | 垂直对齐：`Cesium.VerticalOrigin.TOP`（点在文字下方） / `BOTTOM` / `CENTER` |
| `horizontalOrigin` | 水平对齐：`LEFT` / `RIGHT` / `CENTER` |

### 通用属性

| 属性 | 说明 |
|------|------|
| `disableDepthTestDistance` | 设为 `0` 表示标签在地球背面就被挡住，不穿透 |
| `distanceDisplayCondition` | `new Cesium.DistanceDisplayCondition(最近, 最远)`，单位米 |

**示例：**

```js
viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(113.27, 23.13, 0),
    point: { pixelSize: 10, color: Cesium.Color.RED },
    label: {
        text: '广州',
        font: '20px sans-serif',
        color: Cesium.Color.WHITE,
        outlineColor: Cesium.Color.BLACK,
        outlineWidth: 2,
        style: Cesium.LabelStyle.FILL_AND_OUTLINE,
        verticalOrigin: Cesium.VerticalOrigin.TOP,
        pixelOffset: new Cesium.Cartesian2(0, 10),
    },
    disableDepthTestDistance: 0,
    distanceDisplayCondition: new Cesium.DistanceDisplayCondition(0, 1000000),
})
```

---

## DistanceDisplayCondition — 距离控制

```js
new Cesium.DistanceDisplayCondition(最近距离, 最远距离)
```

相机在 `最近` ~ `最远` 米之间才显示，超过范围隐藏。

| 示例 | 效果 |
|------|------|
| `(0, 1000000)` | 0 ~ 100 万米（约拉到省范围可见） |
| `(0, 500000)` | 0 ~ 50 万米（拉到城市范围） |
| `(0, 10000000)` | 0 ~ 1000 万米（拉到太空都能见） |

---

## Cartesian2 — 2D 像素偏移

```js
new Cesium.Cartesian2(x, y)
```

| 参数 | 说明 |
|------|------|
| `x` | 水平偏移，正=右，负=左 |
| `y` | 垂直偏移，正=上，负=下 |

常用于 `pixelOffset`：

```js
pixelOffset: new Cesium.Cartesian2(0, 10)   // 往下移10像素
pixelOffset: new Cesium.Cartesian2(-20, 0)  // 往左移20像素
```

---

## Math.toRadians — 角度转弧度

```js
Cesium.Math.toRadians(度)
```

Cesium 里角度用弧度，但人能看懂的是度，用这个函数转换。

| 度 | 弧度 | 用途 |
|----|------|------|
| 0 | 0 | 正北 |
| 45 | 0.785 | 东北方向 |
| -15 | -0.262 | 稍微向下俯视 |
| -90 | -1.571 | 垂直向下看地面 |

---

## Color — 颜色

颜色常量：`Cesium.Color.RED` / `GREEN` / `BLUE` / `WHITE` / `BLACK` / `YELLOW` / `ORANGE`

自定义颜色：
```js
Cesium.Color.fromCssColorString('#ff0000')  // 用 CSS 颜色
new Cesium.Color(255, 0, 0, 1)              // (r, g, b, 透明度)
```

---

## fetch — 读外部文件

```js
const res = await fetch('文件名');
const data = await res.json();      // 读 JSON 文件
const text = await res.text();      // 读 txt 文件
```

| 方法 | 用途 |
|------|------|
| `res.json()` | 解析成 JSON 对象（数组/字典） |
| `res.text()` | 解析成纯文本字符串 |
| `.trim()` | 去掉首尾空格/换行 |

---

## createOsmBuildingsAsync — 3D 建筑

```js
const buildingTileset = await Cesium.createOsmBuildingsAsync();
viewer.scene.primitives.add(buildingTileset);
```

加载全球 3D 建筑（OpenStreetMap 数据），需要网络。

---

## 常见问题速查

| 现象 | 原因 |
|------|------|
| 黑屏 / 地球不显示 | Token 无效、网络不通、或地形加载超时 |
| 控制台报错 `Cannot read properties of null` | `getElementById` 找不到对应的 id |
| 方法不生效 | 大小写错误：`flyto` → `flyTo` |
| fetch 报错 | 用 `file://` 打开了，必须 `npx http-server` |
| 标签穿透地球 | 加 `disableDepthTestDistance: 0` |
| 标签拉远重叠 | 加 `distanceDisplayCondition` 控制显示范围 |
| 标签跟点重叠 | 加 `verticalOrigin: TOP` + `pixelOffset: (0, 10)` |

---

## 为什么需要 http-server

因为 `file://` 协议下浏览器禁止发 fetch 请求（安全机制），必须用 http-server 提供 HTTP 服务。

```bash
cd 项目目录
npx http-server -p 8080 -c-1
```
