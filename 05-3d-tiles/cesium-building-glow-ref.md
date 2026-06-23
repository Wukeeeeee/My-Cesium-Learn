# 建筑泛光效果参考

> 文章：[Cesium 打造科技感建筑泛光效果](https://juejin.cn/post/7277802797473644578)
> 用自定义着色器给 3D Tiles 白膜加发光效果

## 核心思路

通过 Cesium 的 **`CustomShader`** 修改 3D Tiles 的片段着色器，给建筑边缘/整体加发光效果，不需要依赖材质贴图。

## 关键 API

```js
const customShader = new Cesium.CustomShader({
  fragmentShaderText: `
    // 自定义着色器代码
    void fragmentMain(FragmentInput fsInput, inout czm_modelMaterial material) {
      // material.diffuse = 漫反射颜色
      // material.emissive = 自发光颜色（类似发光）
      // material.alpha = 透明度
    }
  `,
  lightingModel: Cesium.LightingModel.PBR,  // 或 UNLIT
});
```

## 应用着色器

```js
tileset.customShader = customShader;
```

## 踩坑提醒

- `emissive` 在 `Cesium3DTileStyle` 里不能用，但在 **`CustomShader`** 里可以用 `material.emissive`
- `CustomShader` 需要 Cesium 1.84+
- 配合关太阳 + 夜景背景效果更好

## 相关阅读

- [Cesium CustomShader 官方文档](https://cesium.com/learn/cesiumjs/ref-doc/CustomShader.html)
