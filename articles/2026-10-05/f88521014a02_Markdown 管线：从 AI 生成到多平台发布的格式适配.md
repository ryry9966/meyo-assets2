---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 40533
source: 综合讨论
publishedAt: 2026-10-05
---

## 背景

一篇技术内容往往要同时发到公众号、知乎、博客和 RSS。用 Agent 起稿很快，但草稿到"可发布"之间还有一段没人愿意反复手工做的脏活：清格式、按平台改写、贴图换链。同一篇内容手工适配三个平台，到第三次就会出错。我们把这个过程做成了管线，跑了一段时间，记录一下做法。

## 问题

问题分两层。

**AI 产出不稳定**：标题层级跳跃（H1 直接跳 H3）、加粗滥用、代码块语言标签缺失或写错、偶发夹带 HTML、列表符号混用。

**平台方言不一致**：公众号只接受 HTML 且会剥掉部分标签和表格；掘金、VitePress 基本是 GFM；脚注只有部分平台支持；图片相对路径发出去就是裂图。

直觉做法是用正则清洗，但嵌套列表、代码块内部的"假 Markdown"会让正则误伤，修一次崩一次。

## 做法

核心原则只有一条：**只维护一份 canonical Markdown，其余全是派生物**。

1. **规范化**。用 unified/remark 把 Markdown 解析成 mdast，先过一遍 lint + 自动修复：标题层级重排、统一列表符号、补代码块语言标签、加粗密度超阈值告警。这一步与目标平台无关，任何 AI 草稿进来先洗成同一种"干净方言"。
2. **方言矩阵**。每个目标平台"要什么、不要什么"写成配置，而不是散落在脚本里的 if-else：

```yaml
targets:
  wechat: { tables: to-list, footnotes: strip, emphasis: reduce }
  blog:   { tables: keep, footnotes: keep, images: rehost }
  plain:  { tables: to-text, emphasis: strip, links: bare-url }
```

每条规则实现成一个十几行的小 remark transform，按矩阵开关组合。

3. **渲染分发**。目标是 HTML 的（公众号）走 rehype + 带内联样式的模板；保留 Markdown 的目标只做删减型 transform。图片统一先传图床再回写外链。
4. **校验**。变换后再 lint 一遍；另备一份"全要素样例文档"（表格、嵌套代码块、脚注、图片全都有），每次改动都在 CI 里渲染到全部目标并留档截图。

## 踩坑点

- 清洗必须落在 AST 层，代码块内容当黑盒，别碰。
- 公众号粘贴会丢样式，导出的 HTML 必须内联 style，类名没用。
- 图床要考虑 referer 防盗链，否则发布端裂图。
- frontmatter 别忘了派生时剥掉，有些平台会原样输出一串 `---`。
- 管线必须幂等：同一输入跑两遍结果要一致，否则 diff 不可信。
- "总结如下""希望对你有帮助"这类 AI 套话是内容问题，格式层修不了，只能在生成侧的 prompt 里约束。

## 可复用建议

- 方言矩阵放配置，别放代码分支；新平台接入通常只是加几行配置加一两个 transform。
- lint 前后各跑一次，规则尽量复用 remark-lint / markdownlint 的自定义规则生态。
- 全要素样例文档是这套管线最好的回归测试，比单测更直观。
- 派生文件不进版本库；进了也别手改，改了必被覆盖。
- 在 OpenClaw 里可以做成一个 skill：Agent 终稿落盘到固定目录 → 触发管线 → 输出到 previews 目录，人工确认后再发。**最后一步人工预览别省**，格式自动化的兜底永远是眼睛。

## 总结

多平台格式适配是确定性工程问题，既不该人肉搬运，也不该每次让 LLM"重新格式化一遍"——后者不可复现，还会悄悄改动内容。AI 负责内容，管线负责形态，边界分清之后，新增一个平台的边际成本基本只剩写一条 transform 加一次预览。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/fa5bdb85acde6e1e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/7022b8fafe02fdc7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-05/4ebeddeb5968f07c.png)

