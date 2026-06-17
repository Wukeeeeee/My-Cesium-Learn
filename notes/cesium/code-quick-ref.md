# Cesium 代码速查（手机版）

## 01 — 创建地球

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

## 02 — 加载数据 + 批量加点

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

## 03 — Box 柱状图

```js
// Entity Box（最简）
const h = gdp * 20;        // 数据 → 高度
const w = 80000;           // 柱子粗细

viewer.entities.add({
    position: Cartesian3.fromDegrees(lon, lat, h / 2),  // h/2 = 贴地
    box: {
        dimensions: new Cartesian3(w, w, h),   // (宽, 深, 高)
        material: Cesium.Color.RED,
    },
    // 柱顶标签（单独加）
    label: {
        text: name + '\n' + gdp + '亿',   // \n 换行
        font: '20px sans-serif',
        color: Cesium.Color.WHITE,
        verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
        disableDepthTestDistance: 0,
    }
});
```

## 03 — Primitive Instancing（优化版）

```js
const instances = data.map(item => {
    const h = item.gdp * 20;
    const w = 80000;

    const center = Cesium.Cartesian3.fromDegrees(item.lon, item.lat, h / 2);
    // 让柱子在地球上"站直"
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center);

    return new Cesium.GeometryInstance({
        geometry: new Cesium.BoxGeometry({
            vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
            minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),  // 局部坐标
            maximum: new Cesium.Cartesian3(w/2, w/2, h/2),     // 用 new 不是 fromDegrees！
        }),
        modelMatrix: transform,  // 位置
    });
});

// 一次性提交 GPU（所有柱子 1 个 draw call）
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

## 启动

```bash
cd 项目目录
npx http-server -p 8080 -c-1
# 浏览器打开 http://localhost:8080
```
