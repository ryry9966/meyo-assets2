---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的自动化实践
feedId: 39426
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

写博客、维护文档、让 Agent 产出带截图的运行报告，都绕不开一个问题：图片放哪。对象存储要备案要花钱，第三方图床寿命不可控，本地相对路径一发布就裂图。对 OpenClaw 这类经常"生成内容 + 引用图片"的工作流来说，一个稳定、可脚本化、零成本的方案很实用。GitHub 仓库 + jsDelivr CDN 是我用了快两年的组合，维护成本几乎为零。

## 问题

需求其实很具体：

1. Markdown 里的图片链接在任何平台都能访问；
2. 图片上传能被脚本/MCP 工具接管，Agent 可以自主完成"截图 → 上传 → 拿链接"；
3. 链接可缓存、可版本化，方便日后替换；
4. 不想为此单独维护服务器，也不想付月费。

## 做法

1. 建一个公开仓库（如 `pics`），按 `年月/主题/` 组织目录；
2. 引用格式：`https://cdn.jsdelivr.net/gh/<user>/pics@main/<path>`。`@main` 跟随分支（有缓存），`@v1.0` 这类 tag 不可变，正式发布的文章建议用版本号；
3. 写一个几十行的脚本完成入库、push、输出 URL（压缩步骤可前置，或在脚本里接 pngquant）：

```bash
#!/usr/bin/env bash
repo="yourname/pics"; branch="main"
f="$1"; dest="$(date +%Y%m)/$(basename "$f")"
cp "$f" "$dest"
git add "$dest" && git commit -m "img: $dest" && git push
echo "https://cdn.jsdelivr.net/gh/$repo@$branch/$dest"
```

4. 用细粒度 PAT 做凭证，只授权这一个仓库，权限最小化；
5. 把脚本包成 MCP 工具，暴露 `upload_image(path) -> url`。这样 Agent 写周报、发博客时能自己传图拿链接，配图真正成为自动化流程的一环。

## 踩坑点

- **国内可达性**：jsDelivr 的国内解析历史上多次波动。先在常用网络实测，备好 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备用域名，必要时切 CNAME。
- **缓存不实时**：`@main` 路径有 CDN 缓存，覆盖同名文件后不生效。要么文件名带日期/哈希永不覆盖，要么打 purge 接口（`https://purge.jsdelivr.net/gh/...`）。我倾向前者，幂等且省心。
- **体积限制**：GitHub 单文件超 100MB 直接拒收，jsDelivr 对 gh 文件也有 20MB 上限。上传前统一压缩（转 WebP 或 pngquant），控制在 1MB 内。
- **隐私**：仓库是公开的。Agent 截图里常带 token、内网地址，入库前必须打码或裁剪——这一步建议写进 MCP 工具里强制执行，别指望人肉记得。
- **合规边界**：jsDelivr 定位是开源项目 CDN，拿来做重流量图床属于灰色地带。个人文档、博客量级没问题，别拿去挂论坛热图。

## 可复用建议

- 目录约定 `YYYYMM/主题/文件名`，文件名带日期，天然防缓存冲突；
- 文章发布时给仓库打一个 tag，历史链接永远有效；
- 仓库本身就是备份，再定期 `git clone --mirror` 到第二处更稳；
- 将来若要面向大量读者，迁移对象存储也只是换个 URL 前缀——git 历史让迁移成本很低。

## 总结

GitHub + jsDelivr 不是新玩法，胜在零成本、可版本化、与 Git 工作流天然融合。对 OpenClaw 用户来说，它最大的价值是能被脚本和 MCP 工具完整接管，让"配图"从手工活变成 Agent 自动链路里的普通一环。适合个人文档、博客、报告的量级；重流量场景请直接上对象存储，不必硬撑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/69e39d61da844e45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b89d364b74299f79.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/d4395cbdbc716ad6.png)

