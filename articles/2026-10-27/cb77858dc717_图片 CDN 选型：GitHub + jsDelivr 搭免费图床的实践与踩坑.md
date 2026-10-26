---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 42703
source: 综合讨论
publishedAt: 2026-10-27
---

## 背景

写技术帖、发 Agent 实践记录，绕不开截图和流程图。图片放本地、贴聊天工具的临时链接，过几个月基本全挂；第三方免费图床也经历过跑路和限速。最后我选了 GitHub 仓库 + jsDelivr 的方案：仓库托管源文件，jsDelivr 做全球 CDN 分发，零成本，链路足够稳定，而且整条上传流程可以交给 Agent 自动化。

## 问题

核心诉求三条：

1. 链接可长期引用，Markdown 里贴出去不失效；
2. 国内访问不能太差，社区读者主要在国内；
3. 上传动作要可自动化——做 OpenClaw 插件或 MCP 工具时，最好能把"传图、拿 URL、贴进文档"整条链路交给 Agent。

## 做法

**1. 建仓库**：新建一个公开仓库，比如 `assets`，按 `yyyy/mm/` 目录放图。必须是 public，私有仓库 jsDelivr 不回源。

**2. 引用格式**：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch|tag|commit>/<path>
```

`@` 后面写分支、tag 或 commit hash 都行，这一点排障时会用到。

**3. 自动化上传**，分两层：

- 脚本层：用 `gh` CLI 十几行代码完成上传 + 拼接 URL；
- Agent 层：把脚本封装成 MCP 工具，让 Agent 在整理笔记、生成博文时直接调用——输入本地文件路径，输出 CDN 链接。这是这套方案对我最大的价值。

**4. 缓存刷新**：同名文件覆盖后 CDN 不会立刻更新，手动打一次 purge 接口：

```
https://purge.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

## 踩坑点

- **同名覆盖不生效**：jsDelivr 缓存很激进，覆盖上传后旧图可能被缓存很久。最省心的办法是文件名带内容 hash 或日期，从源头避免覆盖。
- **国内可达性波动**：个别地区、个别时段会慢或不通，属于常态波动，写重要文档时做好换源预案。
- **别当网盘用**：单文件建议控制在几 MB 以内，视频和大压缩包别放，jsDelivr 对这类用法会限制，也容易触发滥用策略。
- **commit hash 更精确**：`@main` 拿到的是分支最新的缓存版本，`@<commit>` 精确到某一次提交。教程配图这类"绝不能漂移"的引用，建议用短 commit hash。

## 可复用建议

- 目录规范先行：`yyyy/mm/文件名-hash前8位.png`，几年后也好检索；
- 把上传逻辑做成 MCP 工具后，Agent 可以在"生成博文"流程里自动完成贴图，链路无人干预；
- 心态上把 GitHub 仓库当源、jsDelivr 当分发层，别把 CDN 当存储；
- 写个小工具批量校验旧文链接，失效的统一换成 commit hash 引用再 purge。

## 总结

GitHub + jsDelivr 不是新方案，但对"写文档、发帖、做自动化"的场景，成本、稳定性和可自动化程度都够用。关键是按"仓库为源、CDN 为缓存"的架构来用，别指望它替代对象存储。我的下一步是把 purge 和链接校验也接进 Agent 工作流，让整条图床链路真正无人值守。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/f466c60477afab6c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/0e716d4182b43bcd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/ad48c707a3965193.png)

