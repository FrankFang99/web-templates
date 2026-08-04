# 模板 001 · byonlab 流体丝绸 · v6.1（抗碎纹 + 冷调丝绸色）

> 来源：https://byonlab.com/#/community  
> 还原：本地纯 WebGL2 实现，无 Three.js，单文件 `index.html`  
> 当前：✅ 鼠标交互 + 3D 立体丝绸 + 冷调中性色 + 无碎纹（接近 byonlab 原版质感）

---

## v6 关键改进（v5.2 → v6.1）

| 问题 | v5.2 | v6.1 解决 |
|------|------|----------|
| **碎纹** (grainy 颗粒) | force brush 太尖，d² falloff 有 hard edge | Gaussian falloff `exp(-2.5r²)` + 1.5x 大 brush + pre-blur 速度场 |
| **颜色偏暖** | 加了 warm highlight | 改为冷调中性色（slight blue-gray in shadows） |
| **高频累积** | velocity 会累积噪声 | velocity_drag = 0.998 每帧温和衰减 |
| **normal 太锐** | 步长 3 像素 + 1.8x scale | 步长 5 像素 + 1.5x scale（柔 + 平滑） |

---

## v6 三大关键技术

### 1. Gaussian falloff 替代 d²
```glsl
// v5.2
float d = 1.0 - min(length(offset), 1.0);
d *= d;  // d² 在 r=1 边界处是 0（hard edge）→ 碎纹源头

// v6
float r2 = dot(offset, offset);
float d = exp(-2.5 * r2);  // 永远平滑，r=1 时仍有 8.2% 力
```

### 2. Pre-blur 速度场
```glsl
// 在 render 之前，把 velocity 9 点高斯模糊
// 解决 render 时的 normal 抖动
const FS_BLUR = `...vec2 sum = vel * 4 + 4邻 * 2 + 4对角 * 1; out = sum/16;`
```
- render 读 velBlurred 而不是 vel
- 9 点高斯（中心 4x，4 邻 2x，4 对角 1x）
- 高频 noise 在 render 前就抹平了

### 3. 冷调中性色
```glsl
// 暗色：bg * (0.50, 0.55, 0.65) + 0.02 blue  → 略冷的灰
// 高光：palette 白色
// mix → 由 lenv 控制从冷暗到亮白
```

---

## 完整 pipeline（v6.1）

```
每帧 6 步：
  1. Advect        vel_0 → vel_1  (BFECC)
  2. Force (add)   vel_1 += force  (additive blend, Gaussian brush)
  3. Divergence    div = ∇·vel_1
  4. Pressure      pressure = Poisson(div)  [Jacobi 32 iter]
  5. GradSub       vel_0 = vel_1 - ∇p * drag  (drag=0.998 抗高频)
  6. Pre-blur      velBlurred = blur(vel_0)  (9 点高斯)

渲染（v6.1）：
  - 读 velBlurred（已平滑）→ 9 点平均 → h_c
  - 8 邻 Sobel 算 normal（步长 5 像素 + 1.5x scale）
  - 双光源 Blinn-Phong（spec exponent 18+22）
  - shadowWeight 控制只在 silk 区域应用 3D
  - 冷调 mix（cool shadow → palette highlight）
  - alpha: 透明混合 bg + silk
```

---

## 参数（v6.1 可调）

```js
// 模拟
dt: 0.014
iterations_poisson: 32
mouse_force: 20
cursor_size: 0.3       // v6: 0.2 → 0.3 (1.5x 大 brush)
isViscous: false
velocity_drag: 0.998   // v6: 每帧 0.2% 衰减 → 抗高频累积

// 渲染
bg: #dfe3e9 (223, 227, 233)  // 浅灰
shadowColor = bg * (0.50, 0.55, 0.65) + blue tint  // 冷调暗
normal_scale: 1.5
normal_step: 5
spec1_exponent: 18
spec2_exponent: 22
```

---

## v3 → v6.1 演进对比

| 版本 | 视觉 | 关键技术 |
|------|------|----------|
| v3 | 完全无 silk | force 不跟鼠标 + 没用 additive |
| v4 | 扁平白丝绸 | force 跟鼠标 + 调色板 |
| v5.1 | 3D 感 + 颗粒 | Sobel normal (2x uPx) |
| v5.2 | 3D 感 + 颗粒 | Sobel (3x uPx) + Phong |
| v6.0 | 太柔（silk 没了） | Gaussian brush + drag 0.985（过头） |
| **v6.1** | **3D 感 + 平滑 + 冷调** | **Gaussian + pre-blur + 冷色 + drag 0.998** |

---

## byonlab bundle 反混淆路径

bundle: `https://byonlab.com/assets/index-CPqvyJk-.js`（1.3 MB minified）

关键类（混淆名 → 实际类）：
- `V` → Pointer 追踪器
- `q` → AutoPilot（lerp 向随机目标）
- `P = new V` → 全局 pointer 实例
- `lt` → Simulation
- `Pe` → ExternalForce pass（shader = `Oe`）
- `je` → Advection pass
- `tt` / `ft` → Pressure + GradSub
- `nt` → Divergence
- `Dt` → 最终渲染 class

Shader 字符串混淆名：
- `Oe` → external force
- `he` → final render（仅 palette lookup，无 Phong）
- `ie` → advection (BFECC)
- `Me` → divergence
- `ue` → viscous diffusion
- `Ne` → pressure Jacobi

**注意**：byonlab 的 render shader 没做 Phong，他们的 3D 感完全靠 fluid sim 自身的 pressure waves。我用 Phong 显式做出 3D，效果更可控但风格略有不同。

---

## 性能
- 1280×720 屏幕 + 模拟分辨率 0.5x → 640×360
- 60fps（M1 Mac / RTX 3060 都轻松）
- 主要瓶颈：32 次 Jacobi 压力迭代 + pre-blur（+1 pass）+ 13 邻域 normal

---

## 文件结构
```
byonlab-liquid-metal/
├── index.html           # 单文件所有代码（~25KB）
├── preview-grid-v4.png  # v6.1 4 联预览
├── preview-v21*.png     # v6.1 各位置单图
├── byon-original.png    # byonlab 原版（存档对比用）
└── analysis.md          # 本文档
```

---

## 参考资料

- Jos Stam 1999, "Stable Fluids"
- Bridson 2015, "Fluid Simulation for Computer Graphics"
- Phong / Blinn-Phong 模型
- WebGL2 + RG16F 参考：https://github.com/mharrys/fluids-2d
- byonlab bundle：`/assets/index-CPqvyJk-.js` 1.3MB
