---
title: 图片 CDN 选型：用 GitHub 仓库 + jsDelivr 搭免费图床，并接进 Agent 工作流
feedId: 40094
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

写插件文档、跑自动化任务、让 Agent 产出截图和图表，都会撞上同一个需求：图片得有个稳定的外链。对象存储要备案和计费，公共图床时好时坏还限外链。最后我选了比较“土”但耐用的方案：GitHub 公开仓库做存储，jsDelivr 做分发。

## 问题

选型时主要看四点：零成本、可脚本化、URL 可预测、国内可访问。前两点 GitHub 天然满足，第三点靠 jsDelivr 的 gh 源解决，第四点要打问号——这也是后来踩坑的主要来源。

## 做法

1. 建一个公开仓库（比如 `images`），目录按 `年/月/哈希.png` 组织。
2. 上传走 GitHub Contents API，一个 PUT 就能提交文件，不需要本地 git：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/<user>/images/contents/2025/06/a1b2c3.png \
  -d '{"message":"upload","content":"<base64>"}'
```

3. 引用地址换成 jsDelivr 格式：
   `https://cdn.jsdelivr.net/gh/<user>/images@main/2025/06/a1b2c3.png`
4. 覆盖已有文件后，主动请求 `https://purge.jsdelivr.net/gh/<user>/images@main/<path>` 清缓存。
5. 接入 OpenClaw：把"压缩 + 上传 + 拼 URL"封装成一个 MCP 工具，入参是本地图片路径，出参是 CDN URL。之后 Agent 生成的任何图片，都能在对话里一句话完成“发布”，下游工具直接引用。

## 踩坑点

- **国内连通性波动**：jsDelivr 的国内节点这几年时好时坏，别把它当唯一出口。核心资源保留 GitHub raw 或自有域名兜底，写一个简单的 fallback 逻辑。
- **缓存不是即时的**：`@main` 路径有缓存窗口，覆盖同名文件后必须 purge，否则拿到的是旧图。用内容哈希命名、永不覆盖，能省掉大半麻烦。
- **文件大小限制**：单文件超过约 50MB jsDelivr 不再服务，仓库也别养太大，建议控制在 1GB 内，上传前统一压缩并转 WebP。
- **这是公开仓库**：任何拿到 URL 的人都能访问。带 token、带内网地址的截图务必打码或干脆别传。
- **API 限流**：批量搬运历史图片时注意频率，加个简单重试加 sleep 就够，认证后额度日常场景用不完。

## 可复用建议

- 文件名用内容哈希前 12 位 + 扩展名，天然去重，URL 本身就是稳定的缓存键。
- MCP 工具里顺手做三件事：压缩、哈希命名、连同返回 purge URL，下游不用再操心。
- 打个 tag（如 `v2025`）钉住版本做长期引用；`@main` 只留给“永远要最新”的场景，比如博客配图。
- 在仓库 README 里写清目录约定和 LICENSE，半年后的自己会感谢现在的你。

## 总结

这套方案的本质是“用公开仓库换免费分发”。它不完美——国内连通性看缘分、隐私为零——但它零成本、可脚本化、URL 可预测，非常适合文档配图、Agent 产出物外链这类中低强度场景。先把 fallback 和命名约定想清楚，再放心把它交给你的自动化流程。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/5ac389488e4cb758.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/d02ff048c5cb31d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/2bf2171bf6f42978.png)

