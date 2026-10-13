---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配实践
feedId: 41432
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

最近用 OpenClaw 的 Agent 批量产出技术帖和文档草稿，AI 输出的 Markdown 大体规范，但一旦要分发到博客、公众号、掘金、GitHub README 等多个平台，就会发现"一份 Markdown 走天下"并不成立。各平台渲染器对语法子集、HTML 白名单、元数据格式的要求都不一样，手工逐篇调整不可持续，于是把格式适配做成了 pipeline。

## 问题

实际遇到的偏差集中在几类：

- 标题层级混乱：AI 有时从 `##` 开始，有时一篇文档里出现多个 `#`；
- 代码块缺少语言标注，或语言名不统一（`js` / `javascript`）；
- 数学公式、脚注、任务列表等扩展语法，平台支持度参差；
- 图片用本地相对路径，发布后必然裂图；
- 公众号类平台只吃内联样式的 HTML，外链还会被剥掉。

## 做法

把管线拆成四层，尽量复用 remark 生态：

1. **归一化（normalize）**。用 `remark-parse` 解析成 mdast，做 AST 级改写：强制单一 H1、补齐代码块语言、统一列表缩进、收拢游离 HTML。这里严禁裸正则——代码块里的 `#` 和反引号会让你怀疑人生。归一化后再过一遍 `markdownlint` 自定义规则作为门禁。
2. **元数据（frontmatter）**。从 Agent 产出元信息生成 title / tags / date / slug，YAML 里含冒号的字符串一律加引号，tags 统一为数组。
3. **平台适配器（adapter）**。每个目标平台一个小模块，输入统一 AST，输出各自方言：博客走 rehype 渲染 HTML；公众号走 rehype + 内联 CSS + 外链转文末引用；dev.to 类直接拼 frontmatter。适配器只做转换，不做业务判断。
4. **发布通道**。有 API 的走 API（GitHub、dev.to），没有的导出剪贴板或可粘贴文件，配 `--dry-run` 先 diff 预览。整条管线包成一个 MCP 工具暴露给 Agent，写完稿自己调用校验和导出，闭环就通了。

整条管线做成幂等的：对已处理文档再跑一遍，输出不变。

## 踩坑点

- 在 AST 之前用正则替换，被代码块内容误伤过两次，之后所有转换都约束在 mdast 节点层面；
- 公众号外链剥离不是 warning 而是静默丢弃，链接必须转成文末参考列表，否则读者根本不知道少了东西；
- 行内 `$...$` 在部分渲染器里会被当字面量，中文全角标点紧挨 `$` 更容易触发解析异常，统一改成 `$$` 或图片方案更稳；
- 图片不做一次性的"上传 + 改写 URL"，本地相对路径在 CI 环境必挂；
- Agent 每次生成措辞略有差异，归一化规则必须写成确定性规则，不能依赖"看起来差不多"。

## 可复用建议

- 单一事实源：只留一份 canonical Markdown，所有平台版本都是派生物，随时可重建；
- AST 优先、正则兜底，正则只处理 AST 覆盖不到的纯文本场景；
- 准备一组"刁钻样例"（嵌套表格、代码块含 fence、脚注与公式混排）做 golden test，改适配器先跑样例；
- 适配器保持声明式小函数，新平台半天能接上；
- 发布前强制 dry-run diff，人看一眼再推。

## 总结

格式适配本质上是个编译问题：canonical Markdown 是源语言，各平台方言是目标语言，管线就是带 lint 门禁的编译器。把转换规则从"人的经验"沉淀成确定性代码后，Agent 产出的内容才能真正做到一次生成、多端一致。这套管线在 OpenClaw 自动化流里跑了数周，新增平台只需补一个适配器，维护成本可控。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/145682fe744c62b5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/1c85109a3bdee8aa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/21d2c785f570380c.png)

