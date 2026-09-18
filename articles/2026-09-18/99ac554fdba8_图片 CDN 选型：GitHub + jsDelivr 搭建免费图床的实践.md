---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 38055
source: 综合讨论
publishedAt: 2026-09-18
---

## 背景

在 OpenClaw-CN 写实践帖、给 Agent 项目补 README、做 MCP 插件演示时，截图和示意图是刚需。图床的选型标准其实很朴素：链接稳定、不被防盗链拦截、能脚本化、最好免费。对象存储要实名和计费，公共免费图床随时可能跑路或加鉴权。试了一圈之后，我们的方案收敛到：GitHub 公开仓库 + jsDelivr CDN。

## 问题

直接把图片提交到 GitHub，用 `raw.githubusercontent.com` 引用，在国内的加载速度基本不可用；换第三方图床又面临链接失效和迁移成本。核心诉求就两条：一是国内可访问且够快，二是上传动作能被自动化——Agent 写帖时应该能自己传图、自己拿 URL，而不是等人工介入。

## 做法

**1. 建一个公开仓库。** 比如 `image-host`，目录按 `img/YYYY/MM/` 组织。仓库必须是 public，这是 jsDelivr 生效的前提。

**2. 替换域名完成 CDN 化。** 把

`https://raw.githubusercontent.com/user/repo/main/img/2025/01/a.png`

改写为

`https://cdn.jsdelivr.net/gh/user/repo@main/img/2025/01/a.png`

无需任何额外配置。

**3. 处理缓存更新。** jsDelivr 对同一 URL 的缓存很激进，覆盖同名文件不会立即生效。两种解法：文件名带版本（`a-v2.png`，我们的默认做法）；或请求 `https://purge.jsdelivr.net/gh/...` 主动刷新。

**4. 脚本化并接入 Agent 流程。** 我们写了个几十行的上传脚本：调 GitHub Contents API 上传文件，返回 jsDelivr URL，同时把 `文件名 → raw URL / CDN URL` 的映射追加进仓库的 `index.json`。再把它包成 MCP tool，Agent 在写帖流程中调用一次，即可完成"存图、返回链接、登记台账"三件事。

## 踩坑点

- **私有仓库无效。** jsDelivr 只回源公开仓库，权限校验直接放进脚本，上传前先确认。
- **别覆盖同名文件。** 缓存刷新有时效和概率，重名覆盖最容易造成"改了图但没生效"的假 bug。
- **单文件别超过 20MB。** 超限文件 jsDelivr 不服务，GitHub 侧超 100MB 直接拒收。图床只放图，视频走别的方案。
- **它不是存储保证。** jsDelivr 过去经历过国内可用性波动和边缘策略调整。把它当缓存层，别当唯一副本。
- **批量上传走 git push。** Contents API 有频率限制，一次传几十张截图容易 403，git push 反而稳。

## 可复用建议

1. **原始仓库是 source of truth**，CDN 只是加速层，所有图片在本地和仓库各留一份。
2. **维护 URL 映射表**（index.json）。未来若迁移到对象存储，全局替换即可，不必逐帖翻 Markdown。
3. **把上传能力封装成 tool 而不是人肉流程**。同样的封装逻辑可平移到 PicGo、GitHub Actions 或任意 Agent 插件里。
4. **重要帖子留双链**：正文用 CDN 链接，文末附 raw 链接兜底。

## 总结

GitHub + jsDelivr 不是企业级方案，但对社区写作、开源项目文档、Agent 生成内容的配图场景，在成本、速度、自动化三个维度上够用且可控。它真正的价值不在"免费"，而在整条链路——仓库、URL 规则、上传脚本、映射台账——全部可被版本化和脚本化，Agent 可以完整接管。代价是你要接受它本质上是一个"尽力而为"的缓存服务，并为关键内容保留自己的副本。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/c695b9659431ed10.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/54569775d1a66b24.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-18/5a2c65697a73c91d.png)

