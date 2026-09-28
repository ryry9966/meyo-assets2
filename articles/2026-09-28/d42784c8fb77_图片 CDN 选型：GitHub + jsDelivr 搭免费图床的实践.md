---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 39256
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

写技术帖、给 Agent 生成的报告配图，是社区里的高频需求，图片放哪一直是个小麻烦：GitHub raw 链接部分地区访问不稳定，公共图床要么有防盗链，要么随时跑路。最后我选了 GitHub 仓库 + jsDelivr 的组合：零成本、无审核门槛，关键是全流程可脚本化——这对 OpenClaw 用户尤其重要，因为图床必须能被 Agent 调用，而不只是人肉上传。

## 做法

1. **建一个公开仓库**（如 `pics`），按 `2025/06/` 这类日期目录组织，避免根目录堆爆。
2. **生成 Fine-grained PAT**：只授权这个仓库的 Contents 写权限，最小化泄露风险。
3. **上传走 Contents API**：文件 base64 后 `PUT /repos/{owner}/{repo}/contents/{path}`，一个请求完成提交，不需要本地 clone。
4. **引用统一走 jsDelivr**：
   `https://cdn.jsdelivr.net/gh/{user}/pics@{commit_sha}/2025/06/abc.png`
5. **封装成 MCP 工具**：`upload_image(path) -> url`，输入本地路径或 base64，输出 CDN URL。写帖 Agent、报告生成 Agent 都能直接调用，返回链接直接塞进 Markdown。

## 踩坑点

- **缓存强、更新不生效**：jsDelivr 对同一 URL 缓存非常激进，同名覆盖后大概率还是旧图。对策：文件名加内容 hash（`sha256 前 12 位`），物理不可变，还天然去重。
- **`@main` 是双刃剑**：方便但缓存不可控。发布时用 commit SHA 钉死版本，草稿期再用分支名。
- **单文件控制在 20MB 以内**，仓库别无脑堆，膨胀后 API 和维护都会变慢。
- **可达性有波动**：jsDelivr 在部分地区/时段出现过不稳定，重要内容保留 raw 链接兜底，或备一个反代域名。
- **API 限流**：带 token 是 5000 次/小时，Agent 批量处理时注意别打满；重试逻辑要幂等（先查文件是否已存在）。

## 可复用建议

- 文件名规范 `{yyyy-MM}/{hash12}.{ext}`，配一个 `manifest.json` 记录"原始名 → CDN URL"映射，Agent 二次引用时先查索引。
- MCP 工具签名保持简单，错误信息里带上限流剩余额度，方便 Agent 自行决策是否等待重试。
- PAT 最小权限 + 定期轮换；仓库是公开的，任何带敏感信息的截图先脱敏再传。

## 总结

这套方案本质是"用 Git 做存储，用 jsDelivr 做分发"：不花钱、可控、可自动化，缺点是可达性依赖第三方，缓存行为需要用命名约定绕开。对于博客配图、Agent 生成物这类**低频写、高频读**的场景，性价比很高；但别拿它扛生产业务流量。把上传动作封成 MCP 工具后，它就成了所有 OpenClaw Agent 的公共基础设施——这才是选型的核心收益。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/21aa2d7c49c4498b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/ab157b3c7cbcacc0.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/fea49f52c7c74bbc.png)

