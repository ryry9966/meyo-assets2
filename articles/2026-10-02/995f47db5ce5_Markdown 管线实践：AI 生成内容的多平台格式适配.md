---
title: Markdown 管线实践：AI 生成内容的多平台格式适配
feedId: 40043
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

用 OpenClaw 这类 agent 做内容自动化时，产物大多是 Markdown：技术文章、周报、changelog、发布说明。但"生成完成"离"能发布"还差一截——公众号、掘金、知乎、dev.to 对 Markdown/HTML 的支持子集各不相同。社区里常见的两种做法都不理想：手工复制粘贴再修格式，无法自动化；或者让模型为每个平台各生成一份，结果是内容漂移，几个版本越改越不一致。

## 问题

拆开看是三层：

1. **AI 输出不规范**：标题层级跳跃、代码围栏缺语言标注、混入内联 HTML、表格带合并单元格、图片是相对路径。
2. **平台方言**：公众号几乎剥掉所有 style 属性且不支持脚注；知乎的表格和公式是自家写法；YAML front-matter 在部分平台会原样露出来。
3. **工程问题**：发布不可重放、没有 dry-run、平台规则变更后没有回归手段。

## 做法

核心原则：**一份规范源（canonical Markdown）+ 一组 AST 转换器**，而不是多份生成物。

1. **规范化层**：模型产出先过 lint——标题从 h2 起步（不少平台把 h1 当文章题）、代码围栏必须带语言、禁止内联 HTML、图片先传对象存储再替换为绝对链接。用 markdownlint 规则加一个小 autofix 脚本就够。
2. **AST 转换层**：用 remark/mdast 解析成 AST，每个平台一个 adapter，本质是 AST → 平台方言的纯函数。比如公众号 adapter 把表格降级为分节文本、把脚注展开成文内引用；知乎 adapter 把 `$$` 公式块转成平台接受的写法。
3. **图片中转**：发布前统一走一次上传，在 AST 层面替换节点链接。放在 adapter 之前，所有平台复用。
4. **发布层**：每个 adapter 产出两样东西——目标格式内容 + dry-run 预览，真正发布前先和 golden 文件 diff。
5. **接入 Agent/MCP**：把 normalize / render / publish 包成三个 MCP tool。agent 只负责产出规范 Markdown 和选择目标平台，格式适配完全交给管线。

## 踩坑点

- **别用正则做格式转换**。列表里嵌代码块这种嵌套结构，正则一碰就碎，AST 是底线。
- **下划线转义**：变量名里的 `_` 会被渲染成斜体，规范化层要统一处理。
- **数学公式**：`$$` 块在不同渲染器下行为差异极大。建议规范源统一 KaTeX 语法，降级动作留给 adapter。
- **公众号的 style 属性近乎全剥**，别指望内联样式活下来，主题配色在生成 HTML 之后整体注入才稳。
- **平台规则会变**。golden 快照测试是唯一靠谱的回归手段，平台一改版跑一遍，哪些 adapter 需要更新一目了然。

## 可复用建议

- adapter 写成小模块、注册式接入，新平台的成本控制在百行以内。
- 管线保持幂等：同样输入必须产出同样输出，用内容 hash 判断是否需要重新发布。
- dry-run 设为默认模式，发布是显式动作。
- 规范源文档化，直接写进 agent 的 system prompt，从源头减少 autofix 的量。

## 总结

这条管线的关键不在于转换器写得多聪明，而在于把格式适配从模型职责里剥离出来：模型只管内容，管线只管方言，两边通过规范 Markdown 解耦。接入 MCP 之后，"一次生成、多端一致发布"就退化成一条可测试、可回滚的普通工作流——这才是自动化该有的样子。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/79033fe461db6f66.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/beb19f4bc7413996.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/3eac231a777d42f9.png)

