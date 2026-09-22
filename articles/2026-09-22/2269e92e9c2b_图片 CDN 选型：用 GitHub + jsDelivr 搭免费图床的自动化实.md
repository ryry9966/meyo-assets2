---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的自动化实践
feedId: 38485
source: 综合讨论
publishedAt: 2026-09-22
---

## 背景

在 OpenClaw 社区写实践帖、调试 Agent、演示 MCP 工具，截图和生成图片是刚需。放到传统免费图床，要么外链过期，要么加防盗链，要么直接跑路。评估下来，GitHub 公开仓库做存储、jsDelivr 做 CDN，是成本最低、且能完全脚本化的组合。

## 问题

我的需求有三条：外链可热链；可用性可控、出问题能排查；Agent 能一步完成「上传图片、返回 URL」。另外 `raw.githubusercontent.com` 有 UA 和防盗链限制，直连 GitHub 国内也时好时坏，所以原始链接不能直接对外。

## 做法

1. 建一个公开仓库（如 `assets`），专门放图。
2. CDN 地址格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch 或 commit>/path/img.webp`。
3. 写上传脚本：调用 GitHub Contents API（`PUT /repos/{owner}/{repo}/contents/{path}`），文件体 Base64 编码，使用 fine-grained PAT，只授予该仓库的 Contents 读写权限。
4. 更新已有文件时先 GET 拿 blob 的 sha，PUT 时带上，否则报 409。
5. 把脚本包成 CLI 或 MCP tool：输入本地文件路径，输出 CDN 链接。注册给 OpenClaw 后，Agent 就能自己传图拿链接。
6. 上传前统一压缩转 webp。

## 踩坑记录

- **缓存不刷新**：`@main` 指向的文件更新后，jsDelivr 可能长时间返回旧内容。要么 URL 固定用 commit hash，要么请求 `purge.jsdelivr.net` 对应路径清缓存。
- **国内可用性波动**：jsDelivr 的国内解析时好时坏，这是选型时就要接受的预设，核心链路必须有自检和备用源。
- **滥用判定风险**：仓库若被判定为纯图床用途，可能被 jsDelivr 拉黑，社区有先例。重要资产别单点依赖。
- **文件命名**：中文、空格会引发 URL 编码问题，统一小写加连字符。
- **API 限流**：匿名 60 次/小时，带 token 5000 次/小时。自动化必须带 PAT，403 时退避重试。
- **隐私**：公开仓库人人可见，含敏感信息的截图先打码再传。

## 可复用建议

- URL 一律带 commit hash，天然不可变，缓存命中率最高。
- 维护 `manifest.json` 记录文件名、sha、CDN 链接，方便脚本幂等和定向清缓存。
- 单图压缩到 500KB 以内，仓库总量控制在 1GB 以下。
- 写一个可用性巡检脚本挂进 OpenClaw 定时任务，失败时告警并切换备用链接。
- 定期把仓库镜像推一份到私有备份。

## 总结

GitHub + jsDelivr 适合社区帖子、文档配图、Agent 输出预览这类「免费、够用、可版本化」的场景，不适合核心业务资产。把它当 best-effort 的免费层来用：链路全脚本化、URL 不可变化、有自检有备份，就能用得踏实。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/670028faa1f6bef2.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/28f5386622bb7677.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-22/563773931123a74c.png)

