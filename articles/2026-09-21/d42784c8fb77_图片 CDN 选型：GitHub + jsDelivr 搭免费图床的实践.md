---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 38334
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

跑 OpenClaw 做自动化常遇到一个朴素问题：Agent 产出的图片——运行截图、MCP 工具生成的图表、抓取的素材——需要一个能外链的稳定地址，才能贴进帖子、README、笔记或回传给前端。市面免费图床要么限速防盗链，要么随时跑路；对象存储要实名、配置和少量费用。对个人和社区规模的使用，GitHub 公开仓库 + jsDelivr 是一个零成本、可自动化、随时可迁移的折中方案。本文记录我把它接进 Agent 工作流的完整过程。

## 问题

选型时的硬性要求：

1. 零成本，能扛小流量（日均几千次加载以内）；
2. Agent 能自己完成「上传 → 拿链接」闭环，不依赖人肉操作；
3. 链接可缓存、可预测、可批量替换域名；
4. 仓库本身就是备份，想换 CDN 时数据不动。

## 做法

1. 建一个公开仓库，例如 `cdn-assets`，按 `yyyy/mm/` 分目录。
2. 自动化上传走 GitHub Contents API，一个只给 Contents 读写权限的 PAT 即可：

```bash
B64=$(base64 -w0 shot.png)
curl -X PUT \
  -H "Authorization: Bearer $GH_PAT" \
  https://api.github.com/repos/USER/cdn-assets/contents/2026/02/shot-$(date +%s).png \
  -d "{\"message\":\"upload\",\"content\":\"$B64\"}"
```

响应里的 `commit.sha` 就是后续拼链接用的引用。

3. 拼 jsDelivr 链接：

```
https://cdn.jsdelivr.net/gh/USER/cdn-assets@<commit_sha>/2026/02/shot-xxx.png
```

4. 在 OpenClaw 侧封装一个 20 行左右的脚本或 MCP tool：输入本地路径，输出 Markdown 图片链接，注册进 Agent 工具列表。之后让 Agent「截图并发帖」就是一条完整链路。

## 踩坑点

- **仓库必须 public**，私有仓库 jsDelivr 不回源。连带风险：公开即公开，截图先脱敏（token、聊天记录）。
- **`@main` 链接有缓存延迟**：分支引用最长缓存 12 小时，覆盖同名文件后大概率拿到旧图。要么每次用新文件名 / commit sha，要么手动 purge。
- **国内直连 `cdn.jsdelivr.net` 时好时坏**。备好 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备用域，写成配置项，不要硬编码。
- **文件名只用小写 ASCII + 连字符**，中文、空格、emoji 在 URL 编码上是持续坑源。
- **控制体积**：单文件别超 20MB，PNG 先转 WebP 压一道；GitHub ToS 不鼓励把仓库当纯文件堆场，总量收敛在几百 MB 内比较稳妥。

## 可复用建议

- 把上传逻辑做成独立 MCP 工具 / skill，而不是塞进某个 Agent 的 prompt，任何会话都能复用。
- 备用域名列表、PAT、仓库地址统一放配置，换 CDN 只改一行。
- 文件名带时间戳，天然幂等，Agent 重试不会互相覆盖。
- 仓库即备份：哪天 jsDelivr 不可用，Cloudflare 前置一层或直接 raw 链接兜底，数据零迁移成本。

## 总结

这套方案的定位很清楚：个人/社区规模的图床，不是生产级对象存储。它换来的是零成本、完全可控、与 Agent 工作流天然兼容——Agent 写文件、调脚本、拿链接，全链路无人工介入。如果你的 Agent 也有发图需求，值得花半小时搭起来。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/34b12efa371fa174.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/182c097541ceba37.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/688d3c76f2178740.png)

