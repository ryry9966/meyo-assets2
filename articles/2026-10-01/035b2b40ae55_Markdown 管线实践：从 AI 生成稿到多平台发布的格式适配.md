---
title: Markdown 管线实践：从 AI 生成稿到多平台发布的格式适配
feedId: 39905
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

在 OpenClaw 这类 Agent 工作流里，Markdown 是事实上的中间格式：模型输出、文档、发布稿都长一个样。但写完不等于发完——公众号、知乎、掘金、静态站对 Markdown 的支持差异很大，真正吃时间的是最后那步“格式适配”。当发布交给 Agent 自动化时，这些差异就从偶发排版问题变成了必现的发布故障。

## 问题

典型痛点：

- 公众号编辑器基本剥掉外部样式，脚注、任务列表直接失效；
- 知乎对表格和 HTML 混排限制多，复杂表格渲染不可控；
- 代码块语言标签（```ts vs ```typescript）各平台解析不一致，高亮时好时坏；
- 图片本地路径失效，各平台图床策略不同；
- 硬换行语义不统一，有的平台当 `<br>`，有的当空格。

## 做法

核心思路：**单一事实源 + AST 级转换 + 平台适配器**。

1. **源文档带 frontmatter**（title、tags、目标平台列表），Agent 生成时只写规范 GFM，不碰任何平台私有语法。
2. **用 unified/remark 解析成 mdast**，所有转换在 AST 上做。每个平台一个 adapter，声明式描述“支持什么、降级什么”：公众号先转 HTML 再走 inline-style 渲染器，脚注降级为括号文本；知乎表格降级为列表；静态站原样输出并补 slug 和 TOC。
3. **图片统一 rehost**：管线先把图片上传到稳定图床，重写 AST 里所有 URL，再分发。
4. **dry-run 校验**：对每个目标平台跑一次渲染快照，diff 后人眼确认，再推 API 或写剪贴板。在 OpenClaw 里可以把整串步骤挂成一条 hook 链：generate → lint → transform → preview → publish。

## 踩坑点

- 最大的坑是用正则/字符串替换做转换：嵌套引用、代码块里的示例语法都会被打碎。必须在 AST 层做。
- 公众号别指望保留任何 class，只能 inline style；颜色字号收敛成一套受限 token，否则每次渲染结果都不一样。
- Agent 生成时统一用空行分段，不要依赖单个换行，换行语义在平台间不可移植。
- URL 或文件名里的下划线、星号会被误解析，transform 里注意 escape。
- 快照测试要包含“最恶心的那篇文档”当固定 fixture，长表格、脚注、公式都塞进去，否则回归测不出来。

## 可复用建议

- 平台差异做成配置化的 profile，而不是散落在代码里的 if，新平台加一个 profile 即可。
- 转换函数保持幂等：对输出再跑一遍不产生变化，重试才安全。
- 源文档永远不改写，改写只发生在渲染产物上，随时可重新生成。
- 维护 10–20 篇覆盖各种极端语法的语料集，当作管线的测试基准。

## 总结

Markdown 管线的本质是“一次创作，多端编译”。把格式适配从人肉复制粘贴变成 AST 级的确定性转换，Agent 自动发布的成功率就从碰运气变成了可测试的工程问题。建议先跑通一个平台的最小链路，验证 dry-run 闭环，再横向加 profile。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/b4f74981fd17673d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/935aab777c357e62.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/efe84c68c68a3931.png)

