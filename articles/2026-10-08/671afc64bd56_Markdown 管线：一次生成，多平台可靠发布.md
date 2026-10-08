---
title: Markdown 管线：一次生成，多平台可靠发布
feedId: 40921
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

用 Agent 产出技术内容的人，几乎都会卡在同一处：模型生成的 Markdown 本身没问题，但发到公众号、知乎、掘金、静态博客时，每个平台的渲染规则都不一样。公众号基本只吃内联样式的 HTML；知乎对表格和代码块有自己的裁剪逻辑；静态博客需要 front matter。如果 Agent 的工具链里只有一个笼统的“发布”动作，最后必然退化成手工复制粘贴。

## 问题

核心矛盾是：AI 生成端追求自由表达，发布端却是多个受限的格式环境。常见的错误做法有两种：一是在提示词里反复强调“请输出适配公众号的格式”，效果不稳定；二是为每个平台单独生成一份内容，同一主题出现多个事实源，后续修改必乱。

## 做法

思路是把发布从 Agent 的职责里拆出来，做成独立的 Markdown 管线，Agent 只负责产出规范化的“单一事实源”。

**1. 定义受限规范（canonical Markdown）**

约束输入子集：标准标题层级、fenced code、表格、图片引用，禁止内嵌 HTML。用 markdownlint 固化规则，Agent 产出后先过 lint，不合格打回重写，而不是靠提示词赌运气。

**2. AST 中间层做转换**

用 unified/remark 解析成 MDAST，所有平台差异都在 AST 层面处理，不要用正则硬替换：

```
canonical.md → remark-parse → transform 链 → 各平台 serializer
```

**3. 平台适配器（adapter）**

每个平台一个 adapter，职责包括：front matter 剥离或改写、代码块转高亮 HTML（rehype-highlight 后把 class 折叠成 inline style）、表格降级或转图片、锚点重写、标点清洗。

**4. 图片资产处理**

所有外链图片先下载、压缩、转存到目标平台或自有图床，替换引用后再渲染。这步必须幂等：按内容哈希判断是否已上传，重跑管线不重复上传。

**5. 校验与发布**

发布前强制 dry-run：渲染成 HTML 后截图做视觉检查。接进 Agent 时把管线暴露为 MCP tools——`lint_markdown`、`render_for(platform)`、`publish(platform, dry_run)`，让模型自主走“生成 → lint → 预览 → 确认 → 发布”的闭环。

## 踩坑点

- **别信提示词约束**。模型迟早会输出内嵌 HTML 或奇怪嵌套，lint + transform 才是兜底，提示词只是减少返工。
- **公众号图片防盗链**。外链图不转存必挂，且上传后 URL 会变，引用替换要在转存之后做。
- **代码高亮 class 会被剥**。多数平台粘贴 HTML 时丢弃 class，必须折叠成 inline style。
- **表格是重灾区**。知乎丢列宽，公众号宽度溢出，超过 3 列的表建议直接渲染成图片。
- **软换行差异**。公众号对单个换行敏感，serializer 里要显式控制段落间距。
- **front matter 残留**会被某些平台渲染成表格，dry-run 截图能兜住这类问题。

## 可复用建议

- 单一事实源 + adapter 模式，平台改版只改对应 adapter，管线主体不动。
- 保留三级中间产物：canonical.md、rendered.html、preview.png，出问题时可逐层回溯。
- publish 工具默认 `dry_run=true`，二次确认后才真正提交，避免 Agent 半自主状态下误发。
- 给 adapter 加版本号，发布记录带上版本，方便排查“平台哪天改版导致渲染劣化”。

## 总结

多平台发布的问题本质不是“AI 写得好不好”，而是格式适配有没有被工程化。把平台差异收敛到 AST 层的 adapter 里，把管线工具化成 MCP 接口，Agent 就能从“会写”进化到“能可靠地发出去”。这条管线代码量不大，remark 生态基本覆盖全部转换需求，值得每个做内容自动化的团队先建后用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/4c12bc7b8f7ce896.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/b79d63112e984cca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/e81ff6bc192160fc.png)

