---
title: 从 AI 生成到多平台发布：一条可维护的 Markdown 适配管线
feedId: 40840
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

在 Agent 自动化写作的场景里，LLM 输出的 Markdown 只是起点。同一篇文章往往要发到静态博客、公众号、掘金、知乎等多个平台，而各平台对 Markdown 和 HTML 的支持差异极大：公众号会剥离大部分 HTML 属性，知乎不支持任务列表，静态站需要 frontmatter，外链图片常被防盗链。与其每发一次手工调一遍格式，不如把「格式适配」做成管线里的固定环节。

## 问题

三个典型痛点：

- **生成不稳定**：模型有时用代码围栏包裹输出，frontmatter 时有时无，标题层级随机。
- **平台方言差异**：同一份 Markdown 在不同平台的渲染结果几乎不可复用。
- **脚本腐化**：逐平台手写字符串替换脚本，改一处坏三处。

## 做法：四段管线

1. **生成层**：prompt 里锁定输出方言为 GFM，约定 frontmatter 固定字段（title、tags、platforms）；代码里用 remark 解析剥离围栏，不写正则。
2. **规范层**：以 canonical Markdown 作为唯一事实源，用 remark-lint 加自定义规则做 normalize——唯一 H1、代码块必须标语言、图片先用本地占位符。
3. **适配层**：每个平台一个 adapter，统一签名 `mdast -> platform payload`，基于 AST 遍历做 transform：

```js
const ast = remark().parse(canonicalMd);
const payload = await wechatAdapter(ast); // 表格降级、代码高亮全内联
```

例如公众号 adapter 会把表格节点降级为列表或图片，把 shiki 输出的 class 全部转成内联 style。

4. **发布层**：adapter 产物交给平台 API 或浏览器自动化，统一支持 dry-run。对 Agent 来说，每个 adapter 就是一个 MCP tool，模型只负责调度。

## 踩坑点

- 正则剥围栏，遇到嵌套代码块必炸，交给解析器处理。
- 公众号对 style 属性选择性保留：`position`、`filter` 会被过滤，高亮必须构建时内联，别指望运行时 class。
- 图片必须转存到平台图床或自己的对象存储并做失败重试，直链 raw 图大概率裂。
- 忘了在发布 payload 里剥离 frontmatter，正文顶部出现三条横线——发布前加一道校验。
- CRLF、全角空格、U+00A0 混入会让 lint 反复失败，normalize 第一步先做字符级清洗。

## 可复用建议

- 内容与渲染分离：canonical md 唯一，adapter 全是纯函数。
- 所有 transform 走 mdast/unist，写成 remark plugin，拒绝字符串替换。
- 每个 adapter 配 snapshot 测试，prompt 或平台规则一变，diff 一目了然。
- frontmatter 用 zod 做 schema 校验，缺字段早失败。
- 保留 dry-run：输出本地 HTML 预览，人看过再发。

## 总结

这条管线的核心不是「转格式」，而是把不确定性收敛在生成层，把平台差异隔离在适配层。做到这两点，接入新平台只是多写一个纯函数和一组快照。

---

## 配图

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/ddb395c0110f7340.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/c737ce848ad4a067.png)

