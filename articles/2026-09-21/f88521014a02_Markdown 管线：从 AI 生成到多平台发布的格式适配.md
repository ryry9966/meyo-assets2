---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 38311
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

在 Agent 工作流里，AI 产出的初稿几乎都是 Markdown。但"生成完成"不等于"可以发布"：GitHub 认 GFM 全集，知乎会剥掉大部分内联 HTML，公众号只接受带内联样式的富文本，掘金又有自己的扩展语法。同一份稿子发三个渠道，手工适配往往要改三轮，而且每轮都可能改出新问题。

## 问题

把适配做成手工活有三个代价：一是不可复现；二是 AI 生成的 Markdown 本身就不规范——标题层级跳跃、列表嵌套错位、混入裸 HTML 和未转义字符，人工修补容易漏；三是新增平台时适配逻辑散落各处，没法测试也没法回归。

## 做法

核心思路：**源 Markdown 只有一份，平台差异全部表达为转换，转换基于 AST 而不是字符串替换。**

1. **规范化入口**。发布前先过 lint（remark-lint 或 markdownlint），强制标题从固定层级开始、统一列表缩进、代码块必须标注语言。AI 输出不规范是常态，别指望 prompt 解决所有问题。
2. **AST 转换层**。用 unified/remark 解析成 mdast，按平台写 transform 插件：公众号渲染器把代码块转成内联样式 HTML；知乎目标剥离 footnote、对表格做降级；不支持任务列表的渠道转普通列表加文字前缀。所有操作落在 AST 层，输出可预测、可 diff。
3. **MCP 工具封装发布**。每个渠道一个 MCP tool，入参统一为规范化后的 Markdown 加平台参数，出参是发布回执链接。Agent 只管调用，不感知平台差异。
4. **落盘中间产物**。每次转换保存 source.md、转换后 HTML 和 manifest.json（记录插件与规则版本），排障时逐层 diff。

## 踩坑点

- 正则替换是最大的坑。嵌套代码块里的分隔线、表格里的竖线都会被误伤，务必走 AST。
- 图片链接：模型常生成相对路径，图床上传必须作为管线里的显式步骤，不能靠发布时手补。
- 公众号编辑器粘贴时会吞掉部分属性，验证要走真实粘贴流程，不能只看渲染出的 HTML。
- 数学公式和 footnote 是重灾区，多数平台不支持，要么转图片要么明确降级，别静默丢内容。
- transform 必须幂等，并配快照测试，平台改版后才有回归依据。

## 可复用建议

- 新平台接入顺序：先写降级策略（明确哪些语法不支持），再写 transform，最后写发布 tool。
- lint 规则收敛为一份共享配置，生成端和发布端用同一套，避免"生成合规、发布报错"。
- 转换器保持纯函数：输入 Markdown、输出 Markdown/HTML，不夹带 IO，方便单测和跨项目复用。

## 总结

这条管线的本质，是把格式适配从"人的经验"变成"可测试的代码"。规范化入口、AST 转换、MCP 封装三段各司其职之后，新增平台的成本就从"重新手改一遍"降到"一个 transform 加一个 tool"。OpenClaw 的插件机制可以直接承载这套结构，各平台 transform 的实现差异很值得在社区里对齐，欢迎贴出来一起看。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/04ce36c836d5e37b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/5e539a856b755943.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/f38316f406bdefa6.png)

