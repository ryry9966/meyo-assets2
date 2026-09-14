---
title: Markdown 管线：从 AI 生成到多平台发布的格式适配实践
feedId: 37548
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

Agent 产出的内容天然是 Markdown，但我们的发布目标从来不止一处：博客（GFM）、微信公众号、知乎、掘金，加上内部文档站。同一份文档在五个地方要排五次版，在接入自动化之前，这是纯人力损耗。

## 问题

各平台对 Markdown 的支持是"方言级"差异：

- 公众号编辑器会清洗不认识的 HTML 标签，H1 渲染巨大，锚点不可用；
- 知乎对表格和代码高亮支持有限，直接粘贴会出现原始语法；
- Agent 输出常带 front matter、思考残留、不规范的语言标注（一堆 ```` ```text ````）；
- 图片相对路径出站即失效。

人工适配一次 20-40 分钟且容易漏。我们把它当成编译问题来解：解析一次，按目标平台变换。

## 做法

管线分四段：

1. **归一化**：剥离 front matter，清理空代码块，统一代码围栏语言标注，压缩连续空行。与目标平台无关。
2. **解析成 AST**：用 remark（mdast）解析，后续所有变换都在 AST 上做，不碰原始字符串。
3. **平台适配器**：每个目标平台一个 transformer，按 AST visitor 遍历改造节点。
4. **序列化与校验**：remark-stringify 回写 Markdown，或转 HTML 后走平台 API；发布前跑 lint（图片外链、链接可达、语言标注完整性）。

核心骨架：

```js
import { unified } from 'unified'
import remarkParse from 'remark-parse'
import remarkStringify from 'remark-stringify'
import { visit } from 'unist-util-visit'

const tree = unified().use(remarkParse).parse(raw)

// 某平台适配器：标题层级偏移
visit(tree, 'heading', node => {
  node.depth = Math.min(node.depth + 1, 6)
})

const out = unified().use(remarkStringify).stringify(tree)
```

适配器配置驱动，类似一份 profile：

```yaml
wechat:
  heading_offset: 1
  strip: [footnote, html_block]
  image: upload_and_rewrite
  table_max_cols: 4   # 超出转列表
```

图片处理独立成段：扫描 image 节点 → 上传图床 → 重写 URL → 记录映射表。映射表落盘，重跑时直接命中，不重复上传。

## 踩坑点

- **不要在字符串层面做正则替换**。嵌套列表、代码块里的星号、表格中的竖线都会误伤，AST 是唯一安全的操作层。
- **注意幂等性**。管线重跑时如果重复执行层级偏移，H1 会一路加到 H6。要么在 front matter 里记录已处理平台，要么保证 transformer 纯函数化，始终从原始版本重新生成。
- **转义问题**：中文正文出现 `*`、`_` 时，stringify 的默认转义可能产生 `\*`，公众号反而把反斜杠显示出来，需要在 profile 里关掉部分 escape。
- **降级而非失败**：不支持的语法（脚注、任务列表、公式）转成兼容形式（如脚注改为文末段落），不要让发布中断。
- **Agent 输出不可信**：语言标注缺失、伪代码块、把 YAML 写进正文，归一化阶段要做足兜底。

## 可复用建议

- **单一事实源**：只维护一份 canonical Markdown，所有平台产物由管线生成，禁止手动改产物。
- **transformer 纯函数化**，配 golden file 快照测试：固定输入，比对每个平台的输出文件。
- **profile 与代码分离**，新增平台只需加一份配置加少量 visitor。
- **保留发布记录**（文档版本 × 平台 × 时间），避免重复推送，也方便回滚。

## 总结

这件事的本质不是"格式转换"，而是把渠道差异收敛到配置层。解析一次、变换多次、校验前置，之后再接定时自动发布就是水到渠成。全套基于 remark 生态，半天可以搭出可用版本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/6462c2dec1cdfec0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/22349617da79571e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/58c79a820feb7bbe.png)

