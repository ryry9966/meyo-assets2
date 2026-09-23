---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床的实践与踩坑
feedId: 38601
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

写博客、发社区帖、跑 Agent 自动化产出图文内容时，图床是绕不开的基础设施。市面上的图床要么收费，要么免费但限制多、随时可能跑路。对于已经重度使用 GitHub 的人来说，「GitHub 仓库 + jsDelivr CDN」是成本最低的方案之一：仓库存原图，jsDelivr 做全球缓存分发，零费用，且所有内容有 Git 历史可追溯。我们在 OpenClaw 的几个自动化写作流程里用了这套方案大半年，把实践和坑整理一下。

## 问题

核心需求就三条：

1. 图片链接稳定、可长期引用；
2. 能被脚本 / MCP 工具自动化上传并返回 URL；
3. 尽量免费。

直接用 GitHub raw 链接（raw.githubusercontent.com）的问题是：无 CDN、国内访问不稳定、速率限制严格，自动化场景容易撞墙。所以需要一层 CDN，而 jsDelivr 恰好能直接读取 GitHub 仓库内容并缓存。

## 做法

步骤很短：

1. 新建一个公开仓库，比如 `assets`，目录按 `images/年/月` 组织；
2. 上传图片后，URL 格式固定为：
   `https://cdn.jsdelivr.net/gh/<user>/assets@<ref>/<path>`
   其中 `<ref>` 可以是分支、tag 或 commit hash；
3. 缓存策略：对外引用优先用 tag 或 commit hash（不可变），用分支名（如 `@main`）则可能命中旧缓存；
4. 主动刷新缓存走 purge 接口：
   `https://purge.jsdelivr.net/gh/<user>/assets@main/images/2025/06/x.png`；
5. 自动化：写一个几十行的脚本，或直接封装成 MCP 工具，完成「接收图片 → 文件名取 sha256 前 12 位 → git commit + push → 拼 URL → 调 purge → 返回链接」。挂进 OpenClaw 工作流后，Agent 发帖即可自动插图并拿到最终地址。

## 踩坑点

- **国内访问不稳**：jsDelivr 自 2022 年备案问题后，大陆节点时好时坏，这是本方案最大的不确定性。面向国内读者的正式站点建议备好备用线路（如自建 Cloudflare Worker 反代，或前端 onerror 切换源），不要单点依赖。
- **分支引用 + 覆盖写 = 缓存灾难**：同名覆盖后，老 URL 可能长时间返回旧图。解法很简单：文件永不覆盖，一律用内容 hash 作新文件名。
- **大小限制**：GitHub 单文件硬上限 100MB，但 jsDelivr 只分发 20MB 以内的文件，大图先压缩。
- **条款风险**：jsDelivr 定位是开源项目 CDN，把公开仓库当纯图床重度使用属于灰色地带，有仓库被限制的先例。控制体量，别放业务核心资源。
- **隐私**：公开仓库任何人都能翻到全部图片，敏感内容绝对不放。

## 可复用建议

- 文件名 = 内容 hash，天然去重、天然不可变，缓存问题消失一大半；
- 定期打 tag（如 `v2025.1`），文章引用 tag 版 URL，之后整理仓库也不怕破坏老链接；
- 仓库里维护一个 `manifest.json` 索引（文件名 → 用途/来源），方便脚本反查；
- 把上传逻辑封装成 MCP 工具暴露给 Agent，比让它自己拼 git 命令可靠得多；
- 心智模型摆正：GitHub 是存储源，jsDelivr 只是缓存层。CDN 抖了换一层即可，源永远在。

## 总结

GitHub + jsDelivr 适合个人博客、社区帖子、Agent 产出的非关键性配图：成本为零、链路简单、全自动化友好。但它不是企业级方案——国内可达性无 SLA，条款有灰色空间。工程上的正确姿势是：hash 命名保证不可变，tag 引用保证稳定，purge 接口兜底刷新，备用线路应对抖动。把这套组合拳跑通后，图床这件事基本可以从心智负担清单里划掉了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/65043d6734ebcb95.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/8e502fd3224e9084.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/8534725fcbf0a884.png)

