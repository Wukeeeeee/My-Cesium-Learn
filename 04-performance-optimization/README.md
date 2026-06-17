# 04 性能优化实验

## 背景

`03-gdp-visualization` 用 Entity Box 画 3D 柱状图，在集成显卡（Iris Xe）上拖拽卡顿（~15-20 FPS）。

本文件夹收集了不同优化方案，对比效果和性能。

## 优化方案

| 文件 | 方案 | 原理 |
|------|------|------|
| `00-original-entity-box.html` | Entity Box（原版） | 34 个 Box × 34 个 draw call |
| `01-billboard.html` | Billboard 假柱子 | Canvas 画红色柱子图片贴上去，零 3D 开销 |
| `02-points.html` | Point 散点图 | 用点大小表示 GDP，最轻量 |
| `03-instancing.html` | Primitive Instancing | 34 个 Box 合并成 1 个 draw call |
| `99-ultimate-optimization.html` | 终极优化 | 特效全关 + requestRenderMode |

## 结论

**卡顿的根源不是 Entity Box 的 draw call 数量，而是 Iris Xe 本身跑 Cesium 的渲染管线就吃力。**

| 方案 | FPS（拖拽时） | 说明 |
|------|-------------|------|
| 原版 Entity Box | 15-20 | 卡 |
| Billboard / Point / Instancing | 20-30 | 略好但有限 |
| 终极优化（requestRenderMode + 关特效） | **100+** ✅ | 流畅 |

**关键发现：** 真正管用的不是换渲染方案，而是 `requestRenderMode: true`——不动时不渲染，GPU 资源集中在拖拽那一帧上，反而更跟手。

## 终极优化清单（来自 99-ultimate-optimization.html）

### Viewer 选项

```js
const viewer = new Cesium.Viewer('cesiumcontainer', {
    terrain: undefined,           // 关地形（国内慢+吃性能）
    skyAtmosphere: false,         // 关大气
    skyBox: false,                // 关天空盒
    sceneModePicker: false,       // 关模式切换按钮
    baseLayerPicker: false,       // 关底图选择器
    navigationHelpButton: false,  // 关帮助按钮
    animation: false,             // 关动画控件
    timeline: false,              // 关时间轴
    fullscreenButton: false,      // 关全屏按钮
    infoBox: false,               // 关信息框
    selectionIndicator: false,    // 关选择指示器
});
```

### 渲染特效（不影响画质太多，但省 GPU）

```js
viewer.scene.globe.enableLighting = false;           // 关光照
viewer.scene.globe.showGroundAtmosphere = false;     // 关地面大气
viewer.scene.globe.showWaterEffect = false;           // 关水面效果
viewer.scene.fog.enabled = false;                     // 关雾
viewer.scene.fxaa = false;                            // 关抗锯齿
viewer.scene.highDynamicRange = false;                // 关 HDR
```

### 终极必杀技（不降画质，只降计算量）

```js
viewer.scene.requestRenderMode = true;                // 不动时不渲染
viewer.scene.maximumRenderTimeChange = Infinity;      // 不主动刷新
```

### 降画质换性能（不推荐，除非真带不动）

```js
viewer.scene.globe.maximumScreenSpaceError = 64;     // 默认16，越大瓦片越糊但越快
viewer.resolutionScale = 0.5;                        // 半分辨率渲染
```

## 跑起来

```bash
cd 04-performance-optimization
npx http-server -p 8080 -c-1
```

打开 `http://localhost:8080`，一个一个点开看左上角 FPS。

## FPS 查看方式

```js
viewer.scene.debugShowFramesPerSecond = true;
```

左上角会显示 `FPS: xx`，拖拽地球时观察变化。
