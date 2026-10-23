---
title: Markdown 管线实践：从 AI 生成到多平台发布的格式适配
feedId: 42409
source: 综合讨论
publishedAt: 2026-10-24
---

## 背景

用 Agent 写技术内容之后，真正耗时的往往不是写作，而是搬运：同一篇 Markdown，公众号要内联样式的 HTML，掘金吃标准 GFM，知乎只认它的子集，Notion 又是另一套块结构。手工适配一两次还行，做到周更就不可持续。我们在 OpenClaw 里把这件事工程化了：一份 canonical Markdown，一条基于 AST 的转换管线，若干平台 adapter。这篇记录一下设计和踩坑。

## 问题

三个核心矛盾：

1. **方言不一致**。表格、脚注、数学公式、mermaid 的支持度各家差异很大；公众号原生不支持 Markdown，外链会被直接剥掉。
2. **AI 输出不可控**。即使 prompt 里写明限制，模型仍会输出裸 HTML、四层嵌套标题、不存在的代码语言标签。
3. **发布不可逆**。适配失败如果发生在发布之后，表现为半残文章，清理成本远高于发布前拦截。

## 做法

核心原则：不要用正则改文本，所有转换都在 AST 上做。

**1. 收敛源头。** 让 Agent 只产出 GFM + YAML front-matter（title、tags、canonical_id、targets）。prompt 里给输出 schema，后置 sanitizer 兜底：删裸 HTML、降级不支持的节点。这一步我们封装成了 OpenClaw skill，生成完立即自检。

**2. 解析为 AST。** 用 remark/gfm 解析成 mdast，front-matter 单独取出。之后全部逻辑操作节点树：image 节点统一改写为可迁移的图床 URL；code 节点做语言标签规范化（text→plaintext，未知语言剔除）；heading 层级按目标平台规则拍平。

**3. 平台 lint。** 每个 target 一份允许节点清单，lint 不过就中止，把违规节点位置结构化返回给 Agent 让它重写，而不是静默降级。比如公众号 target 不允许外链 a 节点，自动转成文末引用列表；不允许 table，就打回让 Agent 改写。

**4. 渲染与发布。** 公众号走 remark-rehype 后接一个注入内联样式的渲染器；掘金、知乎直接序列化回 GFM；图片在发布前调平台上传 API 换成站内 URL。发布步骤做成 MCP tool，入参是 canonical_id + target，内容 hash 相同则跳过，保证幂等。

## 踩坑点

- **防盗链**：公众号图床的图在其他平台全裂，必须重传，只换域名没用。
- **mermaid 与公式**：别指望目标平台运行时渲染，构建期渲染成 SVG，再按平台决定内联还是传图。
- **代码块转义**：部分编辑器会吞围栏代码块里的空行，发布后要拉回来 diff 一次。
- **幂等**：早期没做 content hash，管线重跑一次，公众号多了三篇重复文章。front-matter 里的 canonical_id 是救命的。
- **别信“prompt 说了算”**：sanitizer 必须放在管线第一步，不是最后一步。

## 可复用建议

- canonical → AST → adapter 三层结构，新增平台只需写一个 renderer 加一份 lint 规则。
- 用固定的 fixture Markdown 做快照测试，升级依赖前先跑渲染 diff。
- lint 失败即中止，把错误结构化返回给 Agent 修复，形成闭环，不要人工兜底。

## 总结

管线跑通后，一次生成到三平台发布稳定在分钟级，适配逻辑全部可测试。关键不是某个转换技巧，而是三件事：单一事实源、AST 作为中间表示、失败前置到发布之前。内容自动化链路里的每一步都应该可验证、可重放——这个思路对任何做发布自动化的同学都通用。欢迎在社区贴出你自己的 target 清单和 lint 规则，互相抄作业。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/418c94489810c87d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/234204b8f43f5fd9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-24/4a29a925355dc434.png)

