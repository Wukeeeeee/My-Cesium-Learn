# 🛰️ My Cesium Learn

CesiumJS 3D 地球学习项目，从零开始一步步搭建。

---

## 📂 项目列表

### 01-create-globe — 创建3D地球
| # | 文件 | 内容 |
|:-|:-----|:------|
| 1 | `01-first-globe.html` | 初始化Cesium Viewer，创建第一个3D地球，飞往旧金山 |
| 2 | `02-fly-to-location.html` | 飞往深圳，练习相机角度控制（heading + pitch） |

### 02-add-markers — 添加标记
| # | 文件 | 内容 |
|:-|:-----|:------|
| 1 | `01-add-markers.html` | 从JSON读取数据，批量添加标记点+文字标签 |
| 2 | `02-label-fix.html` | 解决标签穿透地球背面问题（`disableDepthTestDistance`） |
| 3 | `03-label-distance.html` | 相机距离控制标签显隐（`distanceDisplayCondition`） |

### 03-make-chart — 制作图表
| # | 文件 | 内容 |
|:-|:-----|:------|
| 1 | `01-chart-entity.html` | 用Entity Box绘制GDP柱状图 |
| 2 | `02-chart-simple.html` | 简化版，柱子贴地（`position = h/2`） |
| 3 | `03-chart-fast.html` | 🔥 Primitive Instancing 性能优化（34根柱子1次draw call） |
| 4 | `04-test.html` | 练习文件 |

### 04-load-geojson — GeoJSON 数据加载
| # | 文件 | 内容 |
|:-|:-----|:------|
| 1 | `01-load-province.html` | 加载中国省界GeoJSON + 南海九段线 |
| 2 | `china-province.geojson` | 中国34个省级行政区边界数据 |
| 3 | `southsea.geojson` | 南海九段线数据 |

---

## 🚀 启动方式

```bash
npx http-server -p 8080 -c-1
# 浏览器打开 http://localhost:8080
```

> ⚠️ 必须用 http-server，不支持直接双击打开（Cesium 跨域限制）

## 📚 笔记

见 [notes/cesium/](notes/cesium/)

---

*CesiumJS 版本: 1.137 · 学习时间: 2026.06*
