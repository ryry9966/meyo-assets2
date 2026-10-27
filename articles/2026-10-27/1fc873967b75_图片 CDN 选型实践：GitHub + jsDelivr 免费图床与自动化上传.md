---
title: 图片 CDN 选型实践：GitHub + jsDelivr 免费图床与自动化上传
feedId: 42763
source: 综合讨论
publishedAt: 2026-10-27
---

# 背景

写技术帖、做插件文档、跑 Agent 工作流，图片是绕不开的素材：架构图、截图、生成的示意图。图床选型看着小，坑却不少：商业对象存储要付费甚至备案，免费图床动不动跑路或加防盗链，Notion / 公众号的图链出去还有 referer 校验。我最终选了 GitHub 仓库 + jsDelivr 的组合，跑了半年多，把做法和踩过的坑整理如下。

# 问题

需求就四条：

1. 免费，流量上不心疼
2. 链接稳定，Markdown 可直接引用
3. 国内访问速度可接受
4. 能被脚本 / Agent 自动化调用——上传不能每次手动

# 做法

**1. 建一个公开仓库**

专门放图，比如 `username/assets`，按 `年/月` 建目录。仓库必须 public，这是 jsDelivr 生效的前提。

**2. 引用 URL 格式**

```text
https://cdn.jsdelivr.net/gh/用户名/仓库名@分支/路径/文件名.png
```

`@main` 跟随分支，`@commit hash` 把版本钉死。图片建议默认用不可变文件名，原因见踩坑点。

**3. 自动化上传**

核心是 GitHub Contents API，一个 PUT 搞定：

```bash
# content 为文件 base64，PAT 只授予该仓库 contents:write
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/user/assets/contents/2025/06/demo.png \
  -d '{"message":"add demo.png","content":"<base64>"}'
```

拿到返回的 `download_url` 后拼出 jsDelivr 地址即可。我把它封装成了 MCP 工具：Agent 生成图片 → 压成 WebP → base64 → PUT 上传 → 返回 CDN 链接，直接写进 Markdown，全链路无人工介入。

# 踩坑点

- **缓存不更新**：同名覆盖上传后，CDN 可能仍返回旧图。最稳的办法是不覆盖——文件名带时间戳或内容 hash，每次都是新 URL。
- **仓库体积**：单文件超 100MB 直接被拒，仓库建议控制在 1GB 内。截图先压 WebP，通常能省 60% 以上体积。
- **误传无法真正删除**：删文件不等于删历史，git 历史和 CDN 缓存里的图短期都还在。上传前先自查截图里有没有 token、内网面板地址。
- **国内可达性有波动**：jsDelivr 的可用性这几年起起落落。留两个后手：备用域名（如 fastly.jsdelivr.net）用 `onerror` 切换；`raw.githubusercontent.com` 兜底。
- **API 限流**：未认证 60 次/小时；fine-grained PAT 可到 5000 次/小时，但 token 权限只给这一个仓库，别图省事用全局 key。

# 可复用建议

- 把「压缩 → 上传 → 返回链接」封装成 MCP 工具或 CLI 脚本，一次投入长期收益
- 文件名不可变（hash/时间戳），放弃"覆盖更新"的执念
- 仓库里维护一份 `index.json` 记录已传图片元信息，Agent 查重和引用都方便
- 给仓库加个开源 license，把授权说清楚，总没坏处

# 总结

GitHub + jsDelivr 不是图床的终极方案，它本质是"把 CDN 当作 git 仓库的缓存层"。适合博客、文档、Agent 产物这类轻量、公开、低频写的场景；不适合大文件、隐私内容和重度流量。选型判断就三条：图片能否公开、是否需要脚本化、能否接受偶发的国内波动。都能接受，这套基本就是零成本里的最优解。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/010d59b420cd1a20.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/15b22105eb28ca80.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-27/10be49eae90380b8.png)

