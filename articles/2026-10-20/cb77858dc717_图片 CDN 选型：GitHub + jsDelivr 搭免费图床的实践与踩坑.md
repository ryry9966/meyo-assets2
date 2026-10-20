---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 42097
source: 综合讨论
publishedAt: 2026-10-20
---

## 背景

写文档、博客、社区帖时最常见的小工程问题就是图放哪。对象存储要备案要花钱，公共免费图床随时可能关停，链接失效比图丑更致命。对比一圈后我落在 GitHub 仓库 + jsDelivr 上：GitHub 负责存储和版本管理，jsDelivr 负责全球分发，个人规模零成本。

## 问题

三个硬约束：

1. **稳定**——链接不能三天两头 404；
2. **自动化**——希望 Agent 一句话完成"上传 → 返回可引用 URL"，不想每次开网页手动拖拽；
3. **可迁移**——将来换 CDN，正文里的链接不应该全部重写。

## 做法

1. 建一个公开仓库（比如 `pics`），按年份分目录：`img/2025/`。
2. URL 走 jsDelivr 的 GitHub 源：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<tag>/<path>
```

`@main` 跟随分支但缓存策略保守，重要资源建议打 release tag 或用 commit 短哈希，把图片当不可变资源。

3. 自动化上传：复用 GitHub MCP server 的 `create_or_update_file`（本质是 Contents API 的 base64 PUT），给 Agent 配一个 skill——输入本地图片路径，自动生成 `日期-内容hash.png` 文件名、上传、拼出 jsDelivr URL 写回剪贴板。核心就是一层薄封装，参数化 owner/repo/path 三个变量。
4. 可达性兜底：jsDelivr 官方提供 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 备用域名；更稳的做法是 Cloudflare Workers 反代 + 自己的域名，URL 前缀做成配置项，一处替换全局生效。

## 踩坑点

- **同名覆盖不生效**：缓存按路径走，改图后传同名文件，线上可能十几小时不更新。唯一解法是文件名带内容 hash、永不覆盖；官方 purge 接口存在，但别当常规手段。
- **大陆访问波动**：`cdn.jsdelivr.net` 在大陆时好时坏，博客配图可以接受，生产关键路径别指望它。
- **Contents API 并发 409**：携带旧 sha 写入会冲突，随机文件名基本能规避。
- **敏感信息截图**：公开仓库人人可见，内网 IP、token、终端环境变量都算，push 之后即使强推也删不干净。
- **量级与合规**：个人博客、社区帖的量没问题，但别当对象存储跑高频写入，GitHub 条款也不欢迎仓库被当纯 CDN 使用。

## 可复用建议

- 心智模型：**GitHub = 不可变对象存储，jsDelivr = 缓存层**，所有设计围绕"只增不改"。
- URL 前缀配置化，写文用占位符、发布时替换，迁移成本归零。
- 写个十行的 curl 探测脚本定时查 200，坏链早发现。
- 上传流程沉淀成 MCP 工具后，"配图"从三分钟变成一句话——这是这套方案对 Agent 用户最大的价值。

## 总结

GitHub + jsDelivr 不是免费的商业 CDN，而是一个够用、可自动化、可迁移的个人图床基座。按不可变原则组织文件、把链接生成交给 Agent、给可达性留好退路，它能稳定服务很久。真需要 SLA 的场景，还是老实上对象存储。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/488198c09bf69bcb.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/8432521fd0fb9ff1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-20/19deddb41aa1384b.png)

