---
title: 从 AI 生成到多平台发布：一条 Markdown 格式适配管线的落地笔记
feedId: 41137
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

让 Agent 写技术内容不难，难的是发出去。同一篇 Markdown，公众号、知乎、掘金、静态博客的渲染行为各不相同。我们用 OpenClaw 跑了一条「生成 → 适配 → 发布」的管线，迭代半年，把踩过的坑和最终方案整理如下。

## 问题

三个层面的坑：

1. **AI 输出不稳定**。同一条 system prompt，模型有时塞 HTML 标签，有时混 LaTeX，有时列表嵌套深到没法读。prompt 能约束大部分，但兜不住长文和低概率采样。
2. **平台方言**。GFM 只是名义上的通用标准：公众号不支持外链超文本、脚注语法不渲染、表格要在编辑器里手动转；知乎对表格和代码高亮也有自己的脾气。
3. **资源引用**。AI 常写相对路径或占位插图，直接发布会 404；图片在每个平台都要单独上传并替换 URL。

## 做法

核心思路：**canonical Markdown 作为唯一中间表示，平台差异全部收敛到 adapter**。

1. **生成阶段约定语法子集**。在 Agent prompt 里明确允许的集合：GFM 子集、标题 2–4 级、列表最多两层、围栏代码块，禁止裸 HTML。这步只提升下限，不作为正确性保障。
2. **归一化**。用 pandoc 或 remark 转成 AST，所有后续处理基于 AST，不碰字符串正则。
3. **平台适配器**。每个目标平台一个 adapter，输入 canonical AST，输出平台安全的 Markdown 或带 inline style 的 HTML。典型规则：
   - 公众号：外链降级为「文内编号 + 文末 References」；表格转 inline style 的 HTML；代码高亮预渲染成 span。
   - 知乎：表格降级为列表；公式确认目标渲染器方言再输出。
   - 静态站：基本直通，只处理 front matter 和图片路径。
4. **资源处理**。发布前统一上传图片到目标平台或图床，替换 AST 中的 URL，维护 manifest 防止重复上传。
5. **校验与预览**。lint 检查残留 HTML、未展开的脚注、超深嵌套；dry-run 把各平台产物落盘，diff 审阅后再真正发布。

OpenClaw 侧把 publish 封装成 MCP tool：Agent 只负责内容和调用，格式逻辑全在 tool 实现里，Agent 不可绕过。

## 踩坑点

- **正则改 Markdown 必炸代码块**。早期基于正则的链接替换，把代码块里的 Markdown 示例改得面目全非。教训：任何转换先保护 fenced block，最好直接走 AST。
- **不平衡围栏**。文章本身讲 Markdown 时，嵌套三反引号会把解析器搞乱。生成阶段禁止代码块里再写围栏示例，或统一用四反引号包裹。
- **脚注不可移植**。`[^1]` 在多数平台不渲染。归一化阶段就展开成自定义节点，由 adapter 决定呈现方式。
- **相对路径图片**。dry-run 必须校验所有图片 URL 可达，公众号改图要重传，线上 404 很难回滚。
- **平台改版无通知**。给每个 adapter 建 fixture 快照测试，发布前跑样例文档，渲染产物有 diff 就报警。

## 可复用建议

- 内容写一次，格式策略全放 adapter；Agent prompt 里不要出现任何平台名。
- 内部语法规范越窄越好，写进团队 wiki，规范越窄 adapter 越简单。
- 中间产物全部落盘：normalized.md、各平台产物、image manifest，方便复盘和重放。
- dry-run 是默认行为，publish 需要显式参数，别让 Agent 自主上线。

## 总结

多平台发布的复杂度不在「写」，在「适配」。把平台差异收敛成 AST 到 AST 的转换规则，用快照测试兜住平台漂移，用 dry-run 兜住人为失误，管线就能稳定跑。Agent 负责内容质量，管线负责格式确定性，职责分开，两边都能独立演进。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/24d9159abd276096.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/0b7ff19cb1df424a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/633968bd36ae98ca.png)

