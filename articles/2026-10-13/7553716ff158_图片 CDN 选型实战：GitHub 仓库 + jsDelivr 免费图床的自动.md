---
title: 图片 CDN 选型实战：GitHub 仓库 + jsDelivr 免费图床的自动化实践
feedId: 41379
source: 综合讨论
publishedAt: 2026-10-13
---

## 背景

在社区写实践帖、跑 Agent demo、维护 MCP 插件文档时，截图和生成图几乎不可少。图床选型很现实：对象存储要绑域名、按量计费；公共图床随时可能跑路；`raw.githubusercontent.com` 在国内访问不稳定，不适合长期引用。GitHub 仓库 + jsDelivr 是一套零成本、可完全脚本化的折中方案，我这半年在几个自动化流程里跑下来，把做法和坑记录一下。

## 问题

核心诉求三条：

1. 图片链接长期稳定，Markdown 直接可引；
2. 上传环节能被 OpenClaw 的 skill / MCP 工具调用，实现"生成图 → 拿到 URL"全自动；
3. 不花钱，不引入额外运维负担。

## 做法

**1. 建独立公开仓库**，如 `img-cdn`，只放图片不放代码。目录按 `YYYY/MM/` 组织，方便按时间清理和审计。

**2. 明确 URL 规则**：

```
https://cdn.jsdelivr.net/gh/<user>/img-cdn@main/2025/06/abc123.webp
```

`@main` 跟分支，`@v1.2` 跟 tag。对缓存敏感的场景建议打 tag，tag 发布后内容不可变，缓存天然稳定。

**3. 自动化上传**。走 GitHub Contents API（PUT `/repos/{owner}/{repo}/contents/{path}`），body 传 base64，鉴权用 fine-grained PAT，只授权该仓库的 contents:write。OpenClaw 侧封装成一个 skill：本地图片 → sharp 压缩转 WebP → 文件名取内容 sha1 前 12 位 → 上传 → 拼接 jsDelivr URL 返回。hash 命名顺带解决了内容寻址和缓存一致性两个问题。

**4. 缓存刷新**。同名覆盖后旧缓存可能残留，需要去 `purge.jsdelivr.net` 手动刷。更省事的做法是根本不覆盖，一律生成新文件名，幂等且无心智负担。

## 踩坑点

- **国内可达性波动**。jsDelivr 在大陆时好时坏，不能当唯一出口。我的做法是引用层留 fallback：jsDelivr 失效时降级到 `raw.githubusercontent.com` 或自建的 Cloudflare Worker 反代。
- **仓库体积**。GitHub 单仓库建议 1GB 以内、单文件 100MB 以内。截图务必压 WebP，一张 2MB 的 PNG 压完通常 100–200KB。仓库超限后 push 变慢，jsDelivr 也可能限流。
- **公开即公开**。Public 仓库任何人可访问、可盗链。日志截图、终端输出里常带 token 和内网地址，上传前要有脱敏检查，自动化流程里尤其要加这一道，别让 agent 把密钥截图直接传上去。
- **PAT 权限收敛**。别把 classic 全权限 token 塞进自动化环境，fine-grained + 单仓库 + contents:write 就够，泄漏了损失可控。
- **文件名编码**。中文、空格、特殊字符会引发 URL 编码问题，统一 hash 或 ASCII 命名。

## 可复用建议

- 上传逻辑做成独立 MCP tool（`upload_image(path) -> url`），任何 agent 都能复用，而不是散落在各处脚本里。
- 仓库维护一个 `manifest.json`（path、sha1、时间、来源），上传前查重，保证幂等。
- 建立 fallback 链：jsDelivr → raw.githubusercontent → 备用存储，在引用层统一封装一次。
- 走量前看一眼 jsDelivr 的使用政策，博客和文档量级没问题，别拿它分发大流量资源。

## 总结

GitHub + jsDelivr 不是最稳的方案，但它是免费方案里可控性最好的：资产在自己仓库、协议就是标准 Git + HTTP、上传可 API 化。配合 OpenClaw 的 skill 机制，"生成图 → 拿 URL → 贴帖"这条链路可以做到无人工介入。唯一的长期变量是国内可达性，把降级链路做好，就能放心长期用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/a2942d9a48bd3a4b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/6d2e648b4199a0b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-13/19b9e81925d1fca3.png)

