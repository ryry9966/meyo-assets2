---
title: 图片 CDN 选型：GitHub + jsDelivr 零成本图床的落地实践
feedId: 42247
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

写文档、发博客、让 Agent 产出带截图的报告，都绕不开同一件事：Markdown 里的图片链接要稳定、可脚本化、不花钱。国内图床普遍有防盗链或审核流程，SM.MS 免费额度有限，imgur 直连不稳。对 OpenClaw 用户来说，更现实的诉求是：**上传动作能被脚本调用，链接生成是确定性的**，方便 Agent 直接消费。

## 问题

常见的免费方案各有硬伤：平台图床链接不可控、随时可能加防盗链；对象存储要备案、要绑卡；自建需要服务器。而多数场景（个人博客、文档、Agent 低频配图）其实只需要一个"把本地图变成可外链 URL"的自动化节点。

## 做法：GitHub 做源，jsDelivr 做分发

思路很简单：公开仓库存图，用 jsDelivr 的 gh CDN 做全球分发，URL 格式：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<ref>/<path>
```

`<ref>` 用分支名（如 `main`）方便覆盖更新，用 commit hash 则接近不可变链接。

三步落地：

1. 建一个公开仓库，比如 `assets`，内部按 `img/YYYYMM/` 分目录。
2. 写上传脚本，走 GitHub Contents API，做到幂等：先 GET 判断文件是否已存在，404 才 PUT，避免重复 commit 刷提交历史。文件名拼上内容 sha1 前 8 位，天然防覆盖冲突。
3. 脚本最后输出 jsDelivr 链接，再封装成 OpenClaw 侧的一个小工具/CLI，Agent 拿到本地图片路径就能返回 Markdown 图片链接。

核心脚本骨架：

```bash
upload_img() {
  f=$1
  name=$(basename "$f")
  sha=$(sha1sum "$f" | cut -c1-8)
  path="img/$(date +%Y%m)-${sha}-${name}"
  api="https://api.github.com/repos/<user>/assets/contents/$path"
  # 幂等：已存在则跳过上传
  [ "$(curl -s -o /dev/null -w '%{http_code}' "$api")" = "200" ] || \
  curl -s -X PUT "$api" \
    -H "Authorization: Bearer $GITHUB_TOKEN" \
    -d "{\"message\":\"add $name\",\"content\":\"$(base64 -w0 "$f")\"}" > /dev/null
  echo "![img](https://cdn.jsdelivr.net/gh/<user>/assets@main/$path)"
}
```

## 踩坑点

- **国内可用性波动**：jsDelivr 的国内解析这几年时好时坏，2022 年前后经历过长时间不可用。把它定位成"低成本分发层"而不是"唯一依赖"，博客类小流量可以接受，正式产品别押注。
- **分支名 URL 有约 12 小时缓存**：覆盖同名文件后旧图可能还在。要么走 purge（`https://purge.jsdelivr.net/gh/...`），要么用哈希文件名走"只增不改"，后者省心得多。
- **单文件上限 50MB**，仓库也别当网盘用；大量二进制会让 git 仓库膨胀、克隆变慢。
- **API 限流**：匿名 60 次/小时，务必带 token（fine-grained，只给该仓库 contents 读写权限即可）。
- **公开即公开**：仓库里的图任何人可见，别传敏感截图；用量也要克制，别往大规模分发的方向用。

## 可复用建议

- 链接先用 `@main` 起步，正式发布时替换成 commit hash，得到不可变引用。
- 维护一份 `manifest.json`（本地路径 → 远程 URL → sha），将来迁移到 R2/OSS 时是一张现成对照表。
- 上传逻辑做成独立小工具（shell/Node 均可），再包一层 MCP 工具描述，Agent 侧调用就是一行指令的事。
- 给脚本加 `--purge` 子命令，覆盖场景一键清缓存。

## 总结

GitHub + jsDelivr 是一套"零成本、全脚本化、链接确定"的图床组合，代价是国内可用性不保证、缓存策略需要绕着设计。它适合个人博客、文档和 Agent 产出的低频配图，不适合当生产级 CDN。对这个社区更有价值的做法是：把"本地图片 → 上传 → 返回链接"沉淀成可复用的自动化节点——选型以后再换，上层工作流不用动。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/ff73fb99a3b6d81e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/7518e675dcf5a5fb.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/9843a4e7572034c5.png)

