# 04 渲染方案对比

> 不同渲染方案的性能对比实验。真正能提升 FPS 的优化只有 `requestRenderMode`，其他方案只是换一种画法。

## 方案列表

| # | 文件 | 方案 | 说明 |
|---|------|------|------|
| 01 | `01-original-entity-box.html` | Entity Box | 原版 baseline，34 个独立 draw call |
| 02 | `02-billboard.html` | Billboard 贴图 | Canvas 画柱子，2D 贴图替代 3D |
| 03 | `03-points.html` | Point 散点图 | 用点大小表示数值，最轻量 |
| 04 | `04-instancing.html` | Primitive Instancing | 34 个 Box 合并为 1 个 draw call |
| **05** | **`05-ultimate.html`** | **requestRenderMode** | **真正有效的优化，100+ FPS** |
| 06 | `06-instancing-practice.html` | Instancing 练习 | 带详细注释的版本 |

## 实测结论（Iris Xe）

| 方案 | 拖拽 FPS | 是否算优化 |
|------|---------|-----------|
| Entity Box（原版） | 15-20 | — |
| Billboard / Point / Instancing | 20-30 | ❌ 不同画法，提升有限 |
| **requestRenderMode + 关特效** | **100+** | **✅ 真优化** |

## 关键代码（05-ultimate.html）

```js
// 不动时不渲染——最关键的一行
viewer.scene.requestRenderMode = true;
viewer.scene.maximumRenderTimeChange = Infinity;

// 关掉不必要的渲染特效
viewer.scene.globe.enableLighting = false;
viewer.scene.fog.enabled = false;
viewer.scene.fxaa = false;
viewer.scene.highDynamicRange = false;
```

## 跑起来

```bash
cd 04-rendering-comparison
npx http-server -p 8080 -c-1
```

逐个打开看左上角 FPS。
