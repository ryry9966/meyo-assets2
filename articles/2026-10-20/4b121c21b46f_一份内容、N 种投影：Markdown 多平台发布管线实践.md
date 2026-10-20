---
title: 一份内容、N 种投影：Markdown 多平台发布管线实践
feedId: 42112
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

Agent 的产出天然是 Markdown：结构清晰、可版本化、易被下游解析。但在 OpenClaw 的自动化实践里，"生成"只是第一公里，摩擦发生在发布端——公众号、知乎、掘金、静态站点、RSS，每家一套方言：公众号吃 HTML 还要内联样式，知乎不支持脚注和部分表格语法，静态站要 frontmatter，图片链路各家自理。让 Agent 生成完直接"贴"出去，第二个平台就会开始崩。

## 问题

把一份 AI 生成的 Markdown 投到 N 个平台，工程上至少四类麻烦：

1. **方言差异**：GFM、脚注、Mermaid、数学公式，各平台支持度参差不齐，且大多是静默降级——不报错，悄悄丢内容。
2. **资产处理**：图片本地路径在所有平台上都是死路，公众号还要求白名单域名，二次上传会改变 URL。
3. **结构污染**：frontmatter、Agent 偶尔带出的对话腔（"以下是我为您生成的内容"）会直接漏进正文。
4. **幂等性**：重跑一次发布，产出重复文章和重复图片。

## 做法：一条五步管线

核心思路：**Agent 只负责内容，格式适配交给确定性代码**。

1. **单一事实源**。约定一份 canonical Markdown：统一 frontmatter schema（title、slug、tags、cover），语法收敛到各平台支持的交集，平台特性作为可选增强节点单独标记。
2. **归一化层**。用 remark/unified 解析成 AST 后做预处理：剥离对话腔、Mermaid 渲染成图片、脚注转内联、规范中英文间距。所有操作在 AST 上做，不要对原始文本跑正则。
3. **平台适配器**。每个适配器是一个纯函数：canonical AST → 平台 payload。公众号输出内联样式的 HTML，静态站输出带 frontmatter 的文件，掘金/知乎走各自 API。纯函数可独立测试。
4. **资产管线**。发布前统一把图片传到对象存储或平台素材库，回写映射表，重写路径，保留 alt 文本。
5. **发布与校验**。用 MCP 工具包一层，比如 `markdown.publish(platform, slug, dry_run)`。dry-run 返回机器可读的报告：哪些节点会被转换、哪些会被丢弃。slug → 平台文章 ID 的映射落一个 JSON 文件，保证重跑幂等。

Agent 在这条管线里只做两件事：产出符合 schema 的 canonical Markdown，以及根据 dry-run 报告决定是否发布。

## 踩坑点

- **对 Markdown 跑正则**。代码块里的 `#`、嵌套列表、转义字符都会翻车，AST 是底线。
- **静默降级比报错更危险**。脚注和公式被吞掉时没有任何提示，读者看到的是残缺正文。管线遇到不支持的节点应显式报错或进 dry-run 报告。
- **公众号的 HTML 消毒**。class 大面积被剥，样式必须内联；外链图片因 referer 防盗链裂图，必须走素材上传接口。
- **重跑不幂等**。没有映射表，重跑一次素材库就多一批图片，账号很快变垃圾场。
- **Agent 输出超 schema**。prompt 里必须严格定义输出契约；归一化层兜底清理可以，但别当常态，成本更高。

## 可复用建议

- canonical schema 就低不就高：按支持度最低的平台收敛语法，增强特性标记后由适配器按需开启。
- 适配器保持纯函数，配 fixture 测试；发布连接器写薄一点，只做鉴权和请求。
- dry-run 是给 Agent 看的接口，报告结构要稳定、可解析。
- 依赖锁版本。remark 插件升级偶发行为变化，发布管线不应追新。

## 总结

多平台发布的本质是"一份内容、N 种投影"。把投影逻辑从 Agent 手里拿走，收敛成 AST 之上的纯函数适配器，配上 dry-run 和幂等映射，管线就从"每次手糊"变成可测试的确定性代码。Agent 负责写得对，管线负责投得准——分工清楚之后，接入一个新平台只是多写一个适配器的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/430d9870c34b363d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/757a8088faa093c5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/e5954d6e2fb44d23.png)

