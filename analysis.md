# 模板 001 · BYON 丝绸布料（Silk Fabric）

> **源参考**：[byonlab.com/#/community](https://byonlab.com/#/community)
> **类别**：3D Shader / 鼠标交互 / 顶点位移
> **基调**：浅色 / 银白丝绸 / 柔光
> **文件**：`index.html`（11 KB，单文件可运行）

---

## 一、视觉效果

**鼠标移动 → 丝绸布料在鼠标位置被按下，涟漪扩散，整张布有自然流动的褶皱。**
- 鼠标静止时：布料缓慢呼吸（FBM 噪声 + 时间偏移）
- 鼠标移动：布料跟随鼠标产生"被按下去"的凹陷 + 涟漪扩散
- 离开鼠标：布料平滑回弹（lerp 跟随，不是瞬移）

> 截图：`preview.png`（鼠标在 (0.5, 0.5) 中心位置，能看到中心偏下的下压效果）

---

## 二、关键技术

| 技术 | 用途 |
|---|---|
| **Three.js (CDN + importmap)** | WebGL 渲染 |
| **透视相机 + PlaneGeometry(10, 6, 256, 192)** | 高细分平面 = 49,152 顶点，足够细腻 |
| **Vertex Shader 顶点位移** | z = FBM 噪声 + 鼠标 Gaussian 下压 + 涟漪 |
| **Fragment Shader 半 lambert** | 软光照，无硬阴影，silk 质感关键 |
| **#define SEG_W/H + float() cast** | 把 JS 细分参数传入 GLSL（注意 GLSL 不允许 float/int 互除） |
| **鼠标 lerp 跟随** | `uMouse += (target - uMouse) * 0.08` —— 跟手但不抖 |

### 为什么不是 metaball？
- byonlab 原始是**一张连续布料**，不是分离的球
- 表面连续 → 顶点位移 + 软光照
- 鼠标按下去 → Gaussian 下压 + 涟漪
- metaball（我 v1 用的）做出来是"金属球融合"，完全不同的视觉

---

## 三、代码结构

```
index.html
├── <style> HUD（导航 + 标题 + 输入框）
├── <canvas id="c">  全屏 WebGL 画布
├── <script type="importmap">  Three.js CDN 引用
└── <script type="module">
    ├── 1. 场景搭建（renderer / scene / 透视 camera / plane）
    ├── 2. 鼠标追踪（mousemove → uMouseTarget, lerp 到 uMouse）
    ├── 3. uniforms（uTime / uMouse / uColorBase / uMouseStrength）
    ├── 4. Vertex Shader
    │   ├── hash + value noise + FBM（4 层）
    │   ├── displacement = baseFolds + mouseDip + ripple
    │   ├── 法线 = 偏导数（e=0.003 邻域采样）
    └── 5. Fragment Shader
        ├── 半 lambert（柔和）+ view dot（顶光）+ 微高光
        ├── 鼠标周围高光环（突出交互点）
    └── 6. 动画循环（uMouse lerp + 时间 + 渲染）
```

---

## 四、调参速查

| 想要效果 | 改什么 | 推荐值 |
|---|---|---|
| 鼠标按得更深 | `mouseDip` 系数 ↑ | 0.8 ~ 1.5 |
| 鼠标按得更浅 | `mouseDip` 系数 ↓ | 0.3 ~ 0.5 |
| 鼠标涟漪更强 | `ripple` 系数 ↑ | 0.15 ~ 0.3 |
| 涟漪更慢 | `uTime * 3.0` ↓ | 1.5 ~ 2.5 |
| 褶皱更密 | `fbm(uv * 2.0)` 系数 ↑ | 3.0 ~ 5.0 |
| 褶皱更疏 | `fbm(uv * 2.0)` 系数 ↓ | 1.0 ~ 1.5 |
| 褶皱幅度大 | baseFolds 系数 ↑ | 0.4 ~ 0.6 |
| 褶皱幅度小 | baseFolds 系数 ↓ | 0.1 ~ 0.2 |
| 鼠标跟手更紧 | `uMouse += ... * 0.08` ↑ | 0.12 ~ 0.18 |
| 鼠标跟手更"重" | 系数 ↓ | 0.03 ~ 0.06 |
| 改主色 | `uColorBase` / `uColorShadow` | 看主题 |
| 改暗色版 | 整体调暗，`uColorBase: '#3a3548'`, `uColorShadow: '#0a0c14'` | 宇宙感 |
| 改金色 | `uColorBase: '#f0d878'`, `uColorShadow: '#8a6d20'` | 丝绸金 |

---

## 五、变体（改几行就能变样）

### 变体 A：暗色丝绸（宇宙感）
```js
body { background: linear-gradient(180deg, #0a0c14 0%, #050608 100%); }
uColorBase: '#3a3548'
uColorShadow: '#0a0c14'
uColorHighlight: '#a8b3ff'  // 蓝光高光
```

### 变体 B：彩色丝绸
```js
uColorBase: '#ffd6e0'   // 粉色
uColorShadow: '#5a3a4a'
uColorHighlight: '#ffffff'
```

### 变体 C：金属丝绸（混合我 v1 的金属感）
```js
uColorBase: '#c8cdd4'
uColorShadow: '#3a4250'
uColorHighlight: '#ffffff'
// fragment 里把 spec 系数从 0.25 提到 0.8
```

---

## 六、避坑（踩过的坑）

| 问题 | 解决 |
|---|---|
| shader 编译失败：`SEG_W undeclared` | JS 常量要通过模板字符串 `${SEG_W}` 嵌入 shader 顶部 + `#define` |
| shader 编译失败：`float / int` | GLSL 严格类型，**必须显式 `float(SEG_W)`** |
| 顶点位移太小看不出 | 调大基础位移系数（0.3 → 1.5+） |
| 褶皱太密集像噪点 | 降低 `fbm(uv * 2.0)` 系数，疏一些 |
| 鼠标下压太弱 | `mouseDip` 系数 + 减 `exp(-dist² * 4.0)` 衰减率 |
| 看起来像金属球而不是布 | metaball 改 plane 顶点位移（彻底换思路） |
| 性能掉 | 256x192 = 49k 顶点已够；想更流畅降到 192x144 |
| PlaneSize 太大超出视野 | 调小 plane（10x6），camera 距离 4.5 |

---

## 七、可复用函数

### 1. FBM 噪声（4 层）
```glsl
float hash(vec2 p) {
  p = fract(p * vec2(123.34, 456.21));
  p += dot(p, p + 45.32);
  return fract(p.x * p.y);
}
float noise(vec2 p) {
  vec2 i = floor(p), f = fract(p);
  vec2 u = f * f * (3.0 - 2.0 * f);
  return mix(mix(hash(i), hash(i + vec2(1,0)), u.x),
             mix(hash(i + vec2(0,1)), hash(i + vec2(1,1)), u.x), u.y);
}
float fbm(vec2 p) {
  float v = 0.0, a = 0.5;
  for (int i = 0; i < 4; i++) {
    v += a * noise(p);
    p *= 2.07;
    a *= 0.5;
  }
  return v;
}
```

### 2. 顶点法线（用邻域位移算偏导）
```glsl
float e = 0.003;
float dX1 = displacement(uv + vec2(e, 0.0));
float dX0 = displacement(uv - vec2(e, 0.0));
vec3 tangentX = vec3(2.0 * e * (size.x / float(SEG_W)), 0.0, dX1 - dX0);
// 同样算 tangentY
vec3 normal = normalize(cross(tangentX, tangentY));
```

### 3. 鼠标 Gaussian 下压
```glsl
vec2 d = (uv - mouseUV) * vec2(aspect, 1.0);
float dist = length(d);
float dip = -strength * exp(-dist * dist * 4.0);
```

---

## 八、参考

- IQ SDF 教程：<https://iquilezles.org/articles/distfunctions/>
- The Book of Shaders（FBM）：<https://thebookofshaders.com/13/>
- Three.js 入门：<https://threejs.org/docs/index.html#manual/en/introduction/Installation>
- byonlab 源参考：<https://byonlab.com/#/community>
