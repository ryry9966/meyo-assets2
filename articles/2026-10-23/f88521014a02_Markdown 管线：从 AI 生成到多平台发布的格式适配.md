---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 42399
source: 综合讨论
publishedAt: 2026-10-23
---

## 背景

用 Agent 产出技术内容之后，真正的麻烦才开始：同一篇 Markdown 要发到静态博客、公众号、掘金、知乎、GitHub，各平台对 Markdown 的“方言”和限制完全不同。手动搬运改格式一两次还行，变成日常节奏后就是纯体力活，还容易出错。这篇记录我把这条链路自动化的一版做法：OpenClaw 的 Agent 挂几个 MCP 工具，跑通“生成 → 校验 → 适配 → 发布”。

## 问题拆开看

- **方言差异**：公众号只吃内联样式的 HTML，会剥掉 class；掘金/知乎是 Markdown 子集；静态站则需要 frontmatter。
- **元素支持**：脚注、公式、TOC、任务列表，各平台支持参差。
- **资产**：图片相对路径发出去就挂；图床防盗链、重复上传。
- **回归**：平台规则会悄悄变，没有校验手段就只能上线后靠肉眼发现。

## 做法：一份标准，多个适配器

**第一步：内容契约。** 约束 AI 只输出 GFM + YAML frontmatter（title/description/tags/date），prompt 中明确禁止内嵌 HTML。生成后先跑 lint（remark-lint + 自定义规则），不合格直接打回重写，不进下游。

**第二步：统一变换层。** 基于 unified/remark 组织管线，每个平台适配器就是一组纯函数插件：

```js
// 简化示意
const tree = unified().use(remarkParse).parse(md);
const out = await platformAdapters[target](tree, ctx);
```

适配器只做三件事：删/替不支持的节点、改写资产链接、输出目标格式（公众号走内联样式 HTML，博客基本保留原样）。

**第三步：资产处理。** 发布前用 `upload_assets` 工具把本地图片传到目标可用的图床，按内容哈希去重，回写 URL。这一步独立于格式变换，所有 target 复用。

**第四步：dry-run 优先。** 每个目标先产出预览文件落盘，Agent 汇报差异点（例如“公众号版丢弃了 2 个脚注”），人工确认后才真正发布。发布动作做幂等：同一 hash + target 不重复推。

## 踩坑点

1. **别信 AI 的 Markdown。** 即便 prompt 写了约束，模型偶尔仍会吐 HTML 或半角全角混排。lint 是底线，不是可选项。
2. **公众号代码块高亮**：class 全被剥掉，只能构建时做 token 级内联着色；嵌套列表和换行也会出幺蛾子，适配器里要有兜底转换。
3. **公式**：博客用 KaTeX，知乎用自家语法，公众号干脆不支持——我们统一降级为公式转图片，省心。
4. **frontmatter 泄漏**：忘剥 frontmatter 直接发出去，是真实发生过的事故，现在 lint 里有对应检查。
5. **平台规则漂移**：一次公众号更新后代码块行高崩了。解法是给每个 target 存渲染快照，CI 里做 diff。

## 可复用建议

- 单一事实源：canonical Markdown 只有一份，所有差异都进适配器，适配器保持薄。
- 平台适配器做成可分享的 remark 插件包，各目标平台由使用者自行维护，比集中维护现实得多。
- 发布幂等 + 产物留档（每篇按 target 存目录），排查问题有据可查。
- lint 规则跟着事故走：每次踩坑转化为一条规则，管线越跑越硬。

## 总结

这条管线没有黑科技，核心三点：用契约收敛上游不确定性，用纯函数适配器隔离平台差异，用 dry-run 和快照兜住回归。目前从生成到三平台发布一次不到两分钟，人工只剩最后确认。后续打算把适配器做成可共享的插件集，并补一版渲染质量的自动化评测。欢迎在社区里贴出你们各自平台的适配器实现，对齐一下各自的坑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/eab0dfd658ff0e19.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/8f259ae6d718a7e1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-23/77563caa0dba8a90.png)

