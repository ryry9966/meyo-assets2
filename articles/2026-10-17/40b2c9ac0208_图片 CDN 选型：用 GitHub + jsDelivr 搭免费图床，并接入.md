---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床，并接入 Agent 自动化
feedId: 41842
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

在 OpenClaw 的日常使用里，让 Agent 产出图文内容是高频场景：定时生成的周报、自动化发布的学习笔记、插件输出的运行截图。这些内容一旦要被 Markdown 引用，就绕不开图床问题。

对象存储（OSS/COS/S3）稳定但要花钱、要配签名；公共图床服务有配额和防盗链限制；而 GitHub 仓库 + jsDelivr 的组合，零成本、天然版本化、URL 规则简单，对个人开发者和小流量场景是个务实的选择。本文记录一次完整的接入与自动化改造过程。

## 问题

核心诉求有三个：

1. URL 可外链、可缓存，不带鉴权头就能在任意 Markdown / RSS / 网页里引用；
2. 上传过程可被 Agent 调用——最好封装成 MCP 工具，写作时自动完成"传图 → 拿链接"；
3. 成本为零，且随时可迁移（图片本质还是文件，不锁定在某个平台）。

直接用 raw.githubusercontent.com 也能外链，但没有 CDN 缓存，国内访问也经常不稳；jsDelivr 在 GitHub 仓库之上加了一层全球 CDN，正好补齐这块。

## 做法

**第一步**：建一个公开仓库，比如 `assets`，专仓专用，不和代码混放。

**第二步**：上传。手动场景直接 `git push`；自动化场景用 GitHub Contents API：

```text
PUT /repos/{owner}/{repo}/contents/{path}
Body: { "message": "upload", "content": "<base64>" }
```

带一个 fine-grained PAT（只授该仓库 contents:write），一次请求就能传图。

**第三步**：拼 jsDelivr URL：

```text
https://cdn.jsdelivr.net/gh/{owner}/{repo}@main/2025/0615-a3f8c2.webp
```

**第四步（收益最大）**：封装成 MCP 工具。输入本地文件路径，工具内部做压缩（统一转 webp）、base64 编码、调 API、返回最终 URL。之后任何 skill / 插件都能一句话完成上传，Agent 写作流里不再有"人肉传图"这个环节。

## 踩坑点

- **覆盖同名文件 = 缓存灾难**。分支引用（`@main`）的缓存约 12 小时，内容改了 URL 不变，读到的还是旧图。方案：强制 append-only，文件名带日期 + 内容短哈希，永不覆盖。
- **确定性引用用 commit SHA**。`@main` 适合草稿迭代，正式文档建议 `@<sha>`，不可变、缓存寿命长。
- **单文件别超过 20MB**，超过会被 CDN 拒发；上传前压缩，webp 通常能砍掉一半以上体积。
- **更新已有文件时 API 会 409**。Contents API 更新必须带原文件的 sha，先 GET 再 PUT；并发写同一文件会互相踩——append-only 命名顺带解决了这个问题。
- **国内可达性有波动**。jsDelivr 历史上出现过不可用期，别把它当生产级硬依赖，重要内容保留本地副本。
- **公开仓库即公开数据**。截图里的 token、内网 IP、日志先打码再传。

## 可复用建议

1. 把"上传 → 返回 URL"固化成 MCP 工具，所有 Agent 流程直接复用，这是本次改造的核心价值；
2. 仓库里维护一份 `index.json`（文件名、尺寸、上传时间、引用位置），既是审计日志，也方便日后批量迁移；
3. 压缩 + 改名逻辑放在工具里强制执行，不靠自觉；
4. PAT 用 fine-grained、最小权限、只绑这一个仓库，定期轮换；
5. 这是免费公共服务，别拿它分发大文件或当网盘用。

## 总结

GitHub + jsDelivr 不是新方案，但配合 MCP 工具化之后，它从"手动图床"变成了 Agent 内容流水线的一个标准组件。定位要清楚：适合个人博客、Agent 产出的图文、低流量分发；不适合高可用生产环境。守住 append-only 命名、SHA 固定引用、上传前压缩这三条纪律，这套方案可以稳定用很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/cf368a255684cc80.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/bfef9ddaf3fad6d6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/72a2b688bcfb5fbd.png)

