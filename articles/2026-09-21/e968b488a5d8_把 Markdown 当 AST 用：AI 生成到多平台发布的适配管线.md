---
title: 把 Markdown 当 AST 用：AI 生成到多平台发布的适配管线
feedId: 38289
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

现在 Agent 产出的内容基本都是 Markdown：OpenClaw skill 生成的长文、会话整理出的教程、周报、发布稿。但真正要发出去时问题才暴露——微信公众号不认标准 Markdown，知乎、掘金、Notion 各自支持的语法子集不同，图片还牵扯外链和防盗链。多数人的流程是“生成 → 复制粘贴 → 逐个平台手修”，一次性劳动，无法沉淀成能力。

## 问题

格式适配本质上是三个错位：

1. **语法子集错位**：脚注、任务列表、公式、内嵌 HTML，各平台支持度参差；
2. **渲染目标错位**：有的吃 Markdown，有的只吃带内联样式的 HTML——公众号编辑器会把 `class` 和非内联样式全部剥掉；
3. **资源错位**：本地路径、外链图床、防盗链限制，每个平台表现都不一样。

用正则做替换是常见的第一反应，但嵌套列表、代码块里的“伪 Markdown”、表格这些场景，正则几乎必挂。

## 做法

核心思路一句话：**把 Markdown 当 AST 处理，而不是字符串**。基于 unified/remark 工具链，管线分四段：

1. **归一化**：不管来源是哪个模型或 skill，先收敛成一份严格的 CommonMark/GFM 基线。统一松散列表、清理奇异强调符号、裸 HTML 收敛到白名单。
2. **AST 变换**：为每个平台写独立的 mdast transformer。比如公众号目标下，把 heading 映射为带内联样式的 `p`/`blockquote`，把 footnote 节点内联为上标文本。
3. **资源管线**：遍历 AST 提取所有 image 节点，按内容 hash 上传图床后回写 URL。hash 做键保证幂等，重复发布不会重复上传。
4. **序列化输出**：按平台选 renderer——md→md 用 remark-stringify，md→HTML 走 rehype 再注入内联样式，平台私有格式写专门 serializer。

在 OpenClaw 里落地，可以把这套管线包成一个 CLI 或 MCP tool，入参就是 frontmatter 里的 `platforms` 数组，Agent 一次调用完成“归一化 + 变换 + 返回预览链接”。

## 踩坑点

- 公众号 HTML 白名单极窄，只有内联 style 能活下来，主题必须编译成 inline style，别指望 class；
- 图片重写必须幂等且可缓存，否则反复发布会把图床额度刷爆；
- 软换行语义不统一：CommonMark 单换行不折行，GFM 折行，部分国内平台一律按“折”渲染，长段落会被切碎——归一化阶段就要统一换行策略；
- 代码围栏的语言别名（js/javascript、py/python）影响部分平台的高亮，建一张 alias 映射表；
- 脚注、参考文献这类低频语法最容易在转换中**静默丢失**。管线里加一道断言：对比变换前后的 AST 节点类型统计，缺了就 fail，而不是默默输出残稿。

## 可复用建议

- 源文件永远是干净的标准 Markdown，所有平台差异放在渲染层，绝不保存“公众号特供版”；
- frontmatter 当 manifest：`title`、`tags`、`cover`、`platforms` 都写在里面，脚本只认 manifest，不认文件名约定；
- 所有变换支持 `--dry-run`，先产出 diff 预览再落库发布；
- AST 就是最可靠的校验器：发布前跑一遍“目标平台语法白名单检查”，比肉眼审格式靠谱得多。

## 总结

多平台发布不是排版问题，是编译问题：归一化是词法，AST 变换是中间表示，各平台 renderer 是后端。把适配逻辑做成可测试的纯函数管线，Agent 才能真正端到端交付内容，而不是生成完再靠人肉搬运最后一公里。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/54b0837aa4afa6c0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/c3b555a6e96522f2.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/46194af9bfd635b3.png)

