---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 42556
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

用 OpenClaw 这类 Agent 产出内容时，Markdown 是事实上的中间格式。但写完只是第一步：公众号要内联样式的 HTML，知乎、掘金吃受限的 Markdown，静态站要 frontmatter，RSS 要 HTML 摘要。同一份内容手动改三遍，迟早出错。

## 问题

两个层面：

1. **AI 输出不规范**：标题层级漂移、代码块缺语言标记、偶尔混入 HTML、表格和 mermaid 时有时无。
2. **平台方言差异**：公众号过滤 class、锚点失效、外链受限；各平台对 GFM 表格、任务列表的支持也不一致。

如果让 Agent 直接"生成公众号版"，等于把格式决策交给非确定性采样，同样的输入下次输出可能不同。格式转换应该收敛到确定性代码里。

## 做法

核心原则：**一份 canonical Markdown，N 个确定性 adapter**。

1. **规范化**：Agent 只负责产出 canonical 层——统一 frontmatter（title/tags/summary/cover）、标题从 h2 起、代码块补语言标记。写一个 mdast linter 在入库前校验，不合规打回让 Agent 重写。
2. **AST 转换，不用正则**：用 unified/remark 走 mdast，转换逻辑写成插件链。比如公众号 adapter：脚注转文末引用列表、外链改为"链接见文末"、GFM 表格转内联样式 HTML 片段。
3. **渲染与样式**：mdast → HTML 后用 juice 把 CSS 内联进标签，公众号只认 style 属性。每个平台一份样式表进仓库，可版本化。
4. **资产处理**：图片先传目标图床，拿到稳定 URL 后回填 Markdown，映射关系存 JSON，重发不重复上传。
5. **校验**：dry-run 渲染每个目标，做链接检查、图片可达性检查、渲染快照 diff，通过后再真发。Agent 侧只暴露一个 `publish(target)` 的工具，内部就是这条管线。

## 踩坑点

- 正则替换在含 ``` 或分隔线的嵌套代码块里必炸，切 AST 后才稳。
- 公众号会把 `<pre>` 的样式洗掉，要么逐行内联，要么退化为平台自带代码块。
- mermaid 多数平台不渲染，管线里预渲染成 SVG/PNG 再插图。
- 源文件硬换行会导致中文段落断裂，linter 里直接禁掉 80 列折行。
- remark 插件顺序敏感（gfm 要在 heading-id 之前），锁版本 + 快照测试兜底。

## 可复用建议

- 永远不要为每个平台 fork 内容，只 fork adapter。
- 转换规则写成可单测的纯函数 + 快照用例；平台改版时改 adapter，不动内容。
- 划清 Agent 职责边界：写 canonical 内容、调用工具，格式决策全部收进管线。
- 留一份 build log（源文件 hash → 各平台发布记录），排查"线上版本和我本地不一样"时救命。

## 总结

这条管线的本质是把不确定性关进笼子：AI 负责内容，代码负责格式。canonical Markdown 是唯一事实源，adapter 是可测试的纯函数层。搭建成本大概两三天，之后每接入一个新发布平台，就只是多写一个 adapter。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/4408dda92fdc0576.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/1d93637e6953c868.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/19c61d2fd7bfd9b5.png)

