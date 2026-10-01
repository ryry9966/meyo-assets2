---
title: GitHub + jsDelivr 免费图床实践：从手动传图到 Agent 自动化
feedId: 39994
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

写博客、发社区帖、给 MCP 插件配 README 截图，都会遇到同一个问题：图放哪？本地相对路径换个环境就挂；对象存储要实名、有带宽成本；第三方图床有跑路风险。GitHub 仓库 + jsDelivr 是一套零成本方案，而且对 OpenClaw 用户有个额外好处——上传和拼外链这一步，完全可以交给 Agent 自动完成。

## 问题与原理

- `raw.githubusercontent.com` 在大陆基本不可用，直接引用 raw 链接体验很差；
- 把图丢进 issue 靠附件缓存，URL 带时效 token，跨文档引用不可靠；
- jsDelivr 的 GitHub 模式可以把公开仓库文件变成全球分发的 CDN 资源，规则是：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch或tag>/<路径>
```

## 做法步骤

1. **建专用公开仓库**，比如 `imgbed`。私有仓库 jsDelivr 不读取，必须是 public。
2. **命名规范**：文件名用内容哈希或时间戳（如 `shot-1730000000.png`），避免重名覆盖，也让分支缓存问题天然失效。
3. **手动方案**：PicGo 配 GitHub 图床，自定义域名填 jsDelivr 前缀，剪贴板快捷键上传即得外链。
4. **自动化方案**（推荐）：走 GitHub Contents API，几行脚本就能跑通：

```bash
CONTENT=$(base64 -w0 screenshot.png)
curl -s -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/me/imgbed/contents/2025/shot-$(date +%s).png \
  -d "{\"message\":\"upload\",\"content\":\"$CONTENT\"}"
# 返回后拼: https://cdn.jsdelivr.net/gh/me/imgbed@main/2025/shot-xxx.png
```

把它包成 OpenClaw 的 skill 或 MCP tool，之后 Agent 生成截图、画完图，一句话"上传并给我外链"即可。

## 踩坑点

- **大陆访问不稳定**：`cdn.jsdelivr.net` 自 2022 年 ICP 注销后时好时坏。必要时把域名换成 `fastly.jsdelivr.net` / `testingcf.jsdelivr.net` 等镜像，功能一致。
- **缓存更新**：`@main` 这类分支引用缓存约 12 小时，改图不生效是常态；固定 tag/commit 的缓存周期更长，基本可视为不可变。单文件可访问 `purge.jsdelivr.net` + 原路径手动刷新。
- **硬限制**：单文件超过 20MB 不提供服务（GitHub 单文件 100MB 上限，但绑定的是前者）；仓库别堆到 GB 级，它不是网盘。
- **滥用风险**：放大文件、当文件分发站用，可能被 jsDelivr 限制。图床归图床，克制使用。

## 可复用建议

- **源与分发分离**：GitHub 仓库才是源，jsDelivr 只是缓存层。脚本里把 CDN 域名做成变量，随时可切镜像或回退 raw。
- **多源 fallback**：对外发布的关键配图，在 HTML 里给 `img onerror` 换镜像域名的兜底，或构建时批量替换域名。
- **不可变优先**：频繁改动的图（如状态面板截图）别指望缓存刷新，新命名重传一张，旧图留着——反正没有存储成本。
- **场景边界**：适合个人博客、社区帖、Agent 产出物展示；SLA 敏感的生产环境请上正经对象存储，别把业务押在免费服务上。

## 总结

这套方案的本质是"用版本控制当存储，用社区 CDN 当分发"。它免费、天然版本化，尤其契合自动化流水线里"Agent 传图拿外链"的场景。代价是大陆访问的不确定性，所以工程上只需要做两个小设计：**域名可替换、命名不可变**。三行配置换来一套能长期跑的图床，对个人开发者来说，性价比足够高。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/ca8c32c81c94992d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/b202027b3666c300.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/9d1790885bdbf207.png)

