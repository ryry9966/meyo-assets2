---
title: GitHub + jsDelivr 免费图床实践：给自动化流水线里的图片一个稳定外链
feedId: 40942
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

在 OpenClaw 的自动化实践里，一个常见需求是：Agent 跑完任务后产出图片——截图、图表、生成的插画——需要立刻拿到一个可公开访问的 URL，写进 Markdown 或回传给下游工具。对象存储（OSS/COS/S3）当然正规，但要注册云账号、开桶、管密钥，对个人自动化来说偏重；第三方免费图床的寿命又难以预期。GitHub 仓库 + jsDelivr CDN 是一个零成本、纯 API 驱动的折中方案，我在几条流水线里跑了半年，记录一下。

## 问题

我的选型标准，按优先级：

1. **可脚本化**：上传必须是 HTTP API 一发即达，能被脚本 / MCP tool 直接调用；
2. **直链可用**：返回的 URL 能直接嵌 Markdown，访问者无需登录；
3. **零成本**：不付费、不申请 Token 之外的任何资质；
4. **国内可访问**：至少大部分时间可用。

没有方案全占。GitHub + jsDelivr 在 1、2、3 上接近满分，第 4 条是主要风险点，后面细说。

## 做法

**1. 建一个公开仓库**，比如 `pics`，纯存图，无需任何配置。

**2. 准备细粒度 Token**：GitHub Settings → Fine-grained tokens，只授权 `pics` 这一个仓库的 Contents 读写，别用全权限 classic token。

**3. 通过 Contents API 上传**，核心就是一次 PUT，内容 base64 编码：

```bash
B64=$(base64 -w0 ./demo.png)
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  https://api.github.com/repos/USER/pics/contents/202506/demo.png \
  -d "{\"message\":\"upload\",\"content\":\"$B64\"}"
```

**4. 拼 jsDelivr 地址**：

```
https://cdn.jsdelivr.net/gh/USER/pics@main/202506/demo.png
```

**5. 封装成 MCP tool**，这是关键一步：把上面的逻辑包成 `upload_image(path) -> url` 挂进 OpenClaw，之后任何 skill / agent 需要图床时直接调用，上传和返回链接全自动，人不用碰 git。

## 踩坑点

- **缓存不刷新**：同名覆盖后，jsDelivr 的旧缓存可能存活很久。解法有二：文件名带 hash（`demo-a3f2.png`），或主动请求 `purge.jsdelivr.net/gh/...` 清缓存。我的习惯是永不覆盖、只新增。
- **国内访问有波动**：jsDelivr 节点策略近年调整过几次，偶发抽风。个人博客、笔记场景可接受；对外承诺可用性的页面别赌它。
- **别当网盘用**：jsDelivr 的定位是开源项目静态资源分发，塞视频、大压缩包有被限制的风险。图片压成 webp 再传，单文件几百 KB 以内，既快又安全。
- **Contents API 体积限制**：数 MB 的文件走 API 可能报 422，压缩后基本不会触发；实在不行本地 `git push` 兜底。
- **git LFS 文件 jsDelivr 不服务**，仓库别开 LFS。
- **公开仓库人人可见**，敏感截图千万别传。

## 可复用建议

- **命名规范一次定好**：`YYYYMM/<内容hash>.webp`，天然去重、按月归档、永不冲突。
- **URL 生成收敛到一个函数**：不要在 skill 里散落硬编码的 CDN 前缀。哪天迁去对象存储，改一行就能切。
- **上传前自动压缩**：转 webp、限宽 1600px，直接做进 MCP tool，比事后清理省心。
- **保险策略**：GitHub 仓库本身是 source of truth，jsDelivr 只是加速层。它挂了，随时可切 raw 地址或迁移。

## 总结

GitHub + jsDelivr 图床的本质是“用公开仓库换 CDN，用 Token 换自动化”。它不完美——国内可用性有波动、不适合重型资源——但对 OpenClaw 这类个人自动化流水线来说，零成本、纯 API、十分钟搭完，性价比很难拒绝。封装成一个 MCP 工具之后，Agent 侧完全无感，这就是它最大的价值。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/06b3ff9be3a5e1b9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/a682f02a090b00e6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/eb5947441830eb1e.png)

