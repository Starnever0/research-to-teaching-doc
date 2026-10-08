# research-to-teaching-doc

> 把一个主题或已有材料，转化为**面向大众的科普 / 教学向文档** —— 可选同时产出一份**带插图的自包含 HTML 网页**。

这是一个 [WorkBuddy](https://www.workbuddy.cn) Skill：把「调研 → 写成通俗易懂的科普稿 →（可选）做成可分享网页」这条流程固化下来，让每次产出都保持一致的风格与质量。

---

## 它解决什么问题

调研出来的材料往往很"专业"：术语密、数据多、论证长。但**给大众看的东西，目标是"让人理解"，不是"让人信服"**。

这个 skill 负责那一步"降维"：

- **不堆** benchmark 数字表、公式推导、参考文献列表
- **强化** 比喻、对比、以及"为什么这样设计"
- 用**一个贯穿全文的比喻**（如"主角与军师"）把抽象概念讲活
- 主动**串联相似概念**（架构、功能、误区辨析）

## 适用场景

✅ 适合

- 调研某主题并写成**科普 / 教学向**文档
- 把已有技术材料（报告、论文、内部文档、笔记）**降维改写成通俗版**
- 要求"小白也能懂""面向大众""帮读者快速理解"

❌ 不适合

- 需要严谨引用、数据论证、可送审的**技术调研报告 / 学术稿**（那走普通报告流程）

触发词：调研并写成科普、教学稿、科普教学向、通俗易懂、给大众看、小白也能懂、把这份报告改成科普版、写篇科普。

---

## 安装

把本目录放到 WorkBuddy 的**用户级 skills** 目录下（跨项目可用）：

```bash
# macOS / Linux
git clone https://github.com/Starnever0/research-to-teaching-doc.git \
  ~/.workbuddy/skills/research-to-teaching-doc

# Windows (PowerShell)
git clone https://github.com/Starnever0/research-to-teaching-doc.git `
  "$env:USERPROFILE\.workbuddy\skills\research-to-teaching-doc"
```

放置后的目录名需与 skill 名一致（`research-to-teaching-doc`）。

## 使用

直接用自然语言触发即可：

```
调研 Claude Code 的 /advisor 功能，写成一篇科普稿
```

需要网页版时，**明确说一句**：

```
…再配一版网页
…做成可分享的 HTML
--web
```

### 参数

| 参数 | 默认 | 说明 |
| --- | --- | --- |
| （无） | ✅ | **只产出 Markdown 文档** |
| `--web` / "再配一版网页" | ❌ | 额外产出一份自包含 HTML 网页 |

> 设计取向：**默认不生成网页**，避免不必要的产出。未明确要求时，skill 只会询问一句，而不会直接生成。

---

## 产出示例

`assets/sample/` 里放了完整的参考成品（主题：Claude Code 顾问功能）：

| 文件 | 说明 |
| --- | --- |
| [`sample-teaching-doc.md`](assets/sample/sample-teaching-doc.md) | 文档范例 —— 看它的语气、章节骨架与比喻手法 |
| [`sample-teaching-page.html`](assets/sample/sample-teaching-page.html) | 网页范例 —— 看它的插图与排版（单文件、离线可开） |

---

## 目录结构

```
research-to-teaching-doc/
├── SKILL.md                    # 主流程：目标 / 适用 / 输入输出 / 原则 / 步骤 / 参数 / 注意事项
├── references/
│   ├── writing-style.md        # 写作规范：语气、比喻手法、圈层对比法、交付前自检清单
│   └── html-page-spec.md       # 网页规范：设计令牌、组件表、SVG 插图要求、验收清单
└── assets/
    ├── page-template.html      # 可复用网页骨架（含占位符，改内容不动样式）
    └── sample/                 # 参考样例（文档 + 网页）
```

## 标准章节骨架

1. 场景开场 —— 一个大家都会遇到的困扰
2. 一句话说清它是什么 —— 核心比喻 + 角色对照
3. 和熟悉的东西有什么不同 —— 串联相似概念，逐层对比
4. 实际使用 / 表现
5. 背后是怎么运作的 —— 用"发生了什么"讲原理
6. 它聪明在哪 —— 提炼设计亮点
7. 什么时候用 / 别用
8. 要留心的地方 —— 局限与边界
9. 用一句话记住它 —— 收尾金句
10.（可选）延伸小知识 —— 一个概念辨析

---

## 说明

- 网页版**完全自包含**：CSS 与插图（SVG）全部内联，无任何外部依赖，离线双击可打开、可直接分享。
- 科普 ≠ 可以编。产品名、日期、版本、能力边界仍需核实，不确定处会明确标注。
- 具体功能与事实请以官方最新资料为准。

## License

暂未指定。如需开放复用，建议补充一个开源许可证（如 MIT）。
