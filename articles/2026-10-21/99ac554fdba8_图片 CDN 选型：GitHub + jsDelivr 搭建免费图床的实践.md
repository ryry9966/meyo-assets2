---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 42191
source: 综合讨论
publishedAt: 2026-10-21
---

## 背景

OpenClaw 的自动化流程里经常要处理图片：Agent 生成的插图、截图、流程图，最终要落到博客或文档里，都需要一个稳定的外链。我最早直接用 `raw.githubusercontent.com`，实测国内访问不稳定、无缓存，首屏图片经常超时。这篇记录我把 GitHub 仓库 + jsDelivr 作为图床的完整做法，以及封装成 MCP 工具过程中的经验。

## 问题

- raw 链接不走 CDN，大图加载慢，也没有可预期的缓存策略；
- 自动化上传需要一个可编程入口（API），而不是手动拖网页；
- 链接必须可预测、可复现，Agent 生成 Markdown 时才能直接内联。

## 做法

1. 建一个公开仓库（比如 `assets`），与内容仓库分离，避免二进制文件污染主仓库的 git 历史。
2. 创建 fine-grained PAT，只授权这一个仓库的 Contents 读写，权限最小化。
3. 上传走 GitHub Contents API，核心就是一条请求：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_PAT" \
  https://api.github.com/repos/OWNER/assets/contents/2025/06/abc12345.webp \
  -d '{"message":"upload","content":"'"$(base64 -w0 img.webp)"'"}'
```

4. 响应里的 `content.download_url` 是 raw 地址，据此拼出 jsDelivr 链接：`https://cdn.jsdelivr.net/gh/OWNER/assets@main/2025/06/abc12345.webp`。日常用分支名即可，要强一致时用 commit SHA。
5. 覆盖同名文件后，jsDelivr 对分支引用的缓存约 12 小时，可以向 `purge.jsdelivr.net` 发对应路径手动清缓存；更省心的做法是内容 hash 命名，永不覆盖。
6. 在 OpenClaw 侧把第 3 步注册成一个 MCP tool：输入本地图片路径和压缩参数，输出两条 URL。Agent 写文时直接引用返回值，不再手工拼链接。

## 踩坑点

- **别再把它当"国内加速器"**。jsDelivr 自 2022 年后没有国内 ICP 节点，大陆流量走公共 CDN，表现看运营商和时段。上线前务必用目标用户的网络实测，我自己的测试机和读者的体感就不一致过一次。
- **配额不是无限的**：GitHub API 无 token 60 次/小时，带 PAT 5000 次/小时；jsDelivr 对单仓库有合理使用限制。个人博客量级没问题，别当商业 CDN 用，也别拿它扛大流量，仓库有被标记的风险。
- **体积上限**：单文件超过 20MB jsDelivr 不回源，GitHub API 超 100MB 直接拒收。实践里压到 1MB 以内（WebP/AVIF），配额和加载速度都好看。
- **公开仓库任何人可遍历**，敏感截图、带内网信息的截图别往里放。
- **并发冲突**：自动化同时提交同名路径会撞 409/422（SHA 不匹配），脚本里要处理重试或直接靠 hash 命名避开。
- **路径里不要有中文和空格**，URL 编码后回源行为偶发出问题，自动化命名一律 ASCII。

## 可复用建议

- MCP 工具同时返回 jsDelivr URL 和 raw URL 双链接：Markdown 里写 jsDelivr，raw 做兜底，一个挂了另一个还能用。
- 命名用 `{日期}/{内容hash8}-{原名}.webp`，天然幂等，也避开了并发和编码两个坑。
- 上传前加一步压缩（sharp 或 pillow 都行），仓库体积、CDN 命中率、Agent 上下文里贴的链接长度都会更干净。

## 总结

这套方案的本质是"GitHub 当存储、jsDelivr 当分发层"：零成本、免运维、缓存友好，适合个人博客、文档站和 Agent 产出的中低流量图片分发。但它不是对象存储的替代品——没有私有读、配额有限、国内表现看运气。如果图片量级上去了，或对可用性有硬性要求，该上 OSS/R2 就上。选型原则很简单：先明确流量和受众，再决定要不要为"免费"接受这些不确定性。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/5ab836d25866b0bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/a70768972c9a1fec.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-21/7e8d57b835d7b612.png)

