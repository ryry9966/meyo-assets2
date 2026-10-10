---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配
feedId: 41115
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

最近半年我们的内容流程稳定成了这样：Agent 负责写稿，产出统一是一份 Markdown，再由脚本分发到公众号、知乎、Hugo 博客和 RSS。写稿交给模型，格式适配交给人写死的代码——中间这层"Markdown 管线"是今天想聊的东西。

## 问题

直接把 LLM 吐出来的 Markdown 丢给各平台，会遇到三层问题：

1. **模型输出不稳定。** 同样的 prompt，这次给你 HTML 内联标签，下次给你脚注和 mermaid，顺手改个标题层级也不奇怪。
2. **平台方言差异大。** 公众号要求全内联样式、图片必须走自己的图床；知乎会吃掉部分语法；Hugo 要 front matter；mermaid 和脚注很多端根本不渲染。
3. **发布动作不可重复。** 图片重复上传、内容改了三行又全量重发，都很折磨人。

## 做法

核心思路一句话：**把"平台适配"从 prompt 里挪到代码里**。管线分五步：

1. **约定 canonical Markdown。** 一页纸的规范：ATX 标题、全文单 H1、标准表格、带语言标注的围栏代码块；禁止裸 HTML、脚注、mermaid、行内 LaTeX。这份规范是模型和管线之间的契约。
2. **归一化。** 模型输出先过一遍 lint + 清洗：剥离 HTML 块、压平标题层级、超宽表格降级成列表。用 remark 拿到 AST 再操作，别用正则硬抠。
3. **AST 级平台渲染器。** 每个目标端一个 adapter，输入 mdast，输出目标格式：公众号 adapter 从主题 JSON 读样式做内联，Hugo adapter 拼 front matter，知乎 adapter 直接删掉不支持的节点。
4. **资产处理。** 图片先用占位符引用，发布前统一上传、回填 URL，并缓存"内容 hash → 图床地址"映射。mermaid 在管线里预渲染成 SVG，不指望平台支持。
5. **幂等发布。** 发布记录里存正文 hash，没变就跳过，变了走 diff 更新。上线前先跑 dry-run，产出预览文件人工过一眼。

在 OpenClaw 里落地很直接：第 1 步写进技能的系统提示作为输出约束，2–5 步包成一个发布技能或 MCP 工具。Agent 只管产出 canonical 文件，`publish --platform xx` 一条命令走完。

## 踩坑点

- **别指望 prompt 做平台适配。** "帮我转成公众号格式"这类指令确定性很差，我们试过，三周就漂移了。模型只负责写规范内的 Markdown，其余全是代码的事。
- **公众号粘贴 HTML 时 CSS 必须全内联**，且 markdown-it 要开 `breaks: true`，否则单换行被吞、段落黏成一坨。
- **外链图床大概率被防盗链挡掉**，微信尤其明显。老老实实走上传接口，映射表一定要缓存，不然每次重发都重传一遍图。
- **front matter 忘了剥**，知乎会把 YAML 块原文渲染出来，非常丑。
- **宽表格在手机端必炸。** 要么限制列数，要么在规范里直接规定表格不超过 4 列。
- **lint 别配太狠。** markdownlint 全开会逼着模型频繁返工，我们只留了十几条硬规则，其余靠 dry-run 人工兜底。

## 可复用建议

- **维护一份 golden 文件**：把所有允许的语法各写一段，每次改 renderer 跑快照对比。平台改版时能第一时间定位坏在哪。
- **主题做成数据**：样式放 JSON 或变量里，renderer 只读不写，换皮不动代码。
- **规范控制在一页纸**：太长模型记不住，执行率反而下降。

## 总结

这条管线没有任何高深的东西，价值全在"约束收敛"：模型收敛到一份规范，平台差异收敛到几个小 adapter，发布动作收敛到幂等接口。做完之后，新增一个平台大概就是半天写一个 adapter 的事。AI 负责生成，管线负责确定性——分工清楚之后，多平台分发就不再是一件烦人的手工活。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/97220adcb29ecfeb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/9fac3970e0261ca7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/585dbe6ad4d64fd9.png)

