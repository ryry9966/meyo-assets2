---
title: GitHub + jsDelivr 免费图床：从手动上传到 Agent 自动外链
feedId: 40617
source: 综合讨论
publishedAt: 2026-10-06
---

## 背景

在 OpenClaw-CN 写教程、贴 Agent 运行截图、给 MCP 插件写文档，图床是绕不开的基础设施。对象存储要绑卡，有的还要备案；公共免费图床跑路风险高，链接说失效就失效。GitHub 仓库 + jsDelivr 是个被反复提起的老方案，这次我把它落到自动化场景里重新走了一遍，目标是：Agent 截图之后，一条命令拿到稳定外链。

## 问题

直接用 `raw.githubusercontent.com` 有两个硬伤：国内访问不稳定，且有速率限制，做外链体验很差。手动「上传 → push → 拼 URL」流程太长，Agent 产出的截图没法自动转成可分享链接。另外缓存更新、仓库体积这些坑，多数教程不会提前讲。

## 做法

**1. 建仓与目录规划。** 新建一个 public 仓库（public 是 jsDelivr 的硬性前提），按 `img/YYYYMM/` 归档。

**2. URL 规则。** 外链格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/...`。`@main` 写起来方便但有缓存延迟；要稳定就在仓库里打 git tag，用 `@v1.0` 这类版本固定。

**3. 上传脚本。** 走 GitHub Contents API + fine-grained PAT（只授予该仓库 `contents:write`），不依赖本地 git 环境：

```bash
#!/usr/bin/env bash
# img.sh <本地图片>，依赖 jq
set -e
F=$1
KEY="img/$(date +%Y%m)/$(date +%s)-$(md5sum "$F" | cut -c1-8).${F##*.}"
jq -n --arg c "$(base64 -w0 "$F")" \
  '{message:"upload",content:$c}' > /tmp/p.json
curl -sf -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  "https://api.github.com/repos/$GH_USER/pics/contents/$KEY" \
  -d @/tmp/p.json >/dev/null
echo "https://cdn.jsdelivr.net/gh/$GH_USER/pics@main/$KEY"
```

**4. 接入 OpenClaw。** 把脚本封装成一个技能或 MCP 工具（比如 `upload_image`），Agent 跑完任务截图后直接调用，返回的外链写进回复或文档。多机环境只需要一个 token，不需要任何一台机器装 git。

## 踩坑点

- 仓库必须 public，private 仓库 jsDelivr 拉不到，报 403。
- jsDelivr 不是网盘：单图压到 1MB 内（建议转 WebP），仓库别塞大文件，有仓库因滥用被拉黑的先例。
- `@main` 改图后 CDN 有缓存延迟。可以请求 `https://purge.jsdelivr.net/gh/<user>/<repo>@main/<path>` 主动刷新；更省心的做法是文件名带时间戳 + 哈希，天然不撞缓存。
- 国内访问有过波动史：`cdn.jsdelivr.net` 偶发不稳，脚本里可以同时生成 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 备用域名。
- 匿名调 Contents API 只有 60 次/小时，务必带 token（5000/小时）。
- public 仓库人人可见，敏感截图、内网信息、带密钥的终端输出一律别传。

## 可复用建议

- 命名规则统一为「时间戳 + 内容哈希」，顺便规避中文文件名的 URL 编码问题。
- 加一个 GitHub Action 自动压缩图片，push 即优化，不占人工流程。
- 心态上把 GitHub 当源、jsDelivr 当缓存层：CDN 抽风时换域名或直连 raw 都能救急，图永远不丢。
- 正式文档里引用带 tag 的版本 URL，图不会被后续改动影响。
- PAT 最小权限：fine-grained token 只授权这一个仓库，别用全域 classic token。

## 总结

这套方案零成本、可控、可自动化。配合脚本或 MCP 工具，从截图到外链几秒完成，很适合博客、社区帖、插件文档这类低频写场景。但它本质是「借道」开源基础设施，不适合高并发业务。图床没有银弹，按流量和重要度选型即可。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/0e9de53cf071f73b.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/0500fbed8e341660.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-06/27c4d6d8854d0e41.png)

