---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的搭建与自动化实践
feedId: 39134
source: 综合讨论
publishedAt: 2026-09-26
---

## 背景

写博客、维护文档、跑 Agent 流水线，截图和生成的架构图越来越多，图床成了绕不开的小基建。商业 OSS 稳定，但要绑域名、备案、算流量费；各类免费图床又随时可能关停。对 OpenClaw 用户来说，真实需求其实很具体：**Agent 产出图片后自动上传，拿到一个可外链的稳定 URL，直接写进 Markdown。**

## 问题

我的约束条件：

- 零成本或近乎零成本；
- 支持 API 上传，能接进 Agent / MCP 工具链；
- 国内访问尽量不抽风；
- URL 稳定，缓存行为可控。

对比下来，GitHub 仓库 + jsDelivr 是工程量最小的组合：仓库当存储，jsDelivr 当现成的全球 CDN，两边都有 API。

## 做法

1. **建公开仓库**（必须 public，私有仓库 jsDelivr 不回源），按 `年/月` 建目录，比如 `img-bed/2025/06/`。
2. **上传走 Contents API**，一段 curl 就够：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/img-bed/contents/2025/06/pic.png \
  -d '{"message":"upload","content":"'"$(base64 -w0 pic.png)"'"}'
```

3. **引用走 jsDelivr 域名**：

```
https://cdn.jsdelivr.net/gh/<user>/img-bed@main/2025/06/pic.png
```

4. **接入 Agent**：把上传动作封装成 MCP 工具或脚本 skill，输入本地图片路径、输出 CDN URL，写博客流水线直接消费。图片入库前先压一遍（转 webp 或 pngquant），体积能省一半。

## 踩坑点

- **国内可用性**：`cdn.jsdelivr.net` 主域名在大陆时好时坏。备好镜像 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net`、`gcore.jsdelivr.net`，发布前 curl 测一圈再定默认域名。
- **缓存不刷新**：`@main` 这类分支引用会被缓存，仓库更新后 CDN 可能还吐旧图。要么 URL 用 commit SHA / release tag 保证不可变，要么手动访问 `purge.jsdelivr.net` 对应路径强制刷新。
- **体积与合规**：jsDelivr 单文件约 20MB 上限，GitHub 也不希望仓库当网盘用。单仓库控制在 1GB 内、别传视频，既是稳定性问题，也是账号安全问题。
- **文件名**：中文、空格、特殊符号在 URL 里全是坑。统一小写 + 连字符 + 日期，如 `20250612-agent-flow.png`。
- **API 限流**：认证后 Contents API 是 5000 次/小时，批量迁移老图片记得分批。

## 可复用建议

- **按批次打 tag**：每攒一批图发一个 `v202506` release，URL 全部不可变，缓存问题直接消失。
- **域名兜底**：模板里 `<img onerror>` 切换备用镜像域名，别把可用性押在单一域名上。
- **幂等脚本**：「压缩 → 上传 → 返回 URL」做成可重试的幂等流程，同名文件先查再传，Agent 重试不会产生脏数据。

## 总结

GitHub + jsDelivr 替代不了生产级对象存储，但对个人博客、文档站、Agent 产出内容的图床需求，它是成本和工程量的最优解之一。真正的价值在自动化闭环：Agent 画完图，脚本压完传完，URL 已经躺在 Markdown 里了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/a8b816ea60943214.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/ea3e4115d79a6001.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-26/8c448e8b3ee08e2f.png)

