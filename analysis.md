# BYON 丝绸布料 · 模板 001 技术拆解

> 复刻目标：[byonlab.com](https://byonlab.com/#/community) 顶部那个鼠标移动让背景丝绸起伏的效果
> 当前版本：**v8**（整屏丝绸覆盖）

---

## 效果概览

鼠标移动时，背景像液态丝绸布料被轻轻按压，鼠标位置出现圆润的 3D 凹凸 + 涟漪扩散。整体冷调银白色，非常高级感。

- **bg 色**：`#e2e8f0`（冷调灰蓝）
- **核心算法**：Jos Stam (1999) **Stable Fluids**（半拉格朗日平流 + 雅可比压力迭代）
- **核心反直觉**：渲染只读 `length(velocity)`，**没有 Phong / normal 估计**（v6 试过，碎纹根因）

---

## v7.2 → v8 关键差异

| 维度 | v7.2 (成功但单 blob) | v8 (整屏丝绸) |
|---|---|---|
| cursor_size | 0.18（18% 屏幕）| **0.45**（45% 屏幕）|
| mouse_force | 20 | **25** |
| Auto-pilot | lerp to target（停单点）| **Lissajous 连续路径**（不停单点）|
| render-stage blur | 5-tap 1px Gaussian | **3-tap 0.5px 轻量**（保留织物细腻纹理）|
| bg | `#e2e8f0` | **`#eef2f6`**（更白，接近 byonlab 实际）|
| 多 silk blob | 1-2 个同时存在 | **4-6 个同时存在，整屏覆盖** |
| 丝绸织物纹理 | 几乎没 | **保留**（v40c 顶部那种细密波纹）|

**v8 核心思路**：byonlab 的"完整绸布"效果 = **大 brush** + **连续 auto-pilot 路径** + **强 viscous 持续**。
- 大 brush：cursor_size 0.45 让单次 force 覆盖 ~45% 屏幕
- 连续路径：Lissajous（两个 ghost cursor 沿椭圆路径运动）→ silk 持续扫过全屏
- 多 ghost cursor：主 cursor 位置 = 两个 ghost 的平均 → silk 在全屏留下多个 trail

**v6.4 碎纹根因**（v7 解决）：Phong 用 `length(vel)` 当 height 估计 normal → 高频 velocity 场上的 normal 有颗粒 → Phong 高光放大颗粒。byonlab 原作直接 `palette(length(vel))`，不估 normal，silk 感全在调色板里。

---

## v8 仿真流程

每帧 6 步：

```
1. Advect        vel_0 → vel_1 (BFECC 修正)         // 平流
2. Force (add)   vel_1 += (1-r)² × mouse_force × dx  // ADDITIVE BLEND
3. Divergence    div = ∇·vel_1                        // 散度
4. Pressure      pressure = Poisson(div) [Jacobi 32x] // 压力
5. GradSub       vel_0 = (vel_1 - ∇p × dt)            // 减去压力梯度（保不可压）
5.5 Viscous      velViscous = Jacobi(vel_0, ν=30, 32x) // 扩散平滑（抗碎纹关键）
5.6 Render blur  velBlur = 3-tap 0.5px Gaussian(vel)  // 轻微，保留织物纹理
6. Render        palette(length(velBlur)) → mix(bg, c, t)
```

**v8 仿真参数**：
```js
{
  dt: 0.014,
  iterations_poisson: 32,
  iterations_viscous: 32,
  viscous: 30,
  mouse_force: 25,
  cursor_size: 0.45,         // v8 大 brush
  isViscous: true,
}
```

**v8 auto-pilot**：Lissajous 连续路径
```js
// 两个 ghost cursor 沿不同椭圆路径运动
const t = (now - startTime) / 1000
const x1 = cos(phase1 + t * 0.7) * 0.7
const y1 = sin(phase1 + t * 0.91) * 0.49
const x2 = cos(phase2 + t * 0.4) * 0.36
const y2 = sin(phase2 + t * 0.5) * 0.48
// 主 cursor = 两个 ghost 的平均 → silk 持续扫过全屏
pointer.x = (x1 + x2) * 0.5
pointer.y = (y1 + y2) * 0.5
```

---

## 关键参数（SIM_PARAMS）

```js
{
  dt: 0.014,                  // 步长（byonlab 默认）
  iterations_poisson: 32,     // 压力迭代（byonlab 默认；不可压解精度）
  iterations_viscous: 32,     // 粘性迭代（强！byonlab 默认 32）
  viscous: 30,                // 粘性系数（byonlab 默认；越大越平滑）
  mouse_force: 20,            // 鼠标力（byonlab 默认；不要改）
  cursor_size: 0.18,          // brush 半径 NDC（屏幕 ~18%）
  isViscous: true,            // 开启粘性（开！byonlab 默认 false 但真要丝滑要开）
}
```

**调参经验**：
- `mouse_force` 决定 silk 凸起的强度
- `cursor_size` 决定 brush 大小（越大越"软"，越小越"尖"）
- `viscous` + `iterations_viscous` 决定平滑度（值大 → 更丝滑但形态扩散）
- `iterations_poisson` 影响压力解的精度（高 → 流体感更"对"，但慢）
- 抗碎纹核心：**viscous 30 + 32 iter + render-stage 1px blur** 三者缺一不可

---

## 调色板设计（5-stop 1D ramp）

```js
const stops = [
  { t: 0.00, rgb: [0.92, 0.94, 0.97] },  // 接近 bg
  { t: 0.30, rgb: [0.30, 0.34, 0.42] },  // 深冷灰（silk 阴影）
  { t: 0.55, rgb: [0.55, 0.60, 0.70] },  // 中冷灰（silk 主体）
  { t: 0.80, rgb: [0.85, 0.88, 0.92] },  // 浅冷灰（silk 亮区）
  { t: 1.00, rgb: [0.99, 0.99, 1.00] },  // 冷白高光
]
```

**设计思路**：`mix(bg, palette(t), t)` 公式下：
- t=0 → bg（无 silk）
- t=0.3 → bg×0.7 + dark×0.3（深色阴影）
- t=0.55 → bg×0.45 + mid×0.55（中色主体）
- t=0.8 → bg×0.2 + light×0.8（亮区）
- t=1.0 → white（高光）

**调色经验**：
- 想要更"黑"的 silk 阴影 → 把 t=0.30 的 RGB 调低（0.20, 0.25, 0.32）
- 想要更"白"的高光 → 把 t=1.00 推到 (1.0, 1.0, 1.0)
- 想要暖色（米白）→ 所有 stop 加 0.05 红，-0.02 蓝
- 想要暗色模式 → bg 改 #1a1d22，所有 stop 反转（高亮变深色）

---

## 变体速查

### 暗色银丝绸
```js
// CSS 变量改：
--bg: #1a1d22;
--text: #e2e8f0;
--text-muted: rgba(226, 232, 240, 0.55);

// palette 反转：
const stops = [
  { t: 0.00, rgb: [0.10, 0.11, 0.14] },
  { t: 0.30, rgb: [0.55, 0.58, 0.65] },  // silk 在暗色下变亮
  { t: 0.55, rgb: [0.40, 0.45, 0.55] },
  { t: 0.80, rgb: [0.25, 0.28, 0.35] },
  { t: 1.00, rgb: [0.08, 0.09, 0.12] },
]

// render 的 bgColor uniform 同步改：
gl.uniform4f(..., 0x1A/255, 0x1D/255, 0x22/255, 1.0)
```

### 金色丝绸
```js
const stops = [
  { t: 0.00, rgb: [0.92, 0.88, 0.74] },  // 暖白
  { t: 0.30, rgb: [0.45, 0.32, 0.18] },  // 深棕
  { t: 0.55, rgb: [0.78, 0.60, 0.30] },  // 金色主体
  { t: 0.80, rgb: [0.95, 0.85, 0.55] },  // 亮金
  { t: 1.00, rgb: [1.00, 0.95, 0.75] },  // 极亮金
]

// bg 同步调暖：#f5ecd9
```

### 强烈对比（更"赛博"）
```js
// viscous 调小（4），让 silk 边缘更硬
viscous: 4.0,
iterations_viscous: 8,
// palette 加大反差
{ t: 0.00, rgb: [0.50, 0.55, 0.65] },
{ t: 0.30, rgb: [0.10, 0.12, 0.18] },
```

---

## 已知问题 & 解决

| 问题 | 原因 | 解决 |
|---|---|---|
| 碎纹（grainy） | normal 估计放大 advection 噪声 | 丢 Phong 用 palette 查表 |
| 颜色死灰 | bg/palette 不搭 | bg 用 byonlab #e2e8f0，palette 5-stop 渐变 |
| 形态太硬 | viscous 太小 | viscous 30 + 32 iter（v7 默认）|
| 鼠标处有微纹理 | auto-pilot 路径高频累加 | render-stage 5-tap 1px blur |
| 整屏逐渐发暗 | velocity 累积 | 已通过 viscous 解决（无需 drag）|

---

## 性能 & 兼容性

- **目标设备**：现代桌面浏览器（Chrome 90+ / Edge 90+ / Firefox 88+）
- **WebGL2 必需**（EXT_color_buffer_float 扩展）
- **模拟分辨率**：0.5x（1280×720 屏幕 → 640×360 模拟）
- **帧率**：桌面 60fps，笔记本 30-60fps
- **移动端**：未优化（auto-pilot 在 mobile 没意义，需要 touch 适配）

---

## 跟 byonlab 原作的差距

✅ **已达成**（v8 完整解决）：
- 整屏丝绸覆盖（4-6 个 silk blob 同时存在）
- 整体丝绸质感（深色阴影 + 白色高光）
- 鼠标响应（force + advection）
- 流体连续性（Stable Fluids 算法）
- 冷调银白色调
- 大 brush（45% 屏幕）→ 大面积 silk 形变
- 织物纹理保留（v40c 顶部那种细密丝绸波纹）
- Lissajous 连续路径 auto-pilot

✅ **基本达成**：
- byonlab 那种"完整绸布"感
- bg 接近白（#eef2f6）
- 多 silk blob 互相堆叠形成绸布褶皱

⚠️ **细微差距**（v8 已经很接近，不影响主要观感）：
- byonlab 的 silk 内部还有更细的"丝光线纹"（可能是 sub-step 模拟的副产物）
- byonlab 鼠标区域的凹陷感更深（force 略大一些能拉近）

---

## 文件清单

```
byonlab-liquid-metal/
├── index.html                  # v8 主文件（26KB，单文件 demo）
├── analysis.md                 # 本文件
├── byon-original.png           # byonlab.com 原版截图
├── preview-grid-v8.png         # v8 4 联（整屏丝绸）
├── preview-compare-v8.png      # byonlab 原版 vs v8 对比
├── preview-compare-v7.png      # byonlab vs v7.2 旧版对比
├── preview-compare-v6-v7.png   # v6.4（碎纹） vs v7.2（丝滑）对比
└── preview-v40*.png            # v8 4 张单图
```

---

## 参考资料

- **Jos Stam (1999)** "Stable Fluids" — SIGGRAPH 论文，Stable Fluids 算法源头
- **Stable Fluids in WebGL** — 各种 WebGL 实现参考
- **byonlab.com bundle** — 1.3MB minified JS，反混淆分析得到核心算法
- **Lissajous 曲线** — 用两个 sin/cos 频率比生成连续路径

---

*最后更新：2026-08-05 · v8 整屏丝绸版*
