# 大众途观模拟驾驶 · 项目 SOP

> 记录本项目的目标、架构、素材制作流程、关键参数与踩坑经验，便于后续维护和复用。

---

## 1. 项目目标

用纯前端（HTML + Canvas 2D）做一个横向滚动驾驶画面：

- 背景持续滚动，营造前进感
- 一辆大众途观 SUV 固定在画面中下方
- 车轮随车速旋转，车身有轻微悬挂弹跳
- 支持调速按钮、车速显示、全屏
- 可运行在电脑浏览器、安卓车机浏览器、iPhone Safari

无任何框架、无 CDN 依赖，全部离线可用。

---

## 2. 文件清单

| 文件 | 作用 | 是否运行时必需 |
|---|---|---|
| `index_castle.html` | **城堡背景版**（当前主推） | ✅ |
| `index.html` | 原草地背景版（开发调试用） | ✅ |
| `car_player.html` | 单文件打包版（所有图片 base64 内联，用于车机/手机离线） | ✅ |
| `image.png` | 城堡风景背景图（1422×800） | ✅ |
| `tiguan_car.png` | 车身位图（2000×734，透明背景，已抠轮） | ✅ |
| `wheel.png` | 车轮贴图（143×143，透明） | ✅ |
| `tiguan.svg` | 车身矢量源文件（1000×367，车头朝左，轮位已挖空） | 素材源 |
| `tiguan.png` | 途观原始照片（2000×1333，已抠图，车头朝右） | 素材源 |
| `landscape.png` | 草地背景（旧版使用） | 旧版 |
| `index_v1_png.html` | 第一版（照片直接贴图）备份 | 归档 |
| `index.html.bak` | 更早的备份 | 归档 |
| `carbody.png` / `tiguan_body.png` / `tiguan_car.svg` | 中间试错产物 | 可删 |

---

## 3. 运行方式

### 3.1 电脑/局域网调试

```powershell
# 在项目目录启动静态服务
python -m http.server 8123 --bind 0.0.0.0
```

- 本机访问：`http://localhost:8123/index_castle.html`
- 局域网（手机/平板）：`http://<电脑局域网IP>:8123/index_castle.html`

### 3.2 车机 / 手机离线

直接用浏览器打开 `car_player.html` 单文件即可（图片全部内联）。

- 安卓车机：拷贝单文件 → 浏览器打开 → 点「全屏」
- iPhone：无法直接打开本地 HTML，需先通过 http 访问，再「分享 → 添加到主屏幕」

---

## 4. 渲染架构

```
requestAnimationFrame 循环
  ├─ drawBackground()   背景横向滚动（镜像拼接无缝）
  ├─ drawRoad()         底部路面 + 虚线车道线
  ├─ drawCar()          车身位图 + 两个旋转轮子
  └─ updatePhysics()    累加 bgOffset / wheelAngle / time
```

### 4.1 背景无缝滚动

背景图本身左右边缘不连续，**直接平铺会出现接缝**。采用「原图 + 水平镜像」交替铺满：

```js
const period = imgW * 2;                    // 原图 + 镜像 = 一个周期
const phase = ((bgOffset % period) + period) % period;

for (let i = 0; i < count; i++) {
  const idx = startIdx + i;
  const x = idx * imgW + phase;
  const mirrored = (((idx % 2) + 2) % 2) === 1;

  if (mirrored) {
    ctx.save();
    ctx.translate(x + imgW, 0);
    ctx.scale(-1, 1);
    ctx.drawImage(bgImage, 0, 0, imgW, imgH);
    ctx.restore();
  } else {
    ctx.drawImage(bgImage, x, 0, imgW, imgH);
  }
}
```

要点：
- `phase` 必须归一到 `[0, period)`，否则负偏移会导致块索引错乱
- 奇数块镜像、偶数块正向，保证相邻块边缘像素连续
- 滚动方向：`bgOffset` 增大 → 画面向右移动 → 车头朝左前进

### 4.2 车身与车轮

车身图是**已抠掉轮子**的透明 PNG。车轮单独绘制并可旋转：

```js
// 轮子在车身图中的相对位置（矢量化脚本实测值）
const LEFT_X  = 0.1892;   // 车头侧
const RIGHT_X = 0.7955;   // 车尾侧
const WHEEL_Y = 0.780;    // 轮心高度（相对车高）
const WHEEL_R_RATIO = 0.1809;

// 让轮底恰好落在地面线
const carY = groundY - carH * (WHEEL_Y + WHEEL_R_RATIO);
```

**注意**：车身图包含原照片的底部阴影，因此对齐基准是「轮底」而不是「图底」。

### 4.3 高清屏适配

```js
const dpr = baseW > 1400 ? 1 : Math.min(window.devicePixelRatio || 1, 2);
canvas.width  = Math.floor(baseW * dpr);
canvas.height = Math.floor(window.innerHeight * dpr);
```

- 宽屏（车机）用 1x，省 GPU
- 手机用最多 2x，避免模糊
- 所有绘制直接使用 `canvas.width/height`（物理像素），CSS 再缩回视口，无需额外 transform

---

## 5. 素材制作 SOP：照片 → 矢量车身

> 目标：把一张真实途观照片转成「扁平插画风、车头朝左、无轮子」的透明素材。
> 环境：`L:\CodeSoftWare\miniconda\python.exe`（Python 3.10）+ `opencv-python` / `Pillow` / `vtracer`

### 步骤 1 · 去水印

源照片车身中部有「汽车之家」水印，直接矢量化会保留。用 inpaint 抹掉：

```python
wm = np.zeros(a.shape[:2], np.uint8)
wm[640:752, 820:1085] = 255          # 水印区域（原图坐标）
a[:, :, :3] = cv2.inpaint(rgb, wm, 7, cv2.INPAINT_TELEA)
```

### 步骤 2 · 水平翻转

照片车头朝右，与「背景右移 = 前进」的方向冲突，需整体翻转：

```python
a = a[:, ::-1, :].copy()
W = a.shape[1]
wheels = [(W - 1 - cx, cy, r) for (cx, cy, r) in wheels]   # 轮子坐标同步镜像
```

### 步骤 3 · 裁剪 + 缩放

按 alpha 通道裁掉透明边，再缩放到宽 1000（约够 2 倍屏使用）。

### 步骤 4 · 抠掉轮子

在 alpha mask 上挖圆，使轮子位置变透明，供 canvas 叠加可旋转轮子：

```python
# 原图像素坐标（车头朝右时实测）
WHEELS_SRC = [(1458, 776, 90), (545, 768, 90)]   # 前轮、后轮
PAD = 10                                          # 多挖一点，保证轮胎边缘干净

yy, xx = np.mgrid[0:th, 0:tw].astype(np.float32)
for (scx, scy, sr) in WHEELS_SRC:
    cx, cy, r = (scx - x0) * scale, (scy - y0) * scale, (sr + PAD) * scale
    mask[(xx - cx) ** 2 + (yy - cy) ** 2 < r * r] = False
```

同时记录相对比例，供页面使用：

```
x/w = 0.1892（车头侧）、0.7955（车尾侧）
y/h = 0.787、0.7726
r/h = 0.1809
```

### 步骤 5 · 合成到「抠图色」背景

vtracer 不支持透明通道，因此把透明区填成**品红 `#FF00FF`**（车身不含此色）：

```python
comp = np.full((th, tw, 3), 0, np.uint8)
comp[:] = (255, 0, 255)
comp[mask] = rgb[mask]
comp = cv2.bilateralFilter(comp, 5, 35, 35)     # 保边平滑，去金属漆噪点
```

### 步骤 6 · vtracer 矢量化

```python
vtracer.convert_image_to_svg_py(
    TMP_PNG, OUT_SVG,
    colormode='color',
    hierarchical='cutout',   # 关键：各层不重叠
    mode='spline',           # 贝塞尔曲线，平滑
    filter_speckle=6,        # 去小噪点
    color_precision=6,
    layer_difference=22,
    corner_threshold=60,
    length_threshold=3.5,
    max_iterations=10,
    splice_threshold=45,
    path_precision=2,
)
```

### 步骤 7 · 删除背景层 + 加同色描边

```python
# 删除品红色 path（r>190, g<90, b>190）
# 其余 path 加 stroke 同色、width=1，消除相邻色块之间的白缝
tag = tag.replace('/>', ' stroke="%s" stroke-width="1" stroke-linejoin="round"/>' % color)
```

### 步骤 8 · SVG → 透明 PNG

页面直接用位图绘制（避免 canvas 每帧重新栅格化 SVG）。用 PIL 解析 `M/L/C` 路径并按 2 倍分辨率绘制到 RGBA 画布。

> 注意：vtracer 输出用 `transform="translate(x,y)"` 做局部坐标平移，解析时必须叠加。

---

## 6. 关键参数速查

| 参数 | 值 | 位置 | 说明 |
|---|---|---|---|
| 车高占比 | `H * 0.28` | `drawCar()` | 想让车更大改这里 |
| 车宽上限 | `W * 0.8` | `drawCar()` | 防止横向溢出 |
| 地面线 | `H * 0.85` | `drawCar()` / `drawRoad()` | 路面顶部 |
| 背景滚动速度 | `speed * 0.5` | `updatePhysics()` | 越大背景越快 |
| 轮子转速 | `speed * 0.008` | `updatePhysics()` | 越大转越快 |
| 车身弹跳 | `sin(time * 8) * 1.0` | `drawCar()` | 悬挂抖动幅度 |
| 最高车速 | `140` | `MAX_SPEED` | |

---

## 7. 单文件打包（用于离线分发）

思路：把三张图全部转 base64 内联进 HTML。

- 背景 `landscape.png` → **JPEG q88**（926 KB → 135 KB）
- 车身 `tiguan.svg` → 先渲染成 **PNG**（2000×734，27 KB）
- 车轮 `wheel.png` → PNG（10 KB）
- 最终 `car_player.html` 约 **236 KB**

替换方式：把 JS 里的文件名字符串直接换成 `data:image/...;base64,...`。

---

## 8. 跨平台适配

### 安卓车机

- 推荐单文件 HTML，浏览器直接打开
- 全屏：`requestFullscreen` + `webkit` 前缀
- 宽屏（>1400px）自动用 1x 渲染，省 GPU

### iPhone / iPad

| 限制 | 应对 |
|---|---|
| Safari 不支持页面全屏 API | 引导「分享 → 添加到主屏幕」，用 `apple-mobile-web-app-capable` 让桌面图标全屏启动 |
| 无法直接打开本地 HTML | 必须通过 http 访问，或部署到静态托管 |
| `100vh` 含地址栏 | CSS 用 `100dvh` 兜底，并监听 `visualViewport` 重算画布 |
| 高分屏模糊 | DPR 最多 2x 渲染 |

---

## 9. 踩坑记录

| 问题 | 原因 | 解决 |
|---|---|---|
| 车被压扁、轮子对不上 | 把 2000×1333 的图硬拉到 480×180（比例 8:3），且原图自带轮子又叠了手绘轮子 | 按原图比例绘制，去掉手绘轮子 |
| 车身透明、背景穿透 | 复杂路径自交，`fill` 规则判错 | 拆成简单封闭路径分层绘制 |
| SVG 有白色斑驳 | 照片金属漆高光被逐像素量化成碎块 | 中值滤波 + 均值漂移平滑 + 过滤过亮小层 |
| 相邻色块之间有白缝 | 各层独立 path，抗锯齿留出亚像素缝 | 每个 path 加同色 `stroke`，`width=1` |
| 删掉背景层后出现灰矩形 | vtracer `hierarchical='stacked'` 是叠层覆盖，底层被删导致上层显形 | 改用 `cutout`，各层不重叠 |
| 改了图但页面没变化 | 浏览器缓存了同名资源 | 文件名加 `?v=N` 参数 |
| 车看起来倒着走 | 车头朝右 + 背景右移 = 相对左移 | 把车水平翻转成朝左（或反向滚动背景） |
| `Assignment to constant variable` | 分支里对 `const` 重新赋值 | 改 `let` |
| canvas 每帧画 SVG 卡顿 | 缩放 drawImage 会重新栅格化 | 预渲染成 PNG 位图 |
| pip 安装被拒（WinError 5） | 沙箱不允许写 conda 目录 | 授权在沙箱外执行 pip |

---

## 10. 常见修改指引

| 想做 | 怎么改 |
|---|---|
| 换背景 | 替换 `bgFileName`，建议图片左右边缘尽量接近，否则接缝更明显（镜像法可兜底） |
| 车更大/更小 | 调 `drawCar()` 里的 `H * 0.28` |
| 车更靠上/下 | 调 `groundY`（车身和路面要一致改） |
| 背景更快/更慢 | 调 `updatePhysics()` 里的 `bgOffset += speed * X` |
| 轮子位置微调 | 调 `LEFT_X` / `RIGHT_X` / `WHEEL_Y` / `WHEEL_R_RATIO` |
| 重新生成车身素材 | 重跑第 5 节的矢量化流程，注意同步更新上面四个比例值 |
| 重新打包单文件 | 重跑第 7 节流程 |

---

## 11. 已知限制

- 车身是「照片矢量化」的扁平插画风，放大到很大尺寸会看到色块边缘
- 轮子是独立贴图，与车身轮拱之间可能有 1~2px 的视觉缝隙
- 背景图非无缝图，镜像拼接会呈现左右对称的重复感
- 车身位图包含原照片的底部阴影，在浅色背景下可见
