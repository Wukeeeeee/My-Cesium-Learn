# Primitive 定位原理 — 为什么它跟 Entity 不一样

> Entity 加 Box 只需要 `position`，Primitive Box 却要 `modelMatrix` + `eastNorthUpToFixedFrame`？这篇讲清楚。

---

## 核心区别：谁来做坐标变换

### Entity 模式（你熟悉的）

```js
viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(lon, lat, h / 2),  // ← 直接给经纬度
    box: { dimensions: new Cesium.Cartesian3(w, w, h) },
});
```

**Cesium 帮你做了：** 把经纬度转成三维坐标，再把 Box 从局部坐标移到那个位置——你不需要管中间过程。

### Primitive 模式（更底层）

```js
new Cesium.GeometryInstance({
    geometry: new Cesium.BoxGeometry({
        minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),
        maximum: new Cesium.Cartesian3(w/2, w/2, h/2),
    }),
    modelMatrix: transform,  // ← 你要自己告诉 GPU "放哪"
})
```

**你得自己算：** 告诉 GPU 这个 Box 的顶点在局部坐标里的位置，再提供一个矩阵告诉它"把这些顶点移到地球上的哪个位置"。

---

## 两个坐标系

### ① 局部坐标（Local Space）

BoxGeometry 是在**局部坐标**里定义的：

```
        Y ↑
          │
          │     ┌──────┐ (w/2, w/2, h/2)
          │    /      /│
          │   /      / │
          │  └──────┘  │
          │   │      │ /
          │   │      │/
(-w/2, -w/2, -h/2) ───────→ X
         /
        /
```

- 原点 (0,0,0) 在 **Box 的中心**
- `minimum` 和 `maximum` 是**相对于中心**的两个对角顶点
- Box 的 **"上" = Y 轴正方向**

在这个坐标系里，Box 是直立的、中心在原点的——但它还没被放到地球上。

### ② 地球坐标（World Space / FixedFrame）

地球坐标是 Cesium 的全局坐标系，原点在地球中心。

- X 轴指向 (0°经度, 0°纬度)
- Y 轴指向 (90°E, 0°纬度)
- Z 轴指向北极

你没法直接在这个坐标系里说"把盒子放广州"，因为**这个坐标系不是经纬度**。

---

## 关键桥梁：`fromDegrees`

```js
const center = Cesium.Cartesian3.fromDegrees(lon, lat, h / 2)
```

`fromDegrees` 把**经纬度+高度**转成**地球坐标里的一个点**。

```
输入: 113.27°E, 23.13°N, 高度 500000
  ↓
输出: (x: 123456, y: 789012, z: 345678)  ← 地球坐标里的三维位置
```

但这个只是一个**点**，不是完整的"朝向"信息。

---

## 关键桥梁：`eastNorthUpToFixedFrame`

```js
const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center)
```

### 它在解决什么问题？

想象你在广州地面上画了一个箭头指向正北。如果你在北京地面也画一个箭头指向正北——这两个箭头的方向在**三维空间里是不同的**，因为地球是球形的。

```
       广州的"上"          北京的"上"
          ↑                    ↑
          │                    │
          │   ● 广州           │   ● 北京
          │                    │
```

`eastNorthUpToFixedFrame(center)` 生成一个 **4×4 矩阵**，它定义了：

1. **哪个方向是"东"**（在 `center` 那个位置的地球切线上）
2. **哪个方向是"北"**（在 `center` 那个位置指向北极的方向）
3. **哪个方向是"上"**（从 `center` 指向地球外的方向）

然后把这个矩阵作为 `modelMatrix` 传给 GPU，告诉它：

> "把 BoxGeometry 的 Y 轴 → 对准 '上' 方向"
> "把 BoxGeometry 的 X 轴 → 对准 '东' 方向"
> "把 BoxGeometry 的 Z 轴 → 对准 '北' 方向"
> "然后把整个 Box 移到 center 那个位置"

---

## modelMatrix 到底在干什么

```js
// 局部坐标里的一个顶点 (w/2, w/2, h/2)
// → 经过 modelMatrix 变换
// → 变成地球坐标里的实际位置

局部顶点 × modelMatrix = 地球坐标上的顶点
```

`modelMatrix` 就是一个**转换公式**，把 Box 的 24 个顶点从"以原点为中心"变成"以广州为中心、底部贴地"。

---

## 完整流程拆解

```
                    局部坐标                      地球坐标
                    ─────────                    ─────────

BoxGeometry         顶点们                         
(min/max)          (-w/2, -w/2, -h/2)            
                   (w/2, w/2, h/2)              
                   (w/2, -w/2, -h/2)             
                   ... 共 24 个顶点                
                       │                            
                       │  ↓ 乘以 modelMatrix         
                       │                            
                       ▼                            
                  ┌──────────────────┐              
                  │  eastNorthUpTo   │  ← 这个矩阵把"局部 Y 轴朝上"
                  │  FixedFrame      │    映射到"地球表面朝上"
                  │  (center)        │    并把原点移到 center
                  └──────────────────┘              
                       │                            
                       ▼                            
                 地球坐标上的 24 个顶点               
                 柱子立在了广州 ✅                   
```

---

## 常见错误

### 错误 1：把 `fromDegrees` 用在 Box 尺寸上

```js
// ❌ 错！fromDegrees 是算经纬度的，不是算盒子尺寸的
minimum: Cesium.Cartesian3.fromDegrees(-w/2, -w/2, -h/2),
```

`fromDegrees` 输出的是地球坐标上的**位置**，取值范围是几百万到几千万米。而 Box 尺寸只有几万米。用了 `fromDegrees` 盒子会飞到奇怪的地方或者根本看不见。

```js
// ✅ 正确：用 new Cartesian3
minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),
```

### 错误 2：忘记 `modelMatrix`

```js
// ❌ 没给 modelMatrix，所有 Box 堆在地球中心
new Cesium.GeometryInstance({
    geometry: new Cesium.BoxGeometry({...}),
    // 没有 modelMatrix！
})
```

不给 `modelMatrix`，Box 就在局部坐标的原点——地球中心，你看不到。

### 错误 3：不贴地（position 高度设错了）

```js
// ❌ 柱子浮在半空
const center = fromDegrees(lon, lat, h)   // 高度 = h
// Box 的中心在 h，底部在 h/2，悬空了

// ✅ 柱子贴地
const center = fromDegrees(lon, lat, h/2) // 高度 = h/2
// Box 的中心在 h/2，底部在地面
```

---

## 一张图总结

```
Entity:
  entities.add({ position: fromDegrees(lon, lat, h/2), box: {...} })
  └── Cesium 自动算 modelMatrix，你不需要管

Primitive:
  const center = fromDegrees(lon, lat, h/2)
  const transform = eastNorthUpToFixedFrame(center)

  GeometryInstance({
      geometry: BoxGeometry({minimum, maximum}),  ← 局部坐标的形状
      modelMatrix: transform,                      ← 自己算位置
  })
  └── 你手动算 modelMatrix，换来 34→1 个 draw call 的性能提升
```

**Entity 省事，Primitive 省性能。** 懂了这个，你就理解为什么 Instancing 快了。

---

## 实战代码逐段拆解（test copy.html）

### 完整代码

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <script src="https://cesium.com/downloads/cesiumjs/releases/1.137/Build/Cesium/Cesium.js"></script>
    <link href="https://cesium.com/downloads/cesiumjs/releases/1.137/Build/Cesium/Widgets/widgets.css" rel="stylesheet">
    <title>GDP 柱状图</title>
</head>
<body>
    <div id="cesiumcontainer"></div>
    <script type="module">
        // ====== ① API Key ======
        const res = await fetch('apikey.txt');
        Cesium.Ion.defaultAccessToken = (await res.text()).trim();

        // ====== ② 创建地球 ======
        const viewer = new Cesium.Viewer('cesiumcontainer', {
            skyAtmosphere: false,
            sceneModePicker: false,
            baseLayerPicker: false,
        })

        // ====== ③ 飞中国 ======
        viewer.camera.flyTo({
            destination: Cesium.Cartesian3.fromDegrees(110, 35, 8000000),
        });

        // ====== ④ 关特效 ======
        viewer.scene.globe.enableLighting = false;
        viewer.scene.fog.enabled = false;
        viewer.scene.fxaa = false;

        // ====== ⑤ 读数据 ======
        const gdpResponse = await fetch('china-gdp.json')
        const gdp = await gdpResponse.json()

        // ====== ⑥ 造柱子（Primitive Instancing） ======
        const instances = gdp.map(cities => {
            const h = cities.gdp * 20   // GDP → 柱子高度（米）
            const w = 80000             // 柱子粗细（固定）

            // ⑥-a 柱子中心位置（经纬度 + 半高 → 贴地）
            const center = Cesium.Cartesian3.fromDegrees(cities.lon, cities.lat, h / 2)

            // ⑥-b 算旋转矩阵（让柱子"站"在地球表面）
            const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center)

            // ⑥-c 打包工单：形状 + 位置
            return new Cesium.GeometryInstance({
                geometry: new Cesium.BoxGeometry({
                    vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
                    minimum: new Cesium.Cartesian3(-w / 2, -w / 2, -h / 2),
                    maximum: new Cesium.Cartesian3(w / 2, w / 2, h / 2),
                }),
                modelMatrix: transform,
            })
        })

        // ====== ⑦ 所有柱子一次性提交 GPU ======
        viewer.scene.primitives.add(new Cesium.Primitive({
            geometryInstances: instances,
            appearance: new Cesium.PerInstanceColorAppearance({
                closed: true,
                translucent: false
            })
        }))

        // ====== ⑧ 标签（Entity，因为 Primitive 不支持文字） ======
        gdp.forEach(cities => {
            const h = cities.gdp * 20
            viewer.entities.add({
                position: Cesium.Cartesian3.fromDegrees(cities.lon, cities.lat, h + 1000),
                label: {
                    text: cities.name + '\n' + cities.gdp + '亿',
                    font: '20px sans-serif',
                    color: Cesium.Color.WHITE,
                    verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
                    disableDepthTestDistance: 0,
                    distanceDisplayCondition: new Cesium.DistanceDisplayCondition(0, 10000000),
                },
            })
        })

        // ====== ⑨ 显示 FPS ======
        viewer.scene.debugShowFramesPerSecond = true
    </script>
</body>
</html>
```

### 各段详解

#### ⑥ `gdp.map(cities => {...})` — 造柱子

| 行 | 代码 | 在干什么 | 容易忘的 |
|----|------|---------|---------|
| `const h = cities.gdp * 20` | GDP 数值 × 20 → 柱子高度（米） | 数值映射为视觉高度 |
| `const w = 80000` | 柱子粗细 80 公里 | 固定值，所有柱子一样粗 |
| `Cesium.Cartesian3.fromDegrees(lon, lat, h/2)` | 经纬度 → 三维坐标 | **第三个参数是 h/2，不是 h**，这样才能贴地 |
| `Cesium.Transforms.eastNorthUpToFixedFrame(center)` | 算旋转矩阵 | 让柱子"站直"在地球表面，不同城市方向不同 |
| `new Cesium.Cartesian3(-w/2, -w/2, -h/2)` | Box 的最小角（局部坐标） | **用 `new` 不是 `fromDegrees`** |
| `new Cesium.Cartesian3(w/2, w/2, h/2)` | Box 的最大角（局部坐标） | 范围从 `(-w/2, -w/2, -h/2)` 到 `(w/2, w/2, h/2)` |

#### 为什么用 `.map()` 不用 `.forEach()`？

```js
// ❌ forEach：只做事，不返回结果
gdp.forEach(cities => {
    return new Cesium.GeometryInstance({...})  // ← return 了没人接，丢了
})

// ✅ map：每个城市一个 GeometryInstance，收集成数组
const instances = gdp.map(cities => {
    return new Cesium.GeometryInstance({...})  // ← 自动收集到 instances[] 里
})
```

#### ⑦ `viewer.scene.primitives.add()` — 提交 GPU

```js
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,            // ← 34 张工单打包
    appearance: new Cesium.PerInstanceColorAppearance({
        closed: true,        // 封闭实体（画背面）
        translucent: false   // 不透明（不计算透明度，更快）
    })
}))
```

- `geometryInstances`：之前 map 出来的 34 个 GeometryInstance 数组
- `closed: true`：柱子是实心的，背面也要渲染
- `translucent: false`：不透明，不需要混合计算

#### ⑧ 标签为什么单独 forEach？

**Primitive 不支持文字**，所以标签还得用 Entity API，单独遍历一次：

```js
gdp.forEach(cities => {
    viewer.entities.add({
        position: Cesium.Cartesian3.fromDegrees(lon, lat, h + 1000), // ← 柱子顶 + 1000米
        label: { text, font, color, ... }
    })
})
```

`h + 1000`：标签悬浮在柱子顶部往上 1000 米，不会埋在柱子里。

---

## test.html 常见错误速查

| 你容易犯的错 | 后果 | 正确写法 |
|------------|------|---------|
| `fromDegrees` 算 Box 尺寸 | 盒子跑到奇怪位置 | `new Cesium.Cartesian3(w, w, h)` |
| `.forEach` 返回 GeometryInstance | 没人收集，柱子在空气中 | `.map` 收集数组 |
| `appearance: { color: RED }` | 报错 | `new PerInstanceColorAppearance({...})` |
| `fromDegrees(lon, lat, h)` | 柱子悬空 | `fromDegrees(lon, lat, h/2)` 贴地 |
| 忘记 `modelMatrix` | 看不到柱子 | 加上 `modelMatrix: transform` |
| 柱子和标签写在一个循环里 | 结构乱 | 先 map 柱子，再 forEach 标签 |
