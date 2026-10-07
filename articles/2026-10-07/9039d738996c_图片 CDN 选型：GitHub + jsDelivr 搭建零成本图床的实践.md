---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建零成本图床的实践
feedId: 40805
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

在社区写帖、给 Agent 生成报告、写插件文档时，图片托管是个绕不开的小问题。本地 Markdown 里的相对路径图片发出去就裂；买对象存储要实名、要域名、要维护；随手用免费图床又担心跑路。我们的场景很典型：OpenClaw 工作流里，Agent 的截图、生成的示意图、排障过程图，需要一段稳定外链写进 Markdown。评估下来，GitHub 公开仓库 + jsDelivr 这套组合，成本为零、链路可控、可完全脚本化，天然适合自动化场景。

## 问题

核心诉求拆开是四条：

1. 链接长期稳定，不依赖本机路径；
2. 走 CDN 加速，帖子里的图别加载半天；
3. 上传能被脚本或 Agent 调用，而不是手动开网页传；
4. 零费用、零维护负担。

GitHub Raw 链接满足 1 和 3，但它不是 CDN，国内访问慢且有速率限制。jsDelivr 正好补上这块：它镜像 GitHub 仓库内容并分发到全球节点。

## 做法

**第一步：建仓库。** 新建一个公开仓库，比如 `images`，按用途建 `posts/` 等子目录。

**第二步：确认 URL 规则。** 外链格式为：

```
https://cdn.jsdelivr.net/gh/<用户名>/<仓库名>@<分支>/<路径>
```

例如 `https://cdn.jsdelivr.net/gh/foo/images@main/posts/20250615-a1b2.webp`。文件传上去后等一两分钟再验证外链，新文件有传播延迟。

**第三步：把上传做成脚本。** 不想本地装 git 环境的话，GitHub Contents API 一把梭：压缩、base64、PUT 上传、拼 URL，四步。给 OpenClaw 挂 skill 或 MCP 工具时，这就是全部逻辑：

```bash
IMG="$1"; NAME="$(date +%Y%m%d)-$(md5sum "$IMG" | cut -c1-8).webp"
cwebp -q 80 "$IMG" -o "/tmp/$NAME"
curl -s -X PUT -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/<user>/images/contents/posts/$NAME" \
  -d "{\"message\":\"add $NAME\",\"content\":\"$(base64 -w0 /tmp/$NAME)\"}" >/dev/null
echo "https://cdn.jsdelivr.net/gh/<user>/images@main/posts/$NAME"
```

Token 用 fine-grained PAT，只授这一个仓库的 `contents:write`，泄漏了影响也有限。

## 踩坑点

- **缓存不失效是最大的坑。** 同名文件覆盖后，jsDelivr 很可能仍返回旧图。解法是文件名带日期 + 内容哈希，永远用新名字，基本不用考虑刷新；真要强刷，走 `purge.jsdelivr.net` 加对应路径。
- **国内可用性会波动。** 主域名偶发抽风时，`fastly.jsdelivr.net`、`gcore.jsdelivr.net` 可平替，路径不变只换域名。但帖子里固定写一个域名，别一图多链。
- **仓库是公开的。** 截图里的 token、内网 IP、聊天记录先打码再传。建议在 Agent 的上传 skill 里加一步脱敏或提醒。
- **体积红线。** 单文件 100MB 硬限，仓库建议压在几百 MB 内。另外 jsDelivr 的定位是开源项目文件分发，个人图床属于"能跑但别滥用"，量大后建议迁移对象存储。
- **API 409 冲突。** 同名文件重复上传会报 409，哈希命名天然规避。

## 可复用建议

- 命名统一为 `日期-短哈希.webp`，一劳永逸解决缓存和冲突；
- 上传前统一压缩：WebP、质量 80、长边 1600px，社区帖子的图 200KB 以内足够；
- 把"压缩 → 上传 → 返回 URL"封装成 OpenClaw 的 skill/MCP 工具，Agent 写帖时直接产出外链，全程无人值守；
- GitHub 仓库页留作备份和浏览入口，两份保障。

## 总结

这套方案的本质是"GitHub 当存储，jsDelivr 当加速层"：零成本、可脚本化、对自动化友好，很适合社区写作和 Agent 内容生产的量级。但要正视边界——公开性、缓存策略、国内可用性波动、滥用风险。把命名、压缩、权限这几件小事做规范，它能稳定服务很久；哪天量级上去了，迁移到对象存储也只是换个 URL 前缀的事。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/13df0277af2d03b3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/6cd2055633d9c3ff.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/fd7c767f4e0eb363.png)

