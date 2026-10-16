---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的自动化实践
feedId: 41746
source: 综合讨论
publishedAt: 2026-10-17
---

## 背景

OpenClaw 的自动化流程里经常要处理图片：Agent 生成图表、MCP 工具返回截图、插件往频道里推附件。这些东西都需要一个能直连、可热链的 URL。可选方案无非三类：公共图床站（随时跑路、防盗链、不受控）；对象存储（稳定但要开账号、按量付费）；自建（要运维，杀鸡用牛刀）。

GitHub 仓库 + jsDelivr 是老组合，但在 Agent 语境下有了新价值：把它封装成一个 MCP 工具，Agent 就能自己"上传图片 → 拿到 URL → 塞进回复"，全程无人工。

## 问题

直接用 `raw.githubusercontent.com` 有两个痛点：国内访问不稳定，且无 CDN 缓存、每次回源。jsDelivr 解决了这两点，但接入自动化后会遇到几个新问题：缓存刷新、命名冲突、鉴权与幂等。

## 做法

1. 建一个公开仓库（如 `cdn-assets`），只放图片，和代码仓库分离。
2. 申请细粒度 PAT，只授权这一个仓库的 Contents 读写。
3. 上传核心就是一次 contents API 调用：

```bash
curl -X PUT "https://api.github.com/repos/USER/cdn-assets/contents/img/2025-06/a1b2c3d4e5f6.png" \
  -H "Authorization: Bearer $GH_TOKEN" \
  -d "{\"message\":\"upload\",\"content\":\"$(base64 -w0 demo.png)\",\"branch\":\"main\"}"
```

4. 拼接 CDN 地址：`https://cdn.jsdelivr.net/gh/USER/cdn-assets@main/img/2025-06/a1b2c3d4e5f6.png`
5. 把 3、4 步包成一个 MCP 工具（输入：本地路径或 bytes；输出：CDN URL），Agent 侧直接调用。

## 踩坑点

- **缓存不刷新**：同名覆盖后 CDN 仍回旧图。最省心的解法是内容哈希命名（sha256 前 12 位）：内容不变则路径不变，天然幂等、缓存友好，重试也安全。确实要覆盖，再走 `purge.jsdelivr.net` 手动刷。
- **contents API 更新需带 sha**：路径已存在时 PUT 会报 422，要先 GET 拿 sha。用哈希命名后路径冲突即内容相同，直接跳过即可。
- **文件名规范**：纯 ASCII、无空格无中文，否则 URL 编码容易翻车。
- **体积限制**：GitHub 单文件 100MB 硬顶，jsDelivr 对大文件也有上限。图片先转 webp（质量 80 足够），省配额又快。
- **可用性要诚实**：jsDelivr 国内解析时好时坏，无 SLA。博客配图、机器人回复没问题；生产关键路径还是上真 OSS。把上传器抽象成接口，随时可换后端。
- **仓库是公开的**：截图里带密钥、内部数据的图一律先脱敏。

## 可复用建议

- 路径约定 `/{场景}/{yyyy-mm}/{hash}.{ext}`，按月归档，将来清理和迁移都方便。
- Token 只放环境变量，工具实现里绝不落盘、不进日志。
- 上传后立刻请求一次 CDN URL 做预热，Agent 贴出去的链接首访不慢。
- 控制节奏，不要在循环里突发批量上传，触发限流得不偿失。

## 总结

这套方案的本质是：存储外包给 GitHub，分发交给 CDN。它免费、够用、可全自动，适合 OpenClaw 玩家的实验性输出与轻量分发；但它是 best-effort，不是基础设施。抽象好上传接口，哪天要升级到对象存储，只改一个函数。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/94a08864f3ca25c9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/641a4f4bf904d79c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-17/722aa57c89387107.png)

