---
title: 从 Agent 草稿到多平台发布：一条 Markdown 格式适配管线
feedId: 42162
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

用 Agent 写技术稿已经很成熟：让模型跑一圈资料检索，产出一份 Markdown 草稿。问题出在“最后一公里”——同一份草稿要发公众号、知乎、掘金和自己的静态站，四处的 Markdown 方言完全不同。我在 OpenClaw 里把生成和发布拆开：Agent 只负责产出“规范稿”，格式适配交给一条确定性的管线，而不是指望模型针对每个平台各写一份。

## 问题

直接把模型输出丢给平台 API，会遇到三类故障：

1. **语法超集**：AI 爱用脚注 `[^1]`、任务列表、原生 HTML（`<br>`、`<details>`），公众号和知乎的编辑器会静默吞掉或转成纯文本；
2. **结构冗余**：一级标题与正文重复、emoji 当小标题、frontmatter 没剥干净；
3. **增量漂移**：每次让模型“改写成知乎版”，长文的事实细节会悄悄变化，等于多了一次不可控的改写。

## 做法

核心原则：**一份 canonical Markdown，多个 adapter**。管线四步：

1. **源头约束**。在生成用的 system prompt 里给语法白名单：只允许二级以下标题、普通列表、表格、带语言标注的代码块，禁止脚注和行内 HTML。在源头砍掉八成的适配成本。
2. **规范化**。过一遍 remark-lint 自定义规则集，统一中英文间距、标点全半角，剥掉 frontmatter，把 H1 提取为元数据。这一步与平台无关，输出仍是标准 Markdown。
3. **AST 级适配**。用 unified 生态：`remark-parse` 解析成 mdast，每个平台一个纯函数 adapter 在 AST 上做变换，再序列化。比如知乎 adapter 把脚注降级为行内括号：

```js
// adapters/zhihu.js
visit(tree, 'footnoteReference', (node, i, parent) => {
  parent.children[i] = text(`（${defs[node.identifier]}）`)
})
```

公众号 adapter 则把表格转成带内联样式的 HTML 片段。

4. **发布前校验**。每个 adapter 配一份 golden file 快照测试；发布前做 dry-run 渲染，diff 出现异常（比如裸的 `[^`）直接报错，不做静默降级。

在 OpenClaw 里我把第 2–4 步封成一个 skill，Agent 产出规范稿后调用，adapter 按参数选择目标平台。

## 踩坑点

- **代码块语言标注是重灾区**：有的平台忽略它，有的遇到陌生语言标签连围栏都不渲染。白名单只留常用的十来种。
- **表格别信对齐行**，移动端窄屏几乎必炸，重要对比改用列表。
- 数学公式 `$$..$$` 与 `\(...\)` 各平台支持差异极大，非必要不用。
- emoji 标题会破坏部分平台的锚点和目录生成。
- 相对链接离开自己的站点就是死链，适配时统一转绝对链接或删掉。

## 可复用建议

- adapter 必须是纯函数、可快照测试，禁止在里面做网络请求；
- 未知语法宁可报错，不要静默丢弃——丢内容比发布失败更糟；
- 多平台版本永远从 canonical 派生，别让模型各自改写。事实一致性靠管线保证，不靠 prompt 祈祷。

## 总结

格式适配是确定性工程问题，别把它塞回给概率性的模型。Agent 管内容，管线管格式，快照测试管回归——三层各司其职之后，新增一个发布平台，通常只是几十行的一个 adapter 文件。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/6bddf05128050b7f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/1d5b0854ede4b460.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/dad41fada7aca84c.png)

