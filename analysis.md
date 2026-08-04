# 模板 001 · byonlab 流体丝绸 · v5.2（3D 质感突破）

> 来源：https://byonlab.com/#/community  
> 还原：本地纯 WebGL2 实现，无 Three.js，单文件 `index.html`  
> 当前：✅ 鼠标交互 + 3D 立体丝绸 + 明暗阴影（接近 byonlab 原版质感）

---

## 一句话技术总结

**Stable Fluids（GPU） + 速度场当 3D 表面高度场 → 法线计算 → Phong 光照**  
v5.2 的核心 trick：**把速度长度当 3D 表面高度**，速度梯度当法线，**Phong 光照**自然产生明暗阴影，3D 感就出来了。

---

## 关键技术演进（v3 → v5.2）

### v3 之前：只在"画", 没有"3D"
- 用速度长度当 1D 调色板索引
- 简单 `mix(bg, palette, lenv)`
- 结果：扁平白丝绸，**没有阴影**，**没有 3D 感**

### v5.2 突破：把速度场当 3D 表面
```glsl
// 1. 速度长度 = 高度
float h = length(velocity);

// 2. 高度梯度 = 表面法线（Sobel 算子）
vec2 normal = vec2(
  (h_ru + 2.0*h_r + h_rd) - (h_lu + 2.0*h_l + h_ld),
  (h_ld + 2.0*h_d + h_rd) - (h_lu + 2.0*h_u + h_ru)
);

// 3. Phong 光照 = 阴影 + 高光
float diff = max(0.0, dot(normal, lightDir));
float spec = pow(dot(normal, halfDir), 16.0);

// 4. 暗色（比 bg 暗 35%）
vec3 shadowColor = uBgColor.rgb * 0.65;
```

**这就是 byonlab 那种"3D 凸起"的秘密** —— 不是 displacement map，是 **fake 3D surface from 2D velocity field**。

---

## v5.2 关键技术细节

### 1. 13 点采样的 normal（v5.2 vs v5.1）
- v5.1：4 邻 + 4 对角 = 8 点
- v5.2：**9 点平均的中心 + 8 邻 Sobel = 13 点 + 步长 3 像素**
- 效果：normal 更平滑，silk 不再"颗粒感"

### 2. Normal scaling (1.8x)
```glsl
normal = normalize(normal * 1.8 + vec2(0.00001));
```
- 把速度梯度放大 1.8 倍 → 同样高度差产生更陡的"山坡"
- 太小 → 3D 感弱
- 太大（>2.5x）→ 整张图都成阴影（v5 第一次调过头就是这个）

### 3. 多光源（主光 + 补光）
```glsl
vec2 light1 = normalize(vec2(0.5, 0.7));   // 主光：右上 60°
vec2 light2 = normalize(vec2(-0.4, -0.2));  // 补光：左下（弱 0.25x）
```
- 主光强 100%，给主高光 + 主阴影
- 补光弱 25%，提亮阴影区（避免纯黑）

### 4. Blinn-Phong 高光（双 exponent）
```glsl
float spec1 = pow(dot(normal, half1), 18.0);  // 主高光
float spec2 = pow(dot(normal, half2), 22.0) * 0.4;  // 副高光
float spec = spec1 + spec2;
```
- 18 + 22 = 主副高光有微妙差异，不死板
- 不用 Phong，用 Blinn-Phong（half vector）→ 更稳定的物理

### 5. shadowWeight = lenv * 0.7（关键 trick）
```glsl
// 关键：只在 lenv 大（silk 区域）才应用 3D 阴影，bg 区域保持原色
float shadowWeight = lenv * 0.7;
vec3 finalColor = mix(uBgColor.rgb, litByDiff * (0.7 + diff * 0.5), shadowWeight + 0.3);
```
- bg 区域（lenv=0）→ 完全显示 bg（不被 shadow 污染）
- silk 区域（lenv>0）→ 70% 应用 3D 光照
- 解决 v5 第一次调过头"整个画面都暗"的问题

---

## 完整 pipeline（v5.2）

```
每帧 5 步：
  1. Advect        vel_0 → vel_1  (BFECC)
  2. Force (add)   vel_1 += force  (additive blend, uCenter=鼠标位置)
  3. Divergence    div = ∇·vel_1
  4. Pressure      pressure = Poisson(div)  [Jacobi 32 iter]
  5. GradSub       vel_0 = vel_1 - ∇p * dt

渲染（v5.2）：
  - 9 点平均速度场 → h_c
  - 8 邻 Sobel 算 normal（步长 3 像素）
  - 双光源 Phong
  - shadowWeight 控制只在 silk 区域应用 3D
  - alpha: 透明混合 bg + silk
```

---

## v3 → v5.2 演进对比

| 版本 | 视觉 | 关键技术 |
|------|------|----------|
| v3 | 完全无 silk | force shader 不读 uCenter + 没用 additive blend |
| v4 | 扁平白丝绸 | force 跟鼠标 + 调色板 + pow 放大 |
| v5.0 | 颗粒感强 | 初始 3D 尝试，单点 normal 太锐 |
| v5.1 | 3D 感 + 稍颗粒 | Sobel normal (2x uPx) |
| v5.2 | 3D 感 + 平滑丝绸 | Sobel normal (3x uPx) + 9 点平均 + shadowWeight |

---

## 参数（v5.2 可调）

```js
// 模拟
dt: 0.014
iterations_poisson: 32
mouse_force: 20
cursor_size: 0.2  // NDC 空间 0.2 = 20% 屏幕宽度
isViscous: false

// 渲染
bg: #dfe3e9 (223, 227, 233)  // 浅灰偏冷
shadowColor = bg * 0.65  // 35% 暗
normal_scale: 1.8  // normal 放大倍数
spec1_exponent: 18
spec2_exponent: 22
```

调参建议：
- `bg` 改深 → 整体氛围更深沉（深夜模式）
- `bg` 改浅 + `shadowColor = bg * 0.4` → 强烈对比（接近 byonlab）
- `normal_scale` ↑ → 3D 感更强
- `mouse_force` ↑ → silk 更"宽"，但 32 次压力可能爆

---

## byonlab bundle 反混淆路径

bundle: `https://byonlab.com/assets/index-CPqvyJk-.js`（1.3 MB minified）

关键类（混淆名 → 实际类）：
- `V` → Pointer 追踪器（mouse state: coords/diff）
- `q` → AutoPilot（无操作时 lerp 向随机目标）
- `P = new V` → 全局 pointer 实例
- `lt` → Simulation（管理 FBOs + shaders + step）
- `Pe` → ExternalForce pass（shader = `Oe`，blending: AdditiveBlending）
- `je` → Advection pass
- `tt` / `ft` → Pressure + GradSub
- `nt` → Divergence
- `Dt` → 最终渲染 class（output mesh 用 `he` shader）

Shader 字符串混淆名（const 变量）：
- `Oe` → external force
- `he` → final render (palette lookup, **byonlab 没做 Phong**)
- `ie` → advection (BFECC)
- `Me` → divergence
- `ue` → viscous diffusion
- `Ne` → pressure Jacobi

**关键发现**：byonlab 的 render `he` shader 只用 `mix(bgColor, c, lenv)`，**没有 Phong**。但它的 velocity 场自然产生那种 3D 凸起感 —— 因为 fluid sim 的 pressure waves 本身就形成那种 alternating pattern。所以严格说，我的 v5.2 用了不同的视觉技巧达到类似效果。

---

## 性能
- 1280×720 屏幕 + 模拟分辨率 0.5x → 640×360
- 60fps（M1 Mac / RTX 3060 都轻松）
- 主要瓶颈：32 次 Jacobi 压力迭代 + 13 邻域 normal 计算

---

## 文件结构
```
byonlab-liquid-metal/
├── index.html          # 单文件所有代码（~24KB）
├── preview-grid-v2.png # v5.2 4 联预览
├── preview-v17*.png    # v5.2 各位置单图
├── preview-compare-v2.png # 与 byonlab 原版对比
├── byon-original.png   # byonlab 原版（存档对比用）
└── analysis.md         # 本文档
```

---

## 参考资料

- Jos Stam 1999, "Stable Fluids"
- Bridson 2015, "Fluid Simulation for Computer Graphics"
- Phong / Blinn-Phong 模型
- WebGL2 + RG16F 参考：https://github.com/mharrys/fluids-2d
- byonlab bundle：`/assets/index-CPqvyJk-.js` 1.3MB
