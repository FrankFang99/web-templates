# 我的网页模板库

> 看到喜欢的网页就拆解 + 复刻 + 归档到这。
> 选项目模板时来这里翻，挑一个顺手的二次改。

---

## 目录约定

```
模板库/
├── README.md            ← 你正在看
├── byonlab-liquid-metal/   ← 模板 001
│   ├── index.html          （单文件 demo，双击直接打开）
│   ├── preview.png         （效果截图）
│   └── analysis.md         （技术拆解 + 调参指南 + 变体）
└── template-002-xxx/       ← 模板 002（待添加）
```

每个模板 = 1 个文件夹，3 个文件：
1. **`index.html`** —— 单文件 demo，浏览器直接看效果
2. **`preview.png`** —— 静态截图
3. **`analysis.md`** —— 拆解 + 怎么改 + 怎么调

---

## 模板索引

| # | 名称 | 效果 | 适用 | 文件夹 |
|---|---|---|---|---|
| 001 | BYON 丝绸布料 v5.2 | 鼠标移动 → 3D 丝绸流体跟随 + Phong 光照明暗 | 浅色 + 高级感首屏 / 创意社区 | [byonlab-liquid-metal](./byonlab-liquid-metal/) |
| 001 | v5.2 4 联预览 | 不同鼠标位置 + 3D 立体效果 | 看不同视角 | [preview-grid-v2.png](./byonlab-liquid-metal/preview-grid-v2.png) |
| 001 | v5.2 vs byonlab 原版对比 | 进步路线 v4 → v5.1 → v5.2 | 看 3D 突破 | [preview-compare-v2.png](./byonlab-liquid-metal/preview-compare-v2.png) |

---

## 怎么加新模板

每次看到喜欢的网页，告诉我：
1. **URL**（必填）
2. **哪个效果最戳你**（鼠标交互 / 滚动动效 / 3D / 文字效果 ...）

我会自动：
- 抓网页分析（截图 + 提取技术栈 + 定位关键代码）
- 写单文件 demo（npm-free，直接双击打开）
- 写 analysis.md（怎么改 + 怎么调参数 + 变体）
- 加进本 README 索引

---

## 怎么用模板

**1. 看效果** — 双击 `index.html` 在浏览器打开
**2. 复制到项目** — 把 `index.html` 内容贴进你的项目，或用 `<iframe>` 嵌入
**3. 改样式 / 调参数** — 参考 `analysis.md` 的"调参速查"和"变体"章节
**4. 改文案 / 改主题色** — 改 `index.html` 顶部的 CSS 变量即可

---

## 风格倾向（我喜欢的）

- **暗色宇宙 + 巨型渐变文字**（Vite portfolio 那种）—— 适合：作品集 / 投资路演
- **浅色银白 + 液态金属**（BYON 那种）—— 适合：创意社区 / 设计师主页
- **彩色流动渐变**（Apple 产品页）—— 适合：产品落地页
- **极简白底 + 大留白**（Linear）—— 适合：SaaS / 工具产品

看到新的好效果就加进库。
