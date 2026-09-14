---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 37496
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

Agent 产出的内容默认是 Markdown，但发布端从来不是。公众号只接受内联样式的 HTML；知乎会剥掉大部分标签；静态站点吃标准 GFM；Notion 是自己的块模型。常见的错误做法是让模型"直接输出适合各平台的版本"——这等于把确定性的格式工作交给概率模型，每次产出都不同，无法 diff，也无法做回归测试。

## 问题

实际落地时，麻烦集中在三类：

1. **方言差异**：表格、脚注、任务列表、数学公式在各平台的渲染规则互不兼容；
2. **资源处理**：外链图片被防盗链拦截，公众号图片必须走素材上传接口重传；
3. **发布回路**：失败重试容易产生重复文章，且没有可靠的状态可查。

## 做法

**第一步：定义内部规范（canonical spec）。** 以 GFM 子集为唯一源格式：禁止裸 HTML、禁止行内样式、公式统一 `$$` 块。用 markdownlint 加自定义规则在生成端做门禁，不合规直接打回让 agent 重写。不要指望 prompt 能锁死格式，linter 才是硬约束。

**第二步：AST 化，放弃正则。** 用 unified/remark 把 Markdown 解析成 mdast，所有转换都作用在语法树上。正则方案在嵌套代码块和转义字符面前几乎必挂。

**第三步：适配器模式。** 每个目标平台一个 serializer，互相无状态：

```js
// 公众号适配器片段：mdast 节点 → 内联样式 HTML
const handlers = {
  heading: (n, _, next) => `<h2 style="font-size:17px;">${next(n.children)}</h2>`,
  image:   async (n) => `<img src="${await reuploadToWx(n.url)}">`,
};
```

**第四步：资源管线。** 图片先统一落到中转存储，发布时由适配器调用平台上传接口替换 URL，并记录新旧地址映射，便于回滚。

**第五步：状态表。** 维护 `content_id → 平台 post_id + 状态 + 内容哈希` 的映射。哈希未变就跳过重发，幂等问题基本消失。

## 踩坑点

- 公众号对 style 支持有限，flex/grid 会被直接剥掉，排版要降级到 block + margin；
- 标题锚点、脚注跳转链接会被部分平台清洗，目录类交互全部失效；
- AI 产出有系统性格式漂移：开头必放 H1、加粗滥用、列表标记 `-` 和 `*` 混用——这属于统计习惯，只能靠门禁纠正；
- 代码块语言标记部分平台不识别，高亮丢失后要预留 fallback 样式；
- 让 LLM 直接产出平台 HTML，内容改一个字就要全量重跑，且没有任何 diff 能力。

## 可复用建议

- **职责切干净**：LLM 只写内容，格式转换全部交给确定性代码；
- **单一中间表示**：mdast 是唯一枢纽，新增平台只需一个 serializer + 一个上传函数；
- **发布前冒烟测试**：用无头浏览器渲染目标页面，截图比对关键区块是否存活；
- **MCP 工具设计**：对外只暴露 `publish(platform, content_id)`，不要把 HTML 细节暴露给模型，否则它一定会"顺手优化"你的样式。

## 总结

把格式适配从模型手里拿走，交给一条"规范 → AST → 适配器 → 状态表"的管线，AI 生成和多平台发布才算真正解耦。内容层可以频繁迭代，格式层保持稳定可测——这套结构不复杂，但它决定了自动化发布是"能跑"，还是"能维护"。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/90e4601ba033c71d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/e36de9e3d003cc43.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/2339b867ec1b893e.png)

