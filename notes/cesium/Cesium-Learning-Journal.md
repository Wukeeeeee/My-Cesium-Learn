# Cesium Learning Projects

## 01-create-globe — 创建 3D 地球

**位置：** `E:/my_repo/MyCesium/01-create-globe/`
**文件：** `01-first-globe.html`、`02-fly-to-location.html`

```js
// 读 API Key（fetch 读文件，.text() 转文本，.trim() 去换行）
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();

// 创建 3D 地球
const viewer = new Cesium.Viewer('cesiumcontainer');

// 飞到指定位置
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(113.27, 23.13, 400),
    // fromDegrees(经度, 纬度, 高度米)
    orientation: {
        heading: Cesium.Math.toRadians(0.0),   // 朝向
        pitch: Cesium.Math.toRadians(-30.0),   // 俯仰角
    }
});
```

**踩坑：** `flyTo` 的 T 要大写、id 要对应、必须 http-server 启动。

---

## 02-add-markers — 批量标注省会

**位置：** `E:/my_repo/MyCesium/02-add-markers/`
**文件：** `01-add-markers.html`、`02-label-fix.html`、`03-label-distance.html`

```js
// JSON 数据格式：[ { "name":"广州", "lon":113.27, "lat":23.13 }, ... ]

// 读取 JSON 文件
const res = await fetch('china-cities.json');
const cities = await res.json();

// forEach = 对数组里每个城市执行一次
cities.forEach(item => {
    viewer.entities.add({
        position: Cesium.Cartesian3.fromDegrees(item.lon, item.lat, 0),
        point: { pixelSize: 10, color: Cesium.Color.RED },
        label: {
            text: item.name,
            font: '20px sans-serif',
            color: Cesium.Color.WHITE,
            verticalOrigin: Cesium.VerticalOrigin.TOP,
            pixelOffset: new Cesium.Cartesian2(0, 10),
            disableDepthTestDistance: 0,    // 不穿透地球背面
            distanceDisplayCondition:
                new Cesium.DistanceDisplayCondition(0, 500000),
                // 相机距离 0~50万米 才显示
        },
    });
});
```

**踩坑：** 地形瓦片国内访问慢、港澳距离近标签重叠要用 `distanceDisplayCondition`

---

## 03-make-chart — GDP 柱状图

**位置：** `E:/my_repo/MyCesium/03-make-chart/`
**文件：** `01-chart-entity.html`（带地形）、`02-chart-simple.html`（无地形）、`03-chart-fast.html`（Instancing）、`04-test.html`

### Entity Box 画柱子

```js
const h = cities.gdp * 20;   // GDP ×20 → 柱子高度
const w = 80000;

viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(cities.lon, cities.lat, h / 2),
    box: {
        dimensions: new Cesium.Cartesian3(w, w, h),  // (宽, 深, 高)
        material: Cesium.Color.RED,
    },
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
    const center = Cesium.Cartesian3.fromDegrees(item.lon, item.lat, h / 2);
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center);

    return new Cesium.GeometryInstance({
        geometry: new Cesium.BoxGeometry({
            vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
            minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),
            maximum: new Cesium.Cartesian3(w/2, w/2, h/2),
        }),
        modelMatrix: transform,
    });
});

// 所有工单一次性提交 GPU（34个柱子 → 1次画完）
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,
    appearance: new Cesium.PerInstanceColorAppearance({
        closed: true,
        translucent: false,
    }),
}));
```

---

## 04-load-geojson — 加载 GeoJSON

**位置：** `E:/my_repo/MyCesium/04-load-geojson/`
**文件：** `01-load-province.html`、`china-province.geojson`、`southsea.geojson`

```js
// ① 加载省份边界
const res = await fetch('china-province.geojson');
const data = await res.json();
delete data.crs;  // 删除旧版CRS声明，否则Cesium报错

const provinceDs = await Cesium.GeoJsonDataSource.load(data, {
    stroke: Cesium.Color.YELLOW,          // 边界线颜色
    strokeWidth: 2,                        // 边界线粗细
    fill: Cesium.Color.WHITE.withAlpha(0), // 不填充
    clampToGround: false,                  // 不贴地
});
viewer.dataSources.add(provinceDs);

// ② 加载南海九段线（MultiLineString）
const southsea = await fetch('southsea.geojson');
const southseaData = await southsea.json();

const ssDs = await Cesium.GeoJsonDataSource.load(southseaData, {
    stroke: Cesium.Color.RED,
    strokeWidth: 3,
    clampToGround: false,
});
viewer.dataSources.add(ssDs);
```

**踩坑：**
- `GeoJsonDataSource` 不是 `GeojsonDataSource`（大小写敏感）
- 旧版geojson带 `crs` 字段会报错 → `delete data.crs`
- MultiLineString 没有填充，只有 `stroke` 有效

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

// 左上角显示 FPS
viewer.scene.debugShowFramesPerSecond = true;
```

---

## 常见错误

```js
// 1. fromDegrees 不能算 Box 尺寸，要用 new Cartesian3
fromDegrees(-w/2, -w/2, -h/2)  // 错，飞到外太空
new Cartesian3(-w/2, -w/2, -h/2)  // 对

// 2. forEach 不收集 return，.map 才能收集
arr.forEach(i => { return xx })  // 丢了，接不住
arr.map(i => { return xx })      // 收集成新数组

// 3. 柱子贴地：position 高度 = h/2，= h 就悬空
fromDegrees(lon, lat, h/2)  // 贴地
fromDegrees(lon, lat, h)    // 悬空

// 4. GeoJsonDataSource 不是 GeojsonDataSource
Cesium.GeojsonDataSource.load()   // 错，不存在
Cesium.GeoJsonDataSource.load()   // 对

// 5. 旧版 geojson 要删 crs
delete data.crs  // 否则报 Unknown crs name
```

## 启动

---

## 05-3d-tiles — OSM 建筑白膜 + 发光

**位置：** `E:/my_repo/MyCesium/05-3d-tiles/`
**文件：** `01-osm-buildings-nyc.html`、`02-osm-buildings-style-test.html`、`glow.html`

### 加载 OSM Buildings（纽约）

```js
const viewer = new Cesium.Viewer('cesiumContainer', {
    terrain: Cesium.Terrain.fromWorldTerrain(),
    //开启阴影
    shadow: true,
});

//记得加Camera
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(-74, 40.7, 5000000),
    orientation: {
        //俯仰角度
        pitch: Cesium.Math.toRadians(-30),
        //偏航角度
        heading: Cesium.Math.toRadians(0),
    }
})

async function loadTileset() {
    const tileset=await Cesium.createOsmBuildingsAsync();
    //添加到场景
    viewer.scene.primitives.add(tileset);
}
loadTileset();

//开启光照
viewer.scene.globe.enableLighting = true;
//添加太阳
viewer.scene.sun=new Cesium.Sun();
```

### Cesium3DTileStyle 测试（广州）

```js
const viewer = new Cesium.Viewer('cesiumContainer', {
    terrain: Cesium.Terrain.fromWorldTerrain(),
    //开启阴影
    shadows: true,
});

//记得加camera
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(113.27, 23.12, 50000),
    orientation: {
        //俯仰角度
        pitch: Cesium.Math.toRadians(-30),
        //偏航角度
        heading: Cesium.Math.toRadians(0),
    },
    //飞行时间
    duration: 1,
});

async function loadTileset() {
    const tileset=await Cesium.createOsmBuildingsAsync();
    //添加到场景
    //primitive是整体渲染
    viewer.scene.primitives.add(tileset),
    tileset.style=new Cesium.Cesium3DTileStyle({
        //颜色
        //OSM白膜没有颜色
        color: 'color("white")',
        //发光颜色
        emissive: 'color("orange")'
    });
}
loadTileset();

//开启光照
viewer.scene.globe.enableLighting = true;
//添加太阳
viewer.scene.sun=new Cesium.Sun();
```

### CustomShader 建筑发光（glow.html）

```js
// 夜景氛围：关太阳 + 深色背景 + 关光照
viewer.scene.sun = undefined;
viewer.scene.globe.enableLighting = false;
viewer.scene.backgroundColor = Cesium.Color.fromCssColorString('#0a0a1a');

// CustomShader 设置 emissive 实现真发光
const customShader = new Cesium.CustomShader({
    lightingModel: Cesium.LightingModel.UNLIT,
    fragmentShaderText: `
        void fragmentMain(FragmentInput fsInput, inout czm_modelMaterial material) {
            material.emissive = vec3(1.0, 0.6, 0.0);   // 橘色发光
            material.diffuse = vec3(0.4, 0.25, 0.05);
        }
    `,
});
tileset.customShader = customShader;

// 加Camera飞到广州
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(113.27, 23.13, 2000),
});
```

**踩坑：**
- `shadow` 拼写少了 s（应该是 `shadows`）
- `viewer.scene.sun = new Cesium.Sun()` 会报错，Cesium 默认就有太阳
- `Cesium3DTileStyle` 不支持 `emissive`，不报错但完全没效果
- `createOsmBuildingsAsync()` 包围盒是全球，`flyTo(tileset)` 相机不会动，要手动指定坐标


---

## 06-click-query-province — 点击省份查询（鼠标交互 + 射线法判点）

**位置：** `E:/my_repo/MyCesium/06-click-query-province/`
**文件：** `01-click-query-province.html`、`china-province.geojson`、`southsea.geojson`

加载中国省份边界 + 南海九段线，**点击地图自动检测点在哪个省份内**。核心算法是射线法（Ray Casting）。

### 坐标转换流程

```
屏幕坐标 (Pixel)            // {x, y} 像素
    ↓ camera.pickEllipsoid()
笛卡尔坐标 (Cartesian3)      // {x, y, z} 地球3D坐标
    ↓ Cartographic.fromCartesian()
地理坐标 (Cartographic)      // 弧度制经纬度
    ↓ Math.toDegrees()
经纬度 (度)                  // 113.27, 23.13
```

### 四种 Pick 方式

- `scene.pick(pos)` — 返回鼠标下的物体
- `scene.drillPick(pos)` — 返回鼠标下所有物体
- `scene.pickPosition(pos)` — 返回地形表面的3D坐标
- `camera.pickEllipsoid(pos)` — 返回椭球表面坐标（最快）

### 射线法关键代码

```js
function IspointinPolygon(lon, lat, polygon) {
  let inside = false;
  for (let i = 0, j = polygon.length - 1; i < polygon.length; j = i++) {
    const xi = polygon[i][0], yi = polygon[i][1];
    const xj = polygon[j][0], yj = polygon[j][1];
    const latcross = (yi > lat) !== (yj > lat);  // 一个在上一个在下
    if (latcross) {
      const crossX = (xj - xi) * (lat - yi) / (yj - yi) + xi;
      if (crossX > lon) inside = !inside;
    }
  }
  return inside;
}
```

### 踩坑

- `movement.position` 只是像素坐标，不能直接当经纬度用
- `Cartographic.fromCartesian()` 拿到的是弧度，要转度
- `camera.pickEllipsoid()` 不考虑地形高度
- 大范围 Polygon 放大消失 bug → 加 `depthTestAgainstTerrain = true`
