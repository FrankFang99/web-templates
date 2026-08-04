# 模板 001 · byonlab 流体丝绸 · v4（已突破）

> 来源：https://byonlab.com/#/community  
> 还原：本地纯 WebGL2 实现，无 Three.js，单文件 `index.html`  
> 当前：✅ 鼠标交互丝滑流体效果正常运转，质感接近原站

---

## 一句话技术总结

**Stable Fluids（GPU 实现） + 速度场 → 调色板查色 + 鼠标位置驱动 force pass**  
不是 metaball，不是 displacement map，是 Jos Stam 1999 那套流体模拟在 GPU 上跑出来的「速度场」再用调色板渲染成「丝绸色」。

---

## 关键技术（按重要性）

### 1. Stable Fluids 算法（核心）
Jos Stam 1999 论文，5 步流水线（每帧）：

```
1. Advect   vel ← advect(vel, dt)            // 半拉格朗日 + BFECC 误差修正
2. Force    vel += force * brush_falloff     // 鼠标力（ADDITIVE blend）
3. Divergence   div ← ∇·vel
4. Pressure  pressure ← Poisson(div)          // Jacobi 32 iter
5. GradSub   vel ← vel - ∇p * dt             // 投影到无散度场
```

#### 1.1 BFECC advection
普通 semi-Lagrangian 有数值耗散（看起来会糊），BFECC 加一个 back-and-forth 误差估计修正。

#### 1.2 32 次 Jacobi
压力迭代次数是 byonlab 的默认值。少了 → 流不动；多了 → 浪费。

#### 1.3 isViscous=false（性能关键）
byonlab 默认不跑 viscous diffusion。我 v3 跑了，浪费性能；v4 关掉。

### 2. Force pass 用 ADDITIVE BLEND
```js
gl.enable(gl.BLEND)
gl.blendFunc(gl.ONE, gl.ONE)  // dst + src → result
```
force shader 输出 `vec4(force * d, 0, 1)`，加到 vel_1 上。

**v3 的 bug**：先 clear vel.write，再 advect+force 两遍 add — 慢且错。  
**v4 的修复**：用 additive blend，一次画完。

### 3. FS_EXTERNAL_FORCE 用 uCenter（关键修复）
v3 的 bug：shader 接收 `uCenter` 和 `uScale`，但 shader 内部写死 `(vUv - 0.5)`，**完全不用 uCenter**。  
结果：力永远在屏幕中央，鼠标移动方向改变力方向但位置不动。

v4 修复：
```glsl
vec2 ndc = (vUv - 0.5) * 2.0;
vec2 offset = (ndc - uCenter) / max(uScale, vec2(0.0001));
float d = 1.0 - min(length(offset), 1.0);
d *= d;
fragColor = vec4(uForce * d, 0.0, 1.0);
```

**byonlab 原版自己也有这个 bug**（看 bundle 里的 `Oe` shader），但它的 autorotation 让用户感受不到。

### 4. Render = 调色板查色（视觉核心）
```glsl
vec2 vel = texture(uVelocity, vUv).xy;
float lenv = pow(clamp(length(vel), 0.0, 1.0), 0.5);  // 平方根放大
vec3 c = texture(uPalette, vec2(lenv, 0.5)).rgb;
vec3 outRGB = mix(uBgColor.rgb, c, lenv);
```

**这就是为什么"silk"**：把每个像素的速度长度当作 1D 颜色查表的索引。  
- 速度小 → bg 颜色（接近背景）  
- 速度大 → 高亮白（silk 强光）

**平方根放大**（`pow(lenv, 0.5)`）让小速度值也有可见对比。这是 v4 增强的，byonlab 没做。

### 5. 调色板 256×1 灰度渐变
```js
// 3 stop: bg(208) → 中银(235) → 极亮(255)
if (t < 0.5) { 208 + (235-208)*k ... }
else { 235 + (255-235)*k ... }
```
v3 用了 220→255 太接近 bg，silk 看不出。v4 拉宽到 208→255。

### 6. 自动巡航（持续 motion）
没用户操作 1.5s 后，鼠标位置自动 lerp 向随机目标：
```js
pointer.x += (autoTarget.x - pointer.x) * 0.04
```
v3 的 0.015 太慢，silk 看起来静止。v4 调到 0.04。

每帧的 `pointer.dx` 都非零 → force pass 持续触发 → 持续 silk。

---

## byonlab bundle 反混淆路径

bundle: `https://byonlab.com/assets/index-CPqvyJk-.js`（1.3 MB minified）

关键类（混淆名 → 实际类）：
- `V` → Pointer 追踪器（mouse state: coords/diff, autorotate 触发）
- `q` → AutoPilot（无操作时 lerp 向随机目标）
- `P = new V` → 全局 pointer 实例
- `lt` → Simulation（管理 FBOs + shaders + step）
- `Pe` → ExternalForce pass（shader = `Oe`，material blending: AdditiveBlending）
- `je` → Advection pass
- `tt` / `ft` → Pressure + GradSub
- `nt` → Divergence
- `Dt` → 最终渲染 class（output mesh 用 `he` shader）

Shader 字符串混淆名（const 变量）：
- `Oe` → external force
- `he` → final render (palette lookup)
- `ie` → advection (BFECC)
- `Me` → divergence
- `ue` → viscous diffusion（isViscous=true 才用）
- `Ne` → pressure Jacobi

---

## 已知 v4 参数（可调）

```js
SIM_PARAMS = {
  dt: 0.014,             // 模拟步长
  iterations_poisson: 32, // 压力 Jacobi 次数
  mouse_force: 20,        // 鼠标力倍率（byonlab 默认 20）
  cursor_size: 0.2,       // 刷子半径（NDC 空间，0.2 = 20% 屏幕宽）
  isViscous: false,       // 不跑 viscous（byonlab 默认）
}
```

调参建议：
- `mouse_force` ↑ → silk 强度 ↑，但太大会发散
- `cursor_size` ↑ → 刷子范围 ↑，silk 更"宽"
- `iterations_poisson` ↑ → 流体更"可压缩"（数值更稳定），但慢
- `dt` ↑ → silk 速度 ↑，但会不稳定（>0.05 会爆）

---

## 性能
- 1280×720 屏幕 + 模拟分辨率 0.5x → 640×360
- 60fps（M1 Mac / RTX 3060 都轻松）
- 主要瓶颈：32 次 Jacobi 压力迭代（每帧 32 个 fragment pass）

---

## 文件结构
```
byonlab-liquid-metal/
├── index.html          # 单文件所有代码（~21KB）
├── preview-grid.png    # 4 联预览（中心/左/右/下鼠标位置）
├── preview-v12*.png    # 各位置单图
└── analysis.md         # 本文档
```

---

## v3 → v4 关键修复

| 问题 | v3 | v4 |
|---|---|---|
| FS_EXTERNAL_FORCE 不用 uCenter | `(vUv - 0.5)` | `(vUv - 0.5)*2 - uCenter) / uScale` |
| Force pass 方式 | clear+add（错） | additive blend `gl.ONE, gl.ONE` |
| Step 顺序 | advect → clear+add → viscous | advect → force(add) → div → poisson → grad |
| mouse_force | 12 | 20 |
| 调色板范围 | 220-255（太接近 bg） | 208-255（更宽对比） |
| Render | `lenv` 直出 | `pow(lenv, 0.5)` 平方根放大 |
| 自动巡航 lerp | 0.015（太慢） | 0.04 |
| viscous 迭代 | 跑了（浪费） | 不跑（byonlab 默认） |

---

## 参考资料

- Jos Stam 1999, "Stable Fluids"（核心算法）
- Bridson 2015, "Fluid Simulation for Computer Graphics"（教材）
- WebGL2 + RG16F 实现参考：https://github.com/mharrys/fluids-2d
- byonlab bundle 反混淆：`/assets/index-CPqvyJk-.js` 1.3MB
