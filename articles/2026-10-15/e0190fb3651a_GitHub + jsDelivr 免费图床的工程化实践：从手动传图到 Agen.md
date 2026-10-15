---
title: GitHub + jsDelivr 免费图床的工程化实践：从手动传图到 Agent 自动化
feedId: 41585
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

写技术帖、渲染 Agent 输出、给 MCP 插件生成图文报告时，图片外链是刚需。市面上的免费图床要么防盗链、要么限速、要么悄悄跑路，链接一挂整篇文档就废了。GitHub 仓库免费、版本可控，jsDelivr 在其上提供全球 CDN 缓存，组合起来是零成本且工程上足够可控的方案，尤其适合自动化流水线。

## 问题

要同时解决三件事：

1. **链接稳定性**：`raw.githubusercontent.com` 部分地区访问不稳且无 CDN 缓存，需要一层缓存；
2. **自动化**：Agent 截图或生图后要能自动上传并回填 URL，不能依赖手工；
3. **成本与可持续**：低频博客图床不想付费，也不能接受"图床倒闭链接全灭"。

## 做法

1. 建一个公开仓库，如 `pics`，建议独立于代码仓库，专库专用。
2. 图片按日期分层存放：`assets/2025/0115-xxx.webp`。
3. 引用 URL 模板：

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/assets/2025/0115-xxx.webp
```

需要不可变版本时，把 `@main` 换成 `@<commit-sha>` 或打 tag。

4. 自动化接入：给 Agent 配一个 GitHub MCP 插件（或直接调 Contents API），核心流程三步——读文件 → base64 → PUT 提交，拿到 commit 后拼出 jsDelivr 链接回填 Markdown。伪代码：

```text
content = base64(image)
PUT /repos/me/pics/contents/assets/2025/0115-shot.png
url = "https://cdn.jsdelivr.net/gh/me/pics@main/assets/2025/0115-shot.png"
```

这样"生成图 → 传仓库 → 拿链接 → 写文档"的闭环可以无人值守。

## 踩坑点

1. **单文件上限 20MB**：jsDelivr 对超限文件不提供服务。截图入库前统一压缩成 WebP，基本不会碰到。
2. **`@main` 有缓存延迟**：jsDelivr 对分支引用的解析有缓存（官方口径最长 12 小时），刚提交的图可能 404。自动化里建议提交后立刻用 commit sha 版 URL 做探活，或对分支版 URL 做带退避的重试。
3. **同名覆盖不生效**：改了文件内容，CDN 不会立刻刷新，要去官方 purge 工具手动清。更简单的解法是禁止覆盖写，文件名永远带短 hash。
4. **大陆访问波动**：jsDelivr 在国内的可用性历史上反复波动，不能当 SLA 服务用。可用性敏感的页面加 `onerror` 降级到 raw 地址或备用 CDN。
5. **别当网盘**：大文件、视频、纯存储型仓库会被 jsDelivr 拉黑。只放图床用途的图片，仓库体积控制在 GB 级以下。
6. **token 权限收敛**：自动化用 fine-grained PAT，只授这一个仓库的 Contents 读写，泄露也不波及全账号。

## 可复用建议

- **命名规范**：`{日期}-{语义名}-{短hash}.webp`，天然防覆盖、可追溯。
- **维护 `index.json` 清单**：记录文件名、URL、尺寸、上传时间，Agent 侧做去重和引用统计很方便。
- **封装统一的 `upload_image` 插件**：所有 Agent 共用一个入口，出问题只改一处。
- **入库前统一压缩**（如 sharp 转 WebP），仓库体积和 CDN 流量双省。
- **兜底策略**：主链 jsDelivr，备用 raw 地址或 Statically，降级逻辑写在模板里而不是临时救火。

## 总结

GitHub + jsDelivr 不是最快的图床方案，但是最工程可控的免费方案：仓库即存储，git 即版本管理，API 即自动化接口。配上 MCP 插件把上传收进 Agent 工具链后，整条"生成 → 上传 → 引用"链路完全无人值守。它的边界同样清晰——不适合大文件，不适合强可用性承诺。把预期放对，这套组合能稳定用很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/0c7f2abce94fc9a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/e3d12f46361d2318.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/5368def23affe79b.png)

