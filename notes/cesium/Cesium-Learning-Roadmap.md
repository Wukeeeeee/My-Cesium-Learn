# Cesium 学习路线

## 项目位置

`E:/my_repo/MyCesium/learn-cesium/`

```
learn-cesium/
├── index.html           ← ★ 最简单的 3D 地球（从这里开始）
└── add-entities.html    ← 添加标记点
```

## 方案：纯 HTML + CDN

不需要 Vue / Vite / npm，就是一个普通 HTML 文件。

```html
<!-- 只需要这两行 -->
<link href="https://cdn.jsdelivr.net/npm/cesium@1.142/Build/Cesium/Widgets/widgets.css" rel="stylesheet" />
<script src="https://cdn.jsdelivr.net/npm/cesium@1.142/Build/Cesium/Cesium.js"></script>

<!-- 一行创建 3D 地球 -->
<script>
  const viewer = new Cesium.Viewer("cesiumContainer");
</script>
```

## 本地运行

```bash
cd E:/my_repo/MyCesium/learn-cesium
npx http-server -p 8080 -c-1
```

浏览器打开 `http://localhost:8080`

## Viewer 快速入门

```js
const viewer = new Cesium.Viewer("容器id");

viewer.entities.add({ ... });     // 加点、线、面
viewer.flyTo(entities);           // 飞过去看
```

## 学习顺序

1. ✅ 01-hello-cesium — 跑通最简单的 3D 地球
2. ✅ 02-province-markers — 添加实体（点、标签、距离控制）
3. ✅ 03-gdp-visualization — Box 柱状图 + Primitive Instancing 写法
4. ▶ 接下来：glTF 3D 模型、动态轨迹、3D Tiles
