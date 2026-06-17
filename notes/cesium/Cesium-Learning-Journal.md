# Cesium Learning Projects

## 01-hello-cesium

**项目位置：** `E:/my_repo/MyCesium/01-hello-cesium/`

**文件：** `01-hello-globe.html`（旧金山+3D建筑）、`02-shenzhen-practice.html`（深圳）

### 代码

```js
// API Key
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();

// 创建地球
const viewer = new Cesium.Viewer('cesiumcontainer');

// 飞到中国
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(110, 35, 8000000),
    // fromDegrees(经度, 纬度, 高度米)
    // 800万米 ≈ 看整个中国
});
```

### 学到了什么
- `Cesium.Viewer` 创建 3D 地球
- `flyTo` 飞到指定经纬度
- `Cartesian3.fromDegrees()` 经纬度转坐标
- `fetch` 读 `apikey.txt` 管理 token
- 必须用 `http-server` 启动，不支持 `file://`
- Debug：`flyto` 大小写错误、`getElementById` 找不到元素

---

## 02-province-markers

**项目位置：** `E:/my_repo/MyCesium/02-province-markers/`

**文件：** `01-load-cities.html`（基础）、`02-label-distance-limit.html`（防穿透）、`03-distance-display.html`（距离控制）

### 代码

```js
// JSON 数据格式
// [ { "name":"广州", "lon":113.27, "lat":23.13 }, ... ]

// 读取
const res = await fetch('china-cities.json');
const data = await res.json();

// forEach 批量加实体
data.forEach(item => {
    viewer.entities.add({
        position: Cesium.Cartesian3.fromDegrees(item.lon, item.lat, 0),
        point: {
            pixelSize: 10,           // 点大小
            color: Cesium.Color.RED,
        },
        label: {
            text: item.name,
            font: '20px sans-serif',
            color: Cesium.Color.WHITE,
            verticalOrigin: Cesium.VerticalOrigin.TOP,  // 点在文字下方
            pixelOffset: new Cesium.Cartesian2(0, 10),  // 往下移10px
            disableDepthTestDistance: 0,                 // 不穿透地球背面
            distanceDisplayCondition:
                new Cesium.DistanceDisplayCondition(0, 500000),
                // 相机距离 0~50万米 才显示
        },
    });
});
```

### 学到了什么
- **JSON 加载**：`fetch('china-cities.json')` + `forEach` 批量加点
- **标签偏移**：`verticalOrigin: TOP` + `pixelOffset: (0, 10)` 让标签在点下方
- **深度测试**：`disableDepthTestDistance: 0` 背面标签被地球挡住
- **距离控制**：`distanceDisplayCondition(0, 500000)` 控制远近显示
- **视角切换**：`setView()` 瞬间定位 vs `flyTo()` 动画
- **地形问题**：`Terrain.fromWorldTerrain()` 国内访问慢

---

## 03-gdp-visualization

**项目位置：** `E:/my_repo/MyCesium/03-gdp-visualization/`

**文件：** `01-gdp-chart.html`（带地形）、`02-gdp-simple.html`（无地形）、`03-instancing.html`（Instancing 版）、`test.html`（带注释练习版）

### Entity Box 柱状图（基础版）

```js
const h = gdp * 20;        // 数据 → 高度
const w = 80000;           // 柱子粗细

viewer.entities.add({
    position: Cartesian3.fromDegrees(lon, lat, h / 2),  // h/2 = 贴地
    box: {
        dimensions: new Cartesian3(w, w, h),   // (宽, 深, 高)
        material: Cesium.Color.RED,
    },
});
```

### 柱顶标签

```js
viewer.entities.add({
    position: Cartesian3.fromDegrees(lon, lat, h + 1000),  // 柱顶往上
    label: {
        text: name + '\n' + gdp + '亿',   // \n 换行
        font: '20px sans-serif',
        color: Cesium.Color.WHITE,
        verticalOrigin: Cesium.VerticalOrigin.BOTTOM,  // 往上长
        disableDepthTestDistance: 0,
        distanceDisplayCondition:
            new Cesium.DistanceDisplayCondition(0, 10000000),
    }
});
```

| 公式 | 效果 |
|------|------|
| `h = gdp * 20` | GDP → 高度（米） |
| `position = h/2` | 柱子底部贴地 |
| `label = h + 1000` | 标签浮在柱子顶 |

### Primitive Instancing（优化版）

```js
const instances = data.map(item => {
    const h = item.gdp * 20;
    const w = 80000;
    const center = Cesium.Cartesian3.fromDegrees(item.lon, item.lat, h / 2);
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center);
    // ↑ 让柱子在地球上"站直"

    return new Cesium.GeometryInstance({
        geometry: new Cesium.BoxGeometry({
            vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
            minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),  // 局部坐标
            maximum: new Cesium.Cartesian3(w/2, w/2, h/2),     // 用 new 不是 fromDegrees！
        }),
        modelMatrix: transform,  // 位置
    });
});

// 所有柱子 1 次提交 GPU（34 个 draw call → 1 个）
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,
    appearance: new Cesium.PerInstanceColorAppearance({
        closed: true,
        translucent: false,
    }),
}));

// 标签还得用 Entity（Primitive 不支持文字）
data.forEach(item => {
    const h = item.gdp * 20;
    viewer.entities.add({
        position: Cartesian3.fromDegrees(item.lon, item.lat, h + 1000),
        label: { text: item.name + '\n' + item.gdp + '亿', ... }
    });
});
```

### 学到了什么
- **Box 柱子**：`entities.add({ box: { dimensions } })` 创建 3D 柱子
- **柱子贴地**：`position` 高度 = `h/2`，底部贴地面
- **多行文本**：`\n` 在 label 里换行
- **Primitive Instancing**：`GeometryInstance` + `Primitive` 合并 draw call
- **注意**：高海拔城市（拉萨 3650m）柱子会埋地里

---

## 性能优化

```js
// ⭐ 最关键的优化：不动时不渲染
viewer.scene.requestRenderMode = true;

// 关特效（不影响数据展示）
viewer.scene.globe.enableLighting = false;   // 关光照
viewer.scene.fog.enabled = false;            // 关雾
viewer.scene.fxaa = false;                   // 关抗锯齿
viewer.scene.highDynamicRange = false;       // 关 HDR

// 看 FPS
viewer.scene.debugShowFramesPerSecond = true;
```

---

## 常见错误

```js
// ❌ fromDegrees 不能用来算 Box 尺寸
minimum: Cesium.Cartesian3.fromDegrees(-w/2, -w/2, -h/2)

// ✅ 要用 new Cartesian3
minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2)

// ❌ forEach 收集不了 GeometryInstance（要用 map）
data.forEach(item => { return new GeometryInstance({...}) })  // 丢了

// ✅ map 返回数组
const arr = data.map(item => { return new GeometryInstance({...}) })

// ❌ 柱子悬空
fromDegrees(lon, lat, h)   // 高度 = h

// ✅ 柱子贴地
fromDegrees(lon, lat, h/2) // 高度 = h/2
```

---

## 注意

- `cesium.com` **主站**（ion 控制台、文档）在国内访问慢/可能被墙
- 但 `cesium.com/downloads/cesiumjs/...` 的 CDN 地址在国内可以正常访问，**不需要换源**
- 如果遇到 Cesium 加载 render 失败，先检查网络和 token

## API Key

```js
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();
```

> 必须用 `http-server` 启动（`file://` 不支持 fetch）。

## 启动

```bash
cd 项目目录
npx http-server -p 8080 -c-1
# 浏览器打开 http://localhost:8080
```
