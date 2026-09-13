---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践与踩坑
feedId: 37454
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

写博客、项目文档，或者让 agent 自动产出带图报告时，绕不开一个问题：图片放哪。我试过几种方案——免费图床不是关站就是加防盗链，对象存储稳定但要维护计费和密钥。GitHub 仓库本身是可靠的免费存储，但 `raw.githubusercontent.com` 直链在国内基本不可用，也不适合当 CDN。jsDelivr 在 GitHub 之上做了边缘缓存，等于免费把公开仓库变成 CDN 资源，这就是本文的基础组合。

## 问题

我的核心需求只有四条：

1. 免费，且不太可能突然跑路；
2. URL 稳定，可被 Markdown 直接引用；
3. 支持自动化——agent 写文档时能自己传图并拿到链接；
4. 国内可访问性“够用”即可，不追求极致速度。

## 做法

**第一步：建仓库与令牌。** 新建一个公开仓库（如 `images`），目录按 `年/月` 组织。创建 fine-grained PAT，只授权这一个仓库、只给 Contents 读写权限——权限收到这个粒度，泄露了损失也可控。

**第二步：上传。** 直接调 GitHub Contents API，无需额外依赖：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/images/contents/2025/06/a1b2c3d4.png \
  -d '{"message":"upload","content":"'"$(base64 -w0 pic.png)"'"}'
```

也可以用 PicGo 的 GitHub 插件，配置成本更低。

**第三步：引用。** 通过 jsDelivr 的 URL 规则访问：

```
https://cdn.jsdelivr.net/gh/USER/images@main/2025/06/a1b2c3d4.png
```

更新同名文件后如需立即生效，请求一次 `https://purge.jsdelivr.net/gh/...` 刷新缓存。

**第四步：自动化。** 我把上传逻辑封装成一个 MCP tool：输入本地图片路径，输出 CDN URL。这样 OpenClaw 在写文档、发帖时可以自行调用，图片上传和正文写作在同一条链路里完成，不再需要手工中转。

## 踩坑点

- **国内节点波动。** jsDelivr 大陆访问近年有过明显不稳定期，上线前务必用 `curl -w "%{time_total}"` 从目标读者的网络环境实测。备选域名 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可以在代码里做 fallback。
- **同名覆盖会命中旧缓存。** 文件名建议用内容 hash（比如 sha256 前 8 位），天然规避缓存一致性问题，也免得频繁手动 purge。
- **别当网盘用。** jsDelivr 对超过 20MB 的文件不提供服务；大量存放非文档资源有触发仓库滥用的风险，保持“博客/文档配图”这个定位比较安全。
- **用户名和仓库名是 URL 的一部分。** 定好之后不要改，改了全部外链失效。
- **Token 安全。** 不要把 PAT 硬编码进公开脚本或提交到任何仓库，统一用环境变量注入。

## 可复用建议

1. 命名规范：`年/月/{hash}.png`，上传前先 GET 判断文件是否已存在，保证幂等。
2. 正文引用 jsDelivr 链接，文档尾部注释保留 raw 链接，jsDelivr 万一不可用时有兜底。
3. MCP tool 里加超时和重试，上传失败时让 agent 明确报错，而不是静默丢图。
4. 图片量大或有商业用途时，老老实实上对象存储 + 自建 CDN，jsDelivr 的服务条款并不为重度使用背书。

## 总结

GitHub + jsDelivr 这套组合，本质是用免费存储换免费分发，代价是稳定性要自己兜底、用量要自己克制。对个人博客、开源项目文档、agent 产出物配图这类低频场景，它足够好用；再配合一个几十行的 MCP 上传工具，整条“写作—配图—发布”链路可以完全自动化。免费方案没有银弹，但把边界想清楚，它就是当前性价比最高的选择之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/64384ff068993d93.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/3772a2b3d58805f5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/cddc52d4dbf80c5e.png)

