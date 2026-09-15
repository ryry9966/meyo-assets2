---
title: GitHub + jsDelivr 搭免费图床：给 Agent 产出的图片找个稳定的家
feedId: 37741
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 的自动化管线经常要产出图片：Agent 生成的图表、插件抓的屏幕截图、博客/RSS 发布用的配图。这些图片需要一个公网可访问、长期稳定、不怕失效的 URL。自建对象存储要维护要花钱，公共图床随时可能跑路。GitHub 仓库 + jsDelivr 是个务实折中：仓库当存储层，jsDelivr 当全球 CDN，零成本、天然带版本管理，而且和 Agent 的代码化工作流很搭。

## 问题

这套方案不是"传上去就能用"。有三个必须想清楚的点：

1. **缓存语义**：jsDelivr 对同一 URL 缓存很激进，同名覆盖后旧图可能长期残留；
2. **硬限制**：单文件体积、仓库规模、API 速率都有边界；
3. **国内可达性**：jsDelivr 主域时好时坏，得有备用路径；
4. **工具化**：怎么把它包装成 Agent/MCP 可以直接调用的能力。

## 做法

**1. 建仓库**
新建公开仓库（如 `imgs`），按 `yyyy/mm/` 目录组织，文件名用短哈希，避免重名和缓存冲突。

**2. 上传两条路**

- 本地：脚本 `git push`，适合批量导入；
- 自动化：走 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`），入参 base64 + 目标路径，出参直接拼接 jsDelivr URL。这是封装成 MCP 工具的推荐姿势。注意覆盖已有文件要带原文件的 SHA，否则返回 409。

**3. 引用地址**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>
```

更稳的做法是打 release tag，引用 `@v1.0`——版本不可变，缓存语义清晰，永远不会被同名覆盖搞脏。

**4. 缓存清理**
同名文件更新后，请求 `https://purge.jsdelivr.net/gh/<user>/<repo>@main/<path>` 强制刷新对应节点。

**5. 备用域**
国内访问不稳时，可切换 `fastly.jsdelivr.net` / `gcore.jsdelivr.net`，或者在 Cloudflare Workers 上做一层反代，把域名前缀收敛成自己的配置项。

## 踩坑点

- **单文件 >20MB，jsDelivr 不回源**，报错还很隐蔽。图片统一压缩、webp 优先，在脚本里做，别靠手。
- **用 `@main` 引用有缓存延迟**（几分钟到更久），传完立刻刷页面会误以为上传失败，其实是 CDN 没更新。
- **仓库是公开的**。别放私人截图、带敏感信息的图；另外删掉文件后，已缓存的 URL 短期内依然有效，"删除"不等于"失效"。
- **别当免费网盘灌**：单文件 100MB 是 GitHub 硬限制，仓库过大拖慢 clone 和 API 上传。
- **Contents API 有速率限制**，批量导入控制并发，用 token 认证，别匿名打。

## 可复用建议

- **封装成小 MCP 工具**："上传 + 拼 URL"一步完成，Agent 产图后直接拿外链，是这套方案里回报最高的投入。
- **维护一份 `index.json`**：记录路径、原始文件名、尺寸、上传时间。Agent 检索历史图片不用遍历仓库，查索引即可。
- **URL 规则定死**：日期目录 + 哈希文件名 + 优先 tag 引用，从源头消灭缓存冲突。
- **仓库即备份**：源数据始终在自己手里，哪天想迁到自建存储，只换域名前缀，URL 结构不用动。

## 总结

GitHub + jsDelivr 图床适合个人博客、文档、Agent 产出的中小规模图片分发：零成本、版本化、URL 稳定、可完全脚本化。它不适合当生产级对象存储——有缓存延迟、体积上限和国内可达性波动。把上传流程工具化、URL 规则固定、对缓存行为有预期，这套方案就能在 OpenClaw 的自动化管线里稳定服役很久。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/fe1201801f86aa29.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/894fb9274cfcca79.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/a7f3926bf8306424.png)

