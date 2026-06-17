# Cesium Learning Projects

## 01-hello-cesium — 第一个 3D 地球

**项目位置：** `E:/my_repo/MyCesium/01-hello-cesium/`
**文件：** `01-hello-globe.html`（旧金山+3D建筑）、`02-shenzhen-practice.html`（深圳）

```js
// 读 API Key
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();

// 创建 3D 地球（viewer 就是地球，所有操作都通过它）
const viewer = new Cesium.Viewer('cesiumcontainer');

// 飞到中国上空
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(110, 35, 8000000),
    // fromDegrees(经度, 纬度, 高度米)
    // 800万米 ≈ 看到整个中国
});
```

**注意：** Cesium 区分大小写，`flyTo` 不能写成 `flyto`。必须用 http-server 启动，不支持双击打开。

---

## 02-province-markers — 批量标注省会

**项目位置：** `E:/my_repo/MyCesium/02-province-markers/`
**文件：** `01-load-cities.html`（基础）、`02-label-distance-limit.html`（防穿透）、`03-distance-display.html`（距离控制）

```js
// JSON 数据格式：[ { "name":"广州", "lon":113.27, "lat":23.13 }, ... ]

// 读数据
const res = await fetch('china-cities.json');
const data = await res.json();

// forEach = 对每个城市依次执行 {} 里的操作
data.forEach(item => {
    viewer.entities.add({
        // 放哪：经度、纬度、高度
        position: Cesium.Cartesian3.fromDegrees(item.lon, item.lat, 0),

        point: {                             // 画红点
            pixelSize: 10,                   // 点大小
            color: Cesium.Color.RED,
        },
        label: {                             // 加文字
            text: item.name,                 // 城市名
            font: '20px sans-serif',         // 字体大小
            color: Cesium.Color.WHITE,
            verticalOrigin: Cesium.VerticalOrigin.TOP,  // 点在文字下方
            pixelOffset: new Cesium.Cartesian2(0, 10),  // 文字往下移10像素
            disableDepthTestDistance: 0,     // 不穿透地球背面
            distanceDisplayCondition:
                new Cesium.DistanceDisplayCondition(0, 500000),
                // 相机距离 0~50万米 才显示文字
        },
    });
});
```

---

## 03-gdp-visualization — GDP 柱状图

**项目位置：** `E:/my_repo/MyCesium/03-gdp-visualization/`
**文件：** `01-gdp-chart.html`（带地形）、`02-gdp-simple.html`（无地形）、`03-instancing.html`（优化版）、`test.html`（带注释练习版）

### Entity Box（基础版）

```js
const h = gdp * 20;         // GDP → 柱子高度（米）
const w = 80000;            // 柱子粗细

viewer.entities.add({
    position: Cartesian3.fromDegrees(lon, lat, h / 2),  // h/2 = 贴地
    box: {
        dimensions: new Cartesian3(w, w, h),   // (宽, 深, 高)
        material: Cesium.Color.RED,
    },
});

// 柱顶标签
viewer.entities.add({
    position: Cartesian3.fromDegrees(lon, lat, h + 1000),
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
| `position = h/2` | 柱子贴地 |
| `label = h + 1000` | 标签浮在柱顶 |

### Primitive Instancing（优化版）

```js
// .map 收集 GeometryInstance（.forEach 不收集，不行）
const instances = data.map(item => {
    const h = item.gdp * 20, w = 80000;
    const center = Cesium.Cartesian3.fromDegrees(item.lon, item.lat, h / 2);
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center);
    // ↑ 让柱子在地球上站直（不同城市"上"方向不同）

    return new Cesium.GeometryInstance({
        geometry: new Cesium.BoxGeometry({
            vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
            // new 不是 fromDegrees！
            minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),
            maximum: new Cesium.Cartesian3(w/2, w/2, h/2),
        }),
        modelMatrix: transform,  // 位置
    });
});

// 所有柱子 1 次提交 GPU
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,
    appearance: new Cesium.PerInstanceColorAppearance({
        closed: true, translucent: false,
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

---

## ⚡ 性能优化

```js
// ⭐ 不动时不渲染（最关键）
viewer.scene.requestRenderMode = true;

// 关特效省 GPU
viewer.scene.globe.enableLighting = false;    // 关光照
viewer.scene.fog.enabled = false;             // 关雾
viewer.scene.fxaa = false;                    // 关抗锯齿
viewer.scene.highDynamicRange = false;        // 关 HDR

viewer.scene.debugShowFramesPerSecond = true; // 看 FPS
```

---

## ❌ 易错

```js
// fromDegrees 算经纬度，不算 Box 尺寸 ❌
minimum: Cesium.Cartesian3.fromDegrees(-w/2, -w/2, -h/2)
// 用 new Cartesian3  ✅
minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2)

// forEach 不收集 return ❌  map 收集 ✅
arr.forEach(i => { return xx })  // 丢了
arr.map(i => { return xx })      // 收集成新数组

// 柱子贴地：position 高度 = h/2 ✅  高度 = h ❌（悬空）
```

## ▶ 启动

```bash
cd 项目目录
npx http-server -p 8080 -c-1
# 浏览器 → http://localhost:8080
```
