---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的自动化实践
feedId: 41410
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

在 OpenClaw 的自动化流程里，图片外链是个高频需求：Agent 用 matplotlib 画完图表要贴进消息卡片、插件生成的截图要写进文档、机器人回复里的配图需要一个浏览器能直接访问的 URL。本地路径出不了进程，商业对象存储要实名和付费，公共免费图床限流严重且存活时间不可控。最后我选了个朴素的方案：GitHub 仓库当存储，jsDelivr 当 CDN。

## 问题

直接用 raw.githubusercontent.com 有两个硬伤：国内访问不稳定，且缓存行为不可控，不适合当热链。图床的核心诉求其实就三条：URL 稳定、有 CDN 缓存、上传能被脚本/Agent 自动化——这正好是"git 仓库 + 公共 CDN"能覆盖的组合。

## 做法

1. 建一个公开仓库，比如 `imgbed`，约定只放图片，按 `yyyyMM/` 分目录。
2. 生成 fine-grained Personal Access Token：只授权这一个仓库，权限仅 Contents 的 Read and write，并设置有效期。
3. 上传统一走 GitHub Contents API：`PUT /repos/{owner}/{repo}/contents/{path}`，body 里放 base64 的图片内容，首次上传不用带 sha。
4. 外链格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>`。
5. 在 OpenClaw 里封装成一个 skill（或 MCP 工具）：输入本地文件路径，输出 CDN URL。命名用 `时间戳 + 内容 hash 前八位`，保证文件名不可变。

这样 Agent 的绘图工具跑完后直接调上传工具，把返回的 URL 塞进回复，整条链路无人值守。

## 踩坑点

- **缓存不刷新**：jsDelivr 对同一文件名的缓存很顽固，同名覆盖后旧图可能长时间不更新。解法是内容 hash 命名，从根上避免覆盖；确需刷新时可走 purge.jsdelivr.net 手动清理。
- **大小限制**：API 单文件上限 100MB，但 jsDelivr 对超过 20MB 的文件不提供服务。图片先压缩，5MB 以内基本够用。
- **仓库膨胀**：GitHub 对仓库体积有软限制，图床仓库要定期归档旧图或拆分，别让它变成垃圾场。
- **token 安全**：token 只放环境变量，绝不进仓库和日志；最小权限加定期轮换。
- **隐私**：公开仓库谁都能翻，截图上传前先打码，私密内容别走这条链路。
- **可用性波动**：cdn.jsdelivr.net 国内偶发解析异常，可备 fastly.jsdelivr.net、gcore.jsdelivr.net 作为替代域名，关键场景留 raw 兜底。

## 可复用建议

- 把"上传图床"做成独立工具而非绑死某个流程，Agent、插件、CI 都能复用同一个入口。
- 命名规范统一为 `{yyyyMM}/{hash8}_{语义名}.png`：不可变文件名 + 按月归档，缓存和清理两头都省心。
- 上传前加校验：体积上限、格式白名单，失败要有本地路径兜底，别让整个流程因为图床抖动而中断。
- 文档里的图建议同时保留仓库相对路径，克隆仓库后可离线阅读。

## 总结

这套方案的本质，是把"图床"降级成"一个挂了公共 CDN 的 git 仓库"：零成本、可脚本化、和现有开发习惯零冲突，很适合个人项目与社区量级的自动化场景。它的边界同样清楚——不适合隐私内容，也不适合生产级高并发热链。想清楚量级和内容边界，GitHub + jsDelivr 就是自动化工作流里性价比很高的一块积木。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/93bfc049f2fd609a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/39c5422c89cd3569.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/82b95e845fb231b0.png)

