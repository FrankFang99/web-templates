# BYON 丝绸布料 · 模板 001 技术拆解

> 复刻目标：[byonlab.com](https://byonlab.com/#/community) 顶部那个鼠标移动让背景丝绸起伏的效果
> 当前版本：**v7.2**（最终稳定版）

---

## 效果概览

鼠标移动时，背景像液态丝绸布料被轻轻按压，鼠标位置出现圆润的 3D 凹凸 + 涟漪扩散。整体冷调银白色，非常高级感。

- **bg 色**：`#e2e8f0`（冷调灰蓝）
- **核心算法**：Jos Stam (1999) **Stable Fluids**（半拉格朗日平流 + 雅可比压力迭代）
- **核心反直觉**：渲染只读 `length(velocity)`，**没有 Phong / normal 估计**（v6 试过，碎纹根因）

---

## v7 vs v6 关键差异

| 维度 | v6.4 (失败) | v7.2 (成功) |
|---|---|---|
| Render shader | 估计 normal + Blinn-Phong 多光源 | `mix(bg, palette(length(vel)), t)` |
| 碎纹 | 多（normal 在高频 velocity 上有颗粒） | 极少（render-stage 1px blur 兜底） |
| Force brush | `exp(-2.5r²)` Gaussian | `(1-r)²` 圆形（byonlab 原版） |
| Viscous | 4 + 2 iter（弱） | 30 + 32 iter（byonlab 默认，强平滑） |
| bgColor | `#dfe3e9` | `#e2e8f0`（byonlab 同款） |
| cursor_size | 0.32 (NDC) | 0.18 (NDC, ≈ 18% 屏幕) |
| velocity_drag | 0.998 | 不需要（无） |
| render-stage blur | 7-tap 链式（过度） | 5-tap 1px 轻微（just enough） |

**根因**：v6.4 的 Phong 用 `length(vel)` 当 height 估计 normal。velocity 在 advection 后有高频噪声 → normal 有颗粒 → Phong 高光把这些颗粒放大了 → 看起来"碎纹"。

**修复**：byonlab 的 render shader **根本不算 normal**。它直接把 `length(vel)` 当 1D 颜色查表索引。silk 的 3D 感完全来自调色板的色阶分布（深色在 mid-t，亮色在 high-t，bg 在 t=0）。

---

## v7.2 仿真流程

每帧 6 步：

```
1. Advect        vel_0 → vel_1 (BFECC 修正)         // 平流
2. Force (add)   vel_1 += (1-r)² × mouse_force × dx  // ADDITIVE BLEND
3. Divergence    div = ∇·vel_1                        // 散度
4. Pressure      pressure = Poisson(div) [Jacobi 32x] // 压力
5. GradSub       vel_0 = (vel_1 - ∇p × dt)            // 减去压力梯度（保不可压）
5.5 Viscous      velViscous = Jacobi(vel_0, ν=30, 32x) // 扩散平滑（抗碎纹关键）
5.6 Render blur  velBlur = 5-tap 1px Gaussian(vel)    // 兜底去高频
6. Render        palette(length(velBlur)) → mix(bg, c, t)
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

✅ **已达成**：
- 整体丝绸质感（深色阴影 + 白色高光）
- 鼠标响应（force + advection）
- 流体连续性（Stable Fluids 算法）
- 冷调银白色调
- 大 brush（18% 屏幕）→ 大面积 silk 形变

⚠️ **仍有差距**：
- byonlab 一次能展示 4-5 个 silk blob（continuous mouse movement + 更多 cursor stops）
- 我们的 auto-pilot 一次只 1-2 个 blob
- byonlab 的 silk 内部有更细腻的"丝光线纹"（来自更密的 sub-step）

**优化方向**（如需继续追）：
1. auto-pilot 改成连续路径（不停在单点，绕大圆）
2. 增加 force 频率（每个 cursor stop 持续 force 几帧再移开）
3. sub-step 化 simulation（每帧 2-4 sub-step）

---

## 文件清单

```
byonlab-liquid-metal/
├── index.html                  # v7.2 主文件（25KB，单文件 demo）
├── analysis.md                 # 本文件
├── byon-original.png           # byonlab.com 原版截图
├── preview-grid-v7.png         # v7.2 4 联（不同鼠标位置）
├── preview-compare-v7.png      # byonlab 原版 vs v7.2 对比
├── preview-compare-v6-v7.png   # v6.4（碎纹） vs v7.2（丝滑） 对比
└── preview-v32*.png            # v7.2 4 张单图（中间过程）
```

---

## 参考资料

- **Jos Stam (1999)** "Stable Fluids" — SIGGRAPH 论文，Stable Fluids 算法源头
- **Stable Fluids in WebGL** — 各种 WebGL 实现参考
- **byonlab.com bundle** — 1.3MB minified JS，反混淆分析得到核心算法

---

*最后更新：2026-08-05 · v7.2 稳定版*
