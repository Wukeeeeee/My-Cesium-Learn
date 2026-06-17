# Cesium Learning Projects

## 01-hello-cesium — 第一个 3D 地球

**位置：** `E:/my_repo/MyCesium/01-hello-cesium/`
**文件：** `01-hello-globe.html`、`02-shenzhen-practice.html`

```js
// 读 API Key（fetch 读文件，.text() 转文本，.trim() 去换行）
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();

// 创建 3D 地球，'cesiumcontainer' 对应 div 的 id
const viewer = new Cesium.Viewer('cesiumcontainer');

// 飞到中国上空
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(110, 35, 8000000),
    // fromDegrees(经度, 纬度, 高度米)
    // 800万米 ≈ 看整个中国
});
```

**踩坑：** `flyTo` 的 T 要大写、id 要对应、必须 http-server 启动。

---

## 02-province-markers — 批量标注省会

**位置：** `E:/my_repo/MyCesium/02-province-markers/`
**文件：** `01-load-cities.html`、`02-label-distance-limit.html`、`03-distance-display.html`

```js
// JSON 数据格式：[ { "name":"广州", "lon":113.27, "lat":23.13 }, ... ]

// 读 JSON 文件，res.json() 把内容解析成数组
const res = await fetch('china-cities.json');
const data = await res.json();

// forEach = 对数组里每个城市执行一次 { }
data.forEach(item => {
    viewer.entities.add({

        // 位置：经度、纬度、高度（0=地面）
        position: Cesium.Cartesian3.fromDegrees(item.lon, item.lat, 0),

        // 画一个红点
        point: {
            pixelSize: 10,           // 点大小（像素）
            color: Cesium.Color.RED,
        },

        // 加文字标签
        label: {
            text: item.name,                            // 显示城市名
            font: '20px sans-serif',                     // 字体
            color: Cesium.Color.WHITE,
            verticalOrigin: Cesium.VerticalOrigin.TOP,  // 点在文字下方
            pixelOffset: new Cesium.Cartesian2(0, 10),  // 下移10像素不重叠
            disableDepthTestDistance: 0,                // 不穿透地球背面
            distanceDisplayCondition:
                new Cesium.DistanceDisplayCondition(0, 500000),
                // 相机距离 0~50万米 才显示，远了隐藏（防文字挤一起）
        },
    });
});
```

---

## 03-gdp-visualization — GDP 柱状图

**位置：** `E:/my_repo/MyCesium/03-gdp-visualization/`
**文件：** `01-gdp-chart.html`（带地形）、`02-gdp-simple.html`（无地形）、`03-instancing.html`、`test.html`

### Entity Box 画柱子

```js
const h = cities.gdp * 20;   // GDP 数值 ×20 → 柱子高度（米）
const w = 80000;              // 柱子粗细，固定 8 万米

viewer.entities.add({
    // position 高度 = h/2 → 柱子底部贴地、中心在半高
    position: Cesium.Cartesian3.fromDegrees(cities.lon, cities.lat, h / 2),
    box: {
        dimensions: new Cesium.Cartesian3(w, w, h),  // (宽, 深, 高)
        material: Cesium.Color.RED,
    },
});

// 柱顶标签（单独加一个 entity，放在柱子顶上方）
viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(cities.lon, cities.lat, h + 1000),
    label: {
        text: cities.name + '\n' + cities.gdp + '亿',  // \n 换行
        font: '20px sans-serif',
        color: Cesium.Color.WHITE,
        verticalOrigin: Cesium.VerticalOrigin.BOTTOM,  // 文字往上长
        disableDepthTestDistance: 0,                    // 不被柱子挡住
        distanceDisplayCondition:
            new Cesium.DistanceDisplayCondition(0, 10000000),
    }
});
```

| 公式 | 效果 |
|------|------|
| `h = gdp * 20` | GDP 数值转成柱子高度 |
| `position = h/2` | 柱子中心在半高→底部贴地 |
| `label = h + 1000` | 标签浮在柱子顶部 |

### Primitive Instancing（性能优化版）

**为什么用这个？** Entity 每根柱子都单独跟 GPU 通信一次（34次=34个 draw call），卡。Instancing 把所有柱子打包成一次提交（1个 draw call），快。

```js
// .map 遍历数组并收集返回值（.forEach 不收集，不能用）
const instances = data.map(item => {
    const h = item.gdp * 20;
    const w = 80000;

    // ① 算柱子中心在地球上的位置（经纬度 + 半高）
    const center = Cesium.Cartesian3.fromDegrees(item.lon, item.lat, h / 2);

    // ② 算旋转矩阵 — 让柱子在地球表面"站直"
    // 地球是球形的，广州的"上"和北京的"上"方向不同
    // 这个函数算出柱子应该朝哪边才能垂直于地面
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center);

    // ③ 返回一张"工单"（形状 + 位置）
    return new Cesium.GeometryInstance({
        // 形状：在局部坐标（原点）里定义一个长方体
        geometry: new Cesium.BoxGeometry({
            vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
            // minimum/maximum：两个对角顶点，范围从 (-w/2, -h/2) 到 (w/2, h/2)
            // 注意：用 new Cartesian3，不是 fromDegrees！
            minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),
            maximum: new Cesium.Cartesian3(w/2, w/2, h/2),
        }),
        modelMatrix: transform,  // 位置：告诉 GPU 把这个柱子放哪、朝哪
    });
});

// ④ 所有工单一次性提交 GPU（34个柱子 → 1次画完）
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,
    appearance: new Cesium.PerInstanceColorAppearance({
        closed: true,          // 封闭实体（画出柱子的背面）
        translucent: false,    // 不透明，省去透明度计算
    }),
}));

// ⑤ 标签还是用 Entity（Primitive 不支持文字）
data.forEach(item => {
    const h = item.gdp * 20;
    viewer.entities.add({
        position: Cesium.Cartesian3.fromDegrees(item.lon, item.lat, h + 1000),
        label: {
            text: item.name + '\n' + item.gdp + '亿',
            font: '20px sans-serif',
            color: Cesium.Color.WHITE,
            verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
            disableDepthTestDistance: 0,
            distanceDisplayCondition:
                new Cesium.DistanceDisplayCondition(0, 10000000),
        },
    });
});
```

---

## 性能优化

```js
// 最关键的优化：不动时不渲染，拖拽反而更跟手
viewer.scene.requestRenderMode = true;

// 关特效（对数据展示没影响，但省 GPU）
viewer.scene.globe.enableLighting = false;    // 关光照
viewer.scene.fog.enabled = false;             // 关雾
viewer.scene.fxaa = false;                    // 关抗锯齿
viewer.scene.highDynamicRange = false;        // 关 HDR

// 左上角显示 FPS，拖拽时看卡不卡
viewer.scene.debugShowFramesPerSecond = true;
```

---

## 常见错误

```js
// 1. fromDegrees 不能算 Box 尺寸❌  要用 new Cartesian3 ✅
fromDegrees(-w/2, -w/2, -h/2)  // 错，飞到外太空
new Cartesian3(-w/2, -w/2, -h/2)  // 对

// 2. forEach 不收集 return ❌  .map 才能收集 ✅
arr.forEach(i => { return xx })  // 丢了，接不住
arr.map(i => { return xx })      // 收集成新数组

// 3. 柱子贴地：position 高度 = h/2 ✅   = h 就悬空 ❌
fromDegrees(lon, lat, h/2)  // 贴地
fromDegrees(lon, lat, h)    // 悬空

// 4. Primitive 的 appearance 不能用普通对象❌
appearance: { color: RED }           // 错
new PerInstanceColorAppearance({})   // 对
```

## 启动

```bash
cd 项目目录
npx http-server -p 8080 -c-1
# 浏览器 → http://localhost:8080
```
