---
title: 一条 Markdown 管线：从 Agent 生成到多平台发布的格式适配
feedId: 43214
source: 综合讨论
publishedAt: 2026-11-02
---

用 Agent 产出内容之后，真正耗时的往往不是写作，而是发布：同一篇 Markdown 要投到博客、公众号、掘金、知乎，各家语法方言和渲染限制都不一样。手动逐平台改格式，本质是重复劳动且容易出错。我们最近把这条链路固化成了一个基于 OpenClaw 的发布 skill，这里记录一下设计思路，供有同样需求的同学参考。

## 问题

多平台发布的核心矛盾是"一份源文件，多种渲染约束"：

- **Markdown 方言不一致**：表格、脚注、任务列表、数学公式，各平台支持度参差不齐；
- **HTML/CSS 限制**：公众号只认内联样式，class 属性会被整个剥掉；
- **图片链路脆弱**：本地路径和外链都会失效，外链还有防盗链，必须重新托管；
- **AI 生成内容特有的坑**：幻觉链接、代码块里混入未转义的尖括号、frontmatter 漏到正文里。

## 做法

管线拆成五步，每步独立可测：

1. **规范化源文件**。统一 frontmatter（title/tags/platforms），跑 remark-lint 先消除语法噪声，这一步不做任何平台特定改写。
2. **AST 层转换**。用 unified/remark 生态，把 Markdown 解析成 AST 后按平台 profile 做变换。profile 大致长这样：

```yaml
# profiles/wechat.yaml
code_highlight: inline-css
footnote: tail-paragraph
math: render-image
image_target: wx-material-api
```

   比如公众号 profile 会把表格转成内联样式的 HTML、脚注转成尾注段落、代码块语言标签映射到支持的高亮方案。坚决不用正则处理正文——正则分不清代码块内外。

3. **图片管道**。扫描所有 image 节点，下载后按内容 hash 缓存，再上传到目标图床或平台素材接口，回写 URL。同一张图多平台复用时只上传一次。
4. **渲染校验**。dry-run 渲染成目标 HTML，diff 检查：有没有裸露的 frontmatter、未替换的相对路径、被吞掉的代码块。
5. **发布与回执**。有 API 的走 API（GitHub、掘金），没有的输出到剪贴板或草稿文件由人确认。发布结果回写 frontmatter，记录 URL 和版本。

在 OpenClaw 侧，前四步封装成 MCP 工具，Agent 负责编排和异常处理，人只在最后一步确认。

## 踩坑点

- **AST 变换必须跳过 code 节点**。所有"把 A 替换成 B"的规则都要先判断节点类型，否则正文里的代码示例会被改坏，这是我们在第一版里真实翻过的车；
- **内联样式转换放在 rehype 之后做**，不要试图在 Markdown 层写死样式，两层职责混在一起后面没法维护；
- **图片上传要做幂等**：内容 hash 相同就跳过，否则重跑管线会产生一堆重复素材；
- **链接校验别省**。AI 生成的参考链接先抽验一遍，404 的降级为纯文本，比发出去之后挨骂强；
- **数学公式是重灾区**。我们最终选择渲染成图片，放弃了各平台原生公式兼容的幻想。

## 可复用建议

- 把平台差异全部收敛到 profile 文件里，管线本体保持平台无关；
- 每个平台 profile 配一组 golden file 快照测试，改转换逻辑跑一遍就知道影响面；
- 原始 Markdown 永远不动，所有产物都是派生物，随时可以重建；
- 优先打磨图片和代码块的处理，这两类问题占了实际排障时间的八成。

## 总结

这条管线没有黑科技，就是把格式适配从"每次手工"变成"一次工程化"。Agent 适合做编排和兜底，确定性转换交给 AST 工具链，人保留最终确认权。经验上，当发布平台数量超过三个时，这套投入就开始回本了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/a0491ae28087ca91.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/379f861fbf5f4e7d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-02/d8ae85358af09454.png)

