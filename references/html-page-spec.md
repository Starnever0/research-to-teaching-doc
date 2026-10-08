# 网页版设计规范

本文件定义 `research-to-teaching-doc` 在生成 HTML 网页版时的硬性要求、设计令牌、插图规范与验收清单。做网页时读。

网页版默认基于 `assets/page-template.html` 修改：**替换内容、增删插图**，不要从零重写样式。

---

## 1. 硬性要求

1. **自包含单文件**：CSS 与插图（SVG）全部内联，**不得有任何外部依赖**（无 CDN、无外链字体、无外链图片）。目标：离线双击可打开、微信 / 邮件直接发文件即可阅读。
2. **内容与文档版对齐**：网页版是文档版的"视觉增强版"，章节与观点一致；只允许在措辞上更紧凑。
3. **插图必须自己画**：所有图示用**内联 SVG**手绘，不引用外部图片。插图用于把抽象概念变直观，而非装饰。
4. **中文排版**：使用系统中文字体栈，保证跨设备不塌。
5. **篇幅控制**：网页版与文档版同量级，阅读时长 8–12 分钟。

## 2. 设计令牌（直接复用模板中的 CSS 变量）

| 用途 | 变量 | 值 |
| --- | --- | --- |
| 正文墨色 | `--ink` | `#24241F` |
| 次级文字 | `--ink-soft` | `#5E5E56` |
| 弱化文字 | `--ink-faint` | `#8E8E84` |
| 纸张底色 | `--bg` | `#F6F4EF`（暖白纸感） |
| 卡片底色 | `--paper` | `#FFFFFF` |
| 分隔线 | `--line` | `#E7E3D9` |
| 蓝（信息 / 主角） | `--blue` / `--blue-bg` / `--blue-line` | `#1B5FA5` / `#E8F1FB` / `#B9D5F2` |
| 紫（重点 / 军师） | `--purple` / `--purple-bg` / `--purple-line` | `#524AB4` / `#EEEDFD` / `#CAC6F0` |
| 琥珀（注意） | `--amber` / `--amber-bg` / `--amber-line` | `#8A5408` / `#FAEEDA` / `#EFD39B` |
| 青（正面） | `--teal` / `--teal-bg` / `--teal-line` | `#0E6B54` / `#E2F5EE` / `#A9DECB` |

字体栈：

```
-apple-system, BlinkMacSystemFont, "Segoe UI", "PingFang SC",
"Hiragino Sans GB", "Microsoft YaHei", "Source Han Sans SC", sans-serif
```

## 3. 版式规则

- **正文宽度 800px 居中**，左右内边距 ≥ 24px。
- **行高 1.85**，正文字号 16px —— 长文不累眼。
- 标题层级：`h1` ≈ 38px（仅 hero）、`h2` ≈ 24px（带编号徽章）、`h3` ≈ 18px。
- **章节编号徽章**：`h2` 前置一个圆角方块，内为数字，用蓝色系。
- **顶部 hero**：kicker 标签（圆角胶囊）+ 大标题 + 一句副标题 + 元信息（日期 / 阅读时长）。
- **目录**：`<nav class="toc">`，两列（窄屏降为一列），锚点可点击跳转；`section` 设 `scroll-margin-top`。
- **首屏不要堆内容**：hero 之后紧跟目录，再进正文。

## 4. 可复用组件

| 组件 | class | 用途 |
| --- | --- | --- |
| 角色对照卡 | `.cards` + `.card.blue` / `.card.purple` | 两栏并排对比（主角 vs 军师） |
| 提示条 | `.note` / `.note.info` / `.note.warn` | 补充说明 / 关键提醒 |
| 强调金句块 | `.banner` | 开篇定义、结尾总结 |
| 编号步骤 | `.steps` + `.step` | 多步流程 |
| 插图容器 | `figure` + `.fig` + `figcaption` | 包住 SVG，配图注 |
| 对比表 | `.table-scroll` + `table` | 概念并列对比 |
| 比例条 | `.ratio` + `.ratio-bar` | 可视化"占比 / 分配"类概念 |
| 适合 / 不适合 | `.good-bad` + `.col.yes` / `.col.no` | 双栏色块 |
| 代码块 | `pre > code` | 命令与配置；行内用 `code` |

## 5. 插图（内联 SVG）规范

- 每个 SVG 加 `role="img"`，首两个子元素为 `<title>` 与 `<desc>`（可访问性）。
- 根 `<svg>` 设 `viewBox`，并由 CSS `width:100%; height:auto` 自适应；**不要写死像素宽高**。
- 配色**只用上面的令牌色**（浅底 + 同色系描边 + 深色文字），保持全页统一。
- **箭头**：在 `<defs>` 里定义一次 `marker`；每条用作连接线的 `<path>` 必须 `fill="none"`。
- **文字**：`text` 加 `dominant-baseline="central"`；字号不小于 12px。
- **每个图示配一句 `figcaption`**，解释这张图在说什么。
- 建议 3–5 张插图，覆盖：① 角色对照 ② 旧范式结构 ③ 新范式反转 ④ 流程 / 占比 等抽象点。
- 忌讳：装饰性花哨图形、渐变滥用、深色背景铺满。

## 6. 响应式

- 视口 meta：`width=device-width, initial-scale=1.0`。
- 两栏布局（`.cards`、`.good-bad`、表格）在 `max-width:620px` 时降为单栏。
- 宽表格用 `.table-scroll` 横向滚动，不撑破页面。

## 7. 交付前自检清单

- [ ] 单文件、无任何外部依赖（`grep` 检查无 `http://` / `https://` 外链资源）
- [ ] 标签配对完整（`svg` / `section` / `div` / `table` / `figure` 等开闭平衡）
- [ ] 顶栏 hero + 可点击目录齐备
- [ ] 至少 3 张内联 SVG 插图，均有 `title` / `desc` / `figcaption`
- [ ] 文字与底色对比充分（浅底深字），无深色铺底
- [ ] 窄屏下单栏降级正常
- [ ] 章节与文档版一致，只做了视觉增强
- [ ] 用 `present_files` 展示，并勾选预览
