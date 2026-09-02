# 开发学习指南：从零做「GPS 轨迹 3D 可视化」

> 本文件写给**开发新手**（也是给项目作者自己复盘用）。分为两部分：
> 1. **技术栈与概念讲解** —— 帮你理解这个项目用了什么、为什么这样用，以及如何继续学习。
> 2. **测试与迭代流程** —— 记录项目是如何一步步做出来、如何测试、如何排 bug 的。

---

## 一、技术栈与概念讲解（给小白）

### 1. 前端三件套：HTML / CSS / JavaScript

这个项目是**纯前端**，没有后端、没有服务器，浏览器打开 `.html` 就能跑。

- **HTML**：页面的「骨架」。定义有哪些面板、按钮、画布（`<div>`、`<button>`、`<canvas>` 等）。
- **CSS**：页面的「外观」。定位（`position: fixed` 让面板悬浮）、配色、圆角、毛玻璃（`backdrop-filter`）等。
- **JavaScript**：页面的「大脑」。所有逻辑（解析文件、建 3D 场景、录制视频）都是 JS 写的。

> 本项目把三者写在一个文件里（内联 `<style>` 和 `<script>`），便于新手理解和分发。

### 2. ES Modules 与 import map（CDN 引入第三方库）

现代浏览器支持「ES 模块」，可以用 `import` 引入别的库。但我们没有安装环境，所以用 **CDN（内容分发网络）** 直接在线加载：

```html
<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
  }
}
</script>
<script type="module">
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
// ...
</script>
```

- `import map` 把简短的 `'three'` 映射到真实的 URL，之后就能 `import` 了。
- **代价**：必须联网才能加载库（这也是为什么视频录制要预热、避免卡顿）。

### 3. WebGL 与 Three.js：浏览器里画 3D

**WebGL** 是浏览器内置的 3D 绘图底层接口（直接写很繁琐）。**Three.js** 是对它的封装，让 3D 变得简单。

Three.js 的核心概念（本项目都用到了）：

| 概念 | 类比 | 本项目中的例子 |
|---|---|---|
| Scene（场景） | 一个「舞台」 | `scene` |
| Camera（相机） | 观众的眼睛 | `PerspectiveCamera` |
| Renderer（渲染器） | 把舞台画到屏幕 | `WebGLRenderer` |
| Geometry（几何体） | 物体的形状 | `TubeGeometry`（管状轨迹）、`PlaneGeometry`（地面） |
| Material（材质） | 物体的表面 | `MeshStandardMaterial`、`MeshBasicMaterial` |
| Mesh（网格） | 几何体 + 材质 = 物体 | 轨迹 tube、地形 terrain |
| Light（灯光） | 照亮物体 | `AmbientLight`、`DirectionalLight` |
| Group（组） | 把多个物体打包 | `exportGroup`、`satelliteGroup` |

> **学习建议**：先跑通 three.js 官方的「Creating a scene」教程，理解 Scene/Camera/Renderer/Mesh 四要素，再看本项目会轻松很多。

### 4. 坐标系与投影：从经纬度到 3D 场景坐标

这是本项目的**灵魂**，也是最容易搞混的地方。

- **GPS 数据**是经纬度（WGS84），单位是「度」。
- **3D 场景**需要「米」作单位的平面坐标（x、z），y 轴代表海拔。
- 我们用 **Web Mercator（Web 墨卡托）投影**把经纬度换算成米：

```js
const R_MERC = 6378137;
function mercX(lon) { return R_MERC * lon * Math.PI / 180; }          // 东西方向
function mercY(lat) { /* 用 tan/ln 公式，略 */ }                        // 南北方向
```

- 以轨迹第一个点为原点，算出每个点的相对米制偏移，就是场景里的 `x`、`z`；`y` 用海拔（归一化 + 纵向夸张）。

> **为什么用墨卡托而不是更简单的「1 度 ≈ 111 公里」？** 因为卫星地图瓦片也是墨卡托投影，用同一套投影，轨迹才能和底图**精确对齐**（否则会错位约 15%~20%）。

### 5. 地图瓦片（slippy map / XYZ tiles）

卫星影像不是一张大图，而是按「缩放级别 z + 行列 x/y」切成无数张小图（瓦片）。

- 每个瓦片 256×256 像素。
- 缩放级别 z 越大，瓦片越小、越精细（z=15 约 1.2 米/像素）。
- URL 里用 `{z}/{x}/{y}` 占位，替换成具体数字就能取到对应瓦片。

本项目把轨迹包围盒附近的一圈瓦片加载下来，**按墨卡托米制坐标摆放**，拼成一张地面，轨迹就「落」在卫星图上了。

### 6. 地形高程（DEM）与 terrarium 编码

「真实 3D 地形」需要每个位置的高度（DEM，数字高程模型）。AWS 地形瓦片把高度编码进 RGB 颜色里（**terrarium** 编码）：

```
海拔 = (R * 256 + G + B / 256) - 32768
```

本项目解码每个瓦片像素的高度，构建一个网格地面（`BufferGeometry`），把每个顶点抬升到对应高度，再贴卫星纹理，就得到起伏的真实地形。

### 7. 异步与 Promise / async-await

加载瓦片、读取文件都是「异步」操作（不能立刻拿到结果）。本项目大量使用：

```js
async function buildTerrainData(points) {
  await Promise.all(jobs);   // 等所有瓦片下载完
  // ...
}
```

- `Promise`：表示「将来会有结果」的操作。
- `async/await`：让异步代码写起来像同步一样，可读性高。
- **理解这一点**是看懂「地形/卫星瓦片加载」的关键。

### 8. 视频录制：MediaRecorder + canvas.captureStream

不用额外库，浏览器原生就能录视频：

```js
const stream = renderer.domElement.captureStream(60); // 抓取 canvas 的 60fps 画面
const rec = new MediaRecorder(stream, { mimeType: 'video/webm' });
rec.start();
// ... 动画播放 ...
rec.stop(); // 得到 blob → 下载
```

- `captureStream` 只能抓当前 canvas 画面（所以录制时要把 3D 场景渲染到 canvas）。
- 浏览器支持度：WebM（Chrome/Firefox），MP4（Chrome/Edge）。

### 9. 文件解析：DOMParser

GPX/KML 都是 XML 文本。浏览器原生 `DOMParser` 就能解析，无需第三方库：

```js
const doc = new DOMParser().parseFromString(text, 'application/xml');
doc.getElementsByTagNameNS('*', 'trkpt'); // 取所有轨迹点
```

> **教训**：早期用了第三方库 `togeojson`，结果它 CDN 路径 404 还解析不了高驰的 KML，最终改用原生 `DOMParser` 直接解析，更稳、还能顺带取到心率/速度等扩展字段。

---

## 二、测试与迭代流程

### 1. 迭代时间线（本项目是怎么一步步长出来的）

1. **v0 骨架**：单个 HTML + Three.js，解析 GPX/KML、画轨迹 + 网格、OrbitControls、GLB 导出、飞越视频。
2. **修复 KML 崩溃**：定位到 `togeojson` 库失效 + 把 KML 的 `<Point>` 途经点误当轨迹点 → 改用原生 DOM 解析，并顺带提取心率/速度/距离。
3. **速度着色 + 仪表盘 + 回放**：轨迹可按速度着色；播放头 + 实时速度/海拔/心率。
4. **卫星底图**：Esri 影像瓦片 + Web Mercator 对齐，平面图锁定俯视 + 方位角圆盘。
5. **UI 重构**：三个面板可收起，适配小屏。
6. **真实 3D 地形**：AWS terrarium DEM 解码 + 地形网格 + 卫星纹理，轨迹贴合。
7. **视频录制重构**：多模式（跟随/定点/居中/自定义/自由）、轨迹点同步、WebM/MP4、16:9。
8. **细节打磨**：推荐点去重、录制时隐藏图标、自由录制同步等。

### 2. 测试方法论（每次都这么做）

- **先测主流程**：加载 GPX → 能显示轨迹？能着色？能回放？
- **再测边界**：加载 KML（无心率/速度/时间）→ 相关项是否优雅降级（显示「—」、选项自动隐藏）？
- **测每个按钮**：底图切换、着色切换、播放、导出 GLB、导出视频。
- **看控制台**：`F12` 打开 DevTools，看 `console` 有没有报错、`Network` 里瓦片是否 200。

### 3. 遇到的典型 bug 与排查思路

| 现象 | 根因 | 排查方法 | 修复 |
|---|---|---|---|
| KML 加载报 `Cannot read properties of undefined (reading 'x')` | 第三方库失效 + `<Point>` 坐标被误解析 | 看报错堆栈 → 检查 CDN 状态（curl 返回 404）→ 检查数据格式 | 原生 DOM 解析，只取 `<LineString>` |
| 轨迹与卫星图错位 | 轨迹用等距圆柱近似，瓦片用墨卡托 | 理解两种投影差异 | 统一用 Web Mercator |
| 轨迹穿进地形 | 轨迹管底低于地形面 | 观察管半径 vs 抬升量 | 增大 `baseOffset`、加密地形网格 |
| 推荐 3 个相同海拔点 | 平台被当成多个峰 | 打印推荐点索引 | 严格局部极大值 + 空间去重 |
| 跟随录制太快 | 默认时长太短 | 肉眼观感 | 默认时长 8s→15s，上限 60s |
| 地形瓦片加载失败 | AWS 跨域 CORS 偶发 | 看 Network / console 报错 | 捕获异常回退平面图（可换 MapTiler） |

### 4. 迭代经验总结

- **能用原生就少用第三方**：解析 XML、录视频、导出模型都能用浏览器原生能力，减少依赖就减少故障点。
- **数据先行**：先搞清楚数据格式（GPX 里有啥字段、KML 里没有啥），再写代码，能避免大量返工。
- **优雅降级**：KML 没有心率就显示「—」、地形加载失败就回退平面图，而不是崩溃。
- **分离关注点**：轨迹（可导出）、卫星影像（版权不可导出）分开成不同的 Group，既保证功能又合规。
- **小步验证**：每加一个功能就刷新测一次，别攒到最后一起测。

---

## 三、后续学习建议

1. **three.js 官方文档与示例**：`examples/jsm/` 下的例子是最好的教材。
2. **地图投影**：搜「Web Mercator」「slippy map tiles」深入理解瓦片与投影。
3. **地理数据格式**：GPX、KML、GeoJSON、FIT（佳明/高驰心率数据）的关系与转换。
4. **Blender 相机与影视运镜**：如果想做出更专业的飞越视频，学习关键帧、缓动曲线（easing）、跟随/环绕/推拉镜头。
5. **WebGL 进阶**：着色器（Shader）、顶点颜色、法线，理解 Three.js 底层。
