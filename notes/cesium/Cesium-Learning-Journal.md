# Cesium Learning Projects

## 🟢 基础知识（必看）

### HTML 文件结构

```html
<!DOCTYPE html>                 <!-- 告诉浏览器这是 HTML5 -->
<html lang="en">                <!-- 根标签，lang 设置语言 -->
<head>                          <!-- 头部：放配置、引入文件 -->
    <meta charset="UTF-8">      <!-- 设置编码，支持中文 -->
    <script src="...Cesium.js"></script>   <!-- 引入 Cesium 核心库 -->
    <link href="...widgets.css" rel="stylesheet">  <!-- 引入 Cesium 样式 -->
    <title>标题</title>         <!-- 浏览器标签页上显示的文字 -->
</head>
<body>                          <!-- 身体：放页面上能看到的东西 -->
    <div id="cesiumcontainer"></div>  <!-- 一个容器，Cesium 地球画在这里面 -->
    <script type="module">      <!-- type="module" 表示可以用 import/await -->
        // 你的 JavaScript 代码写在这里
    </script>
</body>
</html>
```

### 常用 JS 语法

```js
// === 变量 ===
const x = 5;            // const = 常量/变量，后面不能再改
let y = '你好';         // let = 变量，可以改值
var z = 1;              // 旧写法，不用管

// === 箭头函数 ()=>{}
// 可以理解成"把一段代码装进盒子里，需要的时候执行"
// 写法： (输入) => { 要做的操作 }
const add = (a, b) => { return a + b; };
// 调用： add(1, 2)  →  得到 3

// === forEach（遍历数组）
const arr = ['a', 'b', 'c'];
arr.forEach(item => {
    console.log(item);  // 依次输出 a, b, c
});
// 相当于：对数组里每一项，执行 {} 里的代码

// === map（遍历数组并收集结果）
const result = arr.map(item => {
    return item + '!';   // 每项都加个感叹号
});
// result = ['a!', 'b!', 'c!']
// map 和 forEach 的区别：map 会收集返回值组成新数组

// === fetch（读文件）
const res = await fetch('文件名');     // 请求文件
const data = await res.json();          // 解析成 JSON（数组/对象）
const text = await res.text();          // 解析成纯文本
```

---

## 01-hello-cesium — 第一个 3D 地球

**项目位置：** `E:/my_repo/MyCesium/01-hello-cesium/`

**文件：** `01-hello-globe.html`（默认飞到旧金山+3D建筑）、`02-shenzhen-practice.html`（深圳）

### 完整代码解释

```html
<script src="https://cesium.com/downloads/cesiumjs/releases/1.137/Build/Cesium/Cesium.js"></script>
<!-- 从网上下载 Cesium 库，这样才能用 Cesium.Viewer 等命令 -->
```

```html
<div id="cesiumcontainer"></div>
<!-- 一个空盒子，id="cesiumcontainer" 是它的名字 -->
<!-- Cesium 地球会画在这个盒子里面 -->
```

```js
// ========== ① 读取 API Key ==========
const res = await fetch('apikey.txt');
// fetch = 读取文件
// 'apikey.txt' = 文件名（跟 html 在同个文件夹）
// await = 等文件读完再往下走
// res = response，读到的内容

Cesium.Ion.defaultAccessToken = (await res.text()).trim();
// res.text() = 把文件内容转成文本
// .trim() = 去掉首尾空格和换行
// Cesium.Ion.defaultAccessToken = 告诉 Cesium 你的钥匙
// 有了钥匙才能用 Cesium 的地图服务

// ========== ② 创建 3D 地球 ==========
const viewer = new Cesium.Viewer('cesiumcontainer');
// new = 创建一个新东西
// Cesium.Viewer(容器的id) = 创建 3D 地球
// 'cesiumcontainer' 对应上面 div 的 id
// viewer 就是那个地球，之后所有的操作都通过 viewer 做

// ========== ③ 飞到某个位置 ==========
viewer.camera.flyTo({
    destination: Cesium.Cartesian3.fromDegrees(110, 35, 8000000),
    // fromDegrees(经度, 纬度, 高度)
    // 110°E, 35°N ≈ 中国中心（河南陕西附近）
    // 8000000 = 800万米高，能看到整个中国
});
```

### 📌 常见易错

```js
// ❌ 错误
viewer.camera.flyto(...)      // to 小写了！JS 区分大小写
viewer.camera.flyTo(...)      // ✅ 正确

// ❌ 错误
document.getElementById('cesiumcontainer')  // 找不到
// 如果 div 的 id 是 cesiumContainer（C大写），就不能写 cesiumcontainer

// ❌ 错误
// 双击 .html 文件用浏览器打开 → fetch 会报错
// 必须用 http-server 启动
```

---

## 02-province-markers — 批量标注省会

**项目位置：** `E:/my_repo/MyCesium/02-province-markers/`

**文件：** `01-load-cities.html`（基础版）、`02-label-distance-limit.html`（防穿透）、`03-distance-display.html`（距离控制）

### JSON 数据是什么

```json
[
    { "name": "广州", "lon": 113.27, "lat": 23.13 },
    { "name": "北京", "lon": 116.40, "lat": 39.90 },
    { "name": "上海", "lon": 121.47, "lat": 31.23 }
]
```
- `[ ]` 表示数组（列表），里面每个 `{ }` 是一个城市
- `"name"` 城市名、`"lon"` 经度、`"lat"` 纬度
- 后面要用这些数据在地球上每个城市位置画一个点

### 完整代码解释

```js
// ========== ① 读取 JSON 数据 ==========
const res = await fetch('china-cities.json');
// 请求读取 china-cities.json 这个文件

const data = await res.json();
// 把读到的内容解析成 JS 能用的格式
// data 现在是一个数组，里面 34 个城市

// ========== ② 批量加实体 ==========
data.forEach(item => {
    // data.forEach = 对 data 数组里的每一项，执行一次 { } 里的操作
    // item = 当前正在处理的那个城市
    // 第一次 item = { name:"广州", lon:113.27, lat:23.13 }
    // 第二次 item = { name:"北京", lon:116.40, lat:39.90 }
    // ... 共 34 次

    viewer.entities.add({
        // viewer.entities.add = 在地球上加一个东西（点/文字/模型）
        // 每次循环加一个城市，34 次循环加 34 个

        position: Cesium.Cartesian3.fromDegrees(item.lon, item.lat, 0),
        // position = 放在哪里
        // item.lon = 这个城市的经度
        // item.lat = 这个城市的纬度
        // 第三个参数 0 = 高度，0 表示地面

        point: {
            // point = 画一个点
            pixelSize: 10,      // 点的大小（像素），越大点越粗
            color: Cesium.Color.RED,  // 颜色红色
        },

        label: {
            // label = 在点旁边加文字
            text: item.name,        // 显示这个城市的名字
            font: '20px sans-serif', // 字体：20像素大小，sans-serif 字体类型
            color: Cesium.Color.WHITE,  // 白色文字
            verticalOrigin: Cesium.VerticalOrigin.TOP,
            // 文字对齐方式：TOP = 点在文字下方
            // 不设置的话，文字和点重叠在一起

            pixelOffset: new Cesium.Cartesian2(0, 10),
            // 偏移量：往下移 10 像素（让文字和点不重叠）
            // Cartesian2(水平偏移, 垂直偏移)
            // 正数=右/下，负数=左/上

            disableDepthTestDistance: 0,
            // 0 = 文字被地球挡住时自动隐藏（不穿透地球背面）

            distanceDisplayCondition:
                new Cesium.DistanceDisplayCondition(0, 500000),
            // 控制显示距离：相机距离 0 ~ 50万米 之间才看到文字
            // 拉太远了文字挤在一起很难看，就不显示
        },
    });
});
```

### 重要概念：实体（Entity）

```js
viewer.entities.add({ ... })
// Entity = 地球上的一个东西。可以是一个点、一行文字、一个柱子、一个模型
// 一个 entities.add 可以同时加多种东西：
viewer.entities.add({
    point: { ... },    // 点
    label: { ... },    // 文字
    box: { ... },      // 柱子/方块（03 会用）
    model: { ... },    // 3D 模型
});
```

### 📌 常见易错

```js
// 标签在点下面
verticalOrigin: Cesium.VerticalOrigin.TOP    // 点在文字下方
verticalOrigin: Cesium.VerticalOrigin.BOTTOM // 点在文字上方
verticalOrigin: Cesium.VerticalOrigin.CENTER // 点在文字中间

// 偏移方向
pixelOffset: new Cesium.Cartesian2(0, 10)    // 下移10像素
pixelOffset: new Cesium.Cartesian2(10, 0)    // 右移10像素
pixelOffset: new Cesium.Cartesian2(0, -10)   // 上移10像素
```

---

## 03-gdp-visualization — GDP 柱状图

**项目位置：** `E:/my_repo/MyCesium/03-gdp-visualization/`

**文件：** `01-gdp-chart.html`（带地形）、`02-gdp-simple.html`（无地形）、`03-instancing.html`（优化版）、`test.html`（你的练习版）

### Entity Box 画柱子（基础版）

```js
const h = cities.gdp * 20;
// gdp 是 GDP 数值，乘 20 变成柱子高度
// 广州 GDP=31000 → h = 31000 * 20 = 620000 米

const w = 80000;
// 柱子的粗细 = 8万米（固定值，所有柱子一样粗）

viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(cities.lon, cities.lat, h / 2),
    // position 决定柱子中心在哪
    // h/2 = 柱子高度的一半
    // 如果 h=1000 米，position 高度 = 500 米
    // 那么柱子从地面（0米）到柱顶（1000米），刚好贴地
    // 简记：position 高度 = h/2 → 贴地

    box: {
        dimensions: new Cesium.Cartesian3(w, w, h),
        // dimensions = 柱子的尺寸
        // new Cartesian3(宽度, 深度, 高度)
        // (80000, 80000, h) = 8万米粗、h米高
        material: Cesium.Color.RED,  // 红色
    },
});
```

### 柱子贴地图解

```
高度 = h  →  ═══ 柱顶
              ║
              ║   ← 柱子中心 position = h/2
              ║
高度 = 0  →  ═══ 地面
```

### 柱顶标签

```js
viewer.entities.add({
    position: Cesium.Cartesian3.fromDegrees(lon, lat, h + 1000),
    // h + 1000 = 柱子高度再往上 1000 米
    // 贴柱顶，不会被柱子挡住

    label: {
        text: cities.name + '\n' + cities.gdp + '亿',
        // text = 显示的文字
        // + 是拼接： "北京" + 换行 + "49843亿"
        // \n = 换行符，让城市名和 GDP 分两行显示
        // 最终显示：
        //   北京
        //   49843亿

        font: '20px sans-serif',
        // 字体大小 + 类型

        color: Cesium.Color.WHITE,
        verticalOrigin: Cesium.VerticalOrigin.BOTTOM,
        // BOTTOM = 文字底部对齐 position
        // 效果：文字从柱子顶部往上生长

        disableDepthTestDistance: 0,
        // 0 = 文字不会被柱子/地球挡住

        distanceDisplayCondition:
            new Cesium.DistanceDisplayCondition(0, 10000000),
        // 0 ~ 1000万米范围内显示，太远就隐藏
    }
});
```

### Primitive Instancing（优化版）

**为什么要有这个？**
Entity Box 每个城市一个 draw call（GPU 指令），34 个城市=34 次。Instancing 把所有柱子打包成 1 次，快一些。

```js
// ========== ① 用 .map 代替 .forEach ==========
// .map 会收集每次 return 的结果，组成一个新数组
// .forEach 只做事，不返回结果

const instances = data.map(item => {
    // map 遍历每个城市，return 一个 GeometryInstance
    // 执行完后 instances = [工单1, 工单2, ..., 工单34]

    const h = item.gdp * 20;
    const w = 80000;

    // --- 1. 算柱子在地球上的位置 ---
    const center = Cesium.Cartesian3.fromDegrees(
        item.lon, item.lat, h / 2
    );
    // center = 这个柱子中心在哪（经纬度 + 半高）

    // --- 2. 让柱子"站直" ---
    const transform = Cesium.Transforms.eastNorthUpToFixedFrame(center);
    // 地球是球形的，广州的"上"和北京的"上"方向不一样
    // 这个函数算出柱子应该朝哪个方向才能"站直"
    // 没有这行，所有柱子都朝一个方向，看起来是歪的

    // --- 3. 返回一张"工单" ---
    return new Cesium.GeometryInstance({
        // GeometryInstance = 一张工单
        // 工单上写着"形状是什么 + 放在哪里"

        geometry: new Cesium.BoxGeometry({
            // BoxGeometry = 柱子的形状（局部坐标）
            // 局部坐标就是：不考虑地球，就在原点造一个柱子

            vertexFormat: Cesium.VertexFormat.POSITION_AND_NORMAL,
            // 告诉 GPU 只需要"位置+朝向"信息
            // 不需要纹理坐标，省内存

            minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2),
            maximum: new Cesium.Cartesian3(w/2, w/2, h/2),
            // = 柱子的两个对角
            // 从 (-w/2, -w/2, -h/2) 到 (w/2, w/2, h/2)
            // 中心在 (0,0,0)，宽 w，深 w，高 h
            // 注意：用 new Cartesian3，不要用 fromDegrees！
        }),

        modelMatrix: transform,
        // modelMatrix = 把柱子从"原点"移到"地球上的位置"
        // 就是前面算出来的 transform
    });
});

// ========== ② 所有工单一次性提交 GPU ==========
viewer.scene.primitives.add(new Cesium.Primitive({
    geometryInstances: instances,
    // instances = 那叠工单（34 个 GeometryInstance）

    appearance: new Cesium.PerInstanceColorAppearance({
        // 外观：怎么画这些柱子
        closed: true,         // 封闭实体（把背面也画上）
        translucent: false,   // 不透明（不计算透明度，更快）
    }),
}));

// ========== ③ 标签还得用 Entity ==========
// Primitive 不支持文字，所以标签还是用 entities.add
data.forEach(item => {
    const h = item.gdp * 20;
    viewer.entities.add({
        position: Cartesian3.fromDegrees(item.lon, item.lat, h + 1000),
        label: { text: item.name + '\n' + item.gdp + '亿', ... }
    });
});
```

### 完整数据映射公式

| 公式 | 含义 |
|------|------|
| `h = gdp * 20` | GDP → 柱子高度（米） |
| `position 高度 = h/2` | 柱子中心放半高处→底部贴地 |
| `label 高度 = h + 1000` | 标签在柱子顶上 1000 米 |
| `w = 80000` | 柱子粗细固定 8 万米 |

---

## ⚡ 性能优化

```js
// ⭐ 最关键的一行：不动时不渲染
// 默认 Cesium 每秒画 60 帧，哪怕你啥也没动
// 这行让 Cesium 只在拖拽/旋转时才渲染，省 GPU
viewer.scene.requestRenderMode = true;

// 关掉不必要的特效（对数据展示没影响但吃性能）
viewer.scene.globe.enableLighting = false;    // 关光照
viewer.scene.fog.enabled = false;             // 关雾
viewer.scene.fxaa = false;                    // 关抗锯齿
viewer.scene.highDynamicRange = false;        // 关 HDR

// 显示 FPS（左上角会显示）
viewer.scene.debugShowFramesPerSecond = true;
```

---

## ❌ 常见错误速查

```js
// === 1. fromDegrees / new Cartesian3 混用 ===
// ❌ 错：fromDegrees 用来算经纬度，不是用来算盒子尺寸的
minimum: Cesium.Cartesian3.fromDegrees(-w/2, -w/2, -h/2)
// 结果：盒子飞到奇怪的地方或消失

// ✅ 对：用 new Cartesian3 算盒子尺寸
minimum: new Cesium.Cartesian3(-w/2, -w/2, -h/2)

// === 2. forEach / map 选错 ===
// ❌ 错：forEach 不收集返回值
data.forEach(item => { return new GeometryInstance({...}) })
// 结果：return 的东西丢了，没人接

// ✅ 对：map 收集返回值组成新数组
const arr = data.map(item => { return new GeometryInstance({...}) })

// === 3. 柱子贴地 ===
// ❌ 柱子悬空
fromDegrees(lon, lat, h)      // 高度 = 全高，中心在 h，一半悬空

// ✅ 柱子贴地
fromDegrees(lon, lat, h/2)    // 高度 = 半高，地面到柱顶

// === 4. 大小写 ===
// ❌ Cesium 大小写敏感
flyto  → 报错
FlyTo  → 报错
flyTo  → ✅

// === 5. 启动方式 ===
// ❌ 双击 html 文件打开 → fetch 读文件会失败
// ✅ 必须用 http-server 启动
```

---

## ▶ 启动项目

```bash
# 打开终端（命令行）

# 进入项目目录
cd E:/my_repo/MyCesium/03-gdp-visualization

# 启动本地服务器
npx http-server -p 8080 -c-1

# 浏览器打开
# http://localhost:8080
# 然后点击对应的 .html 文件
```

## 🔑 API Key

```js
// 所有项目都这么读 key：
const res = await fetch('apikey.txt');
Cesium.Ion.defaultAccessToken = (await res.text()).trim();

// 换 key 只需改 apikey.txt 一个文件
```

> 💡 `fetch` 必须通过 http-server 才能用，直接双击 html 文件会报错。
