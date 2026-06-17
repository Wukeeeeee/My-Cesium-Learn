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
