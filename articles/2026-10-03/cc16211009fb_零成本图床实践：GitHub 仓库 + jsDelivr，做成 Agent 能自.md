---
title: 零成本图床实践：GitHub 仓库 + jsDelivr，做成 Agent 能自己调用的上传工具
feedId: 40190
source: 综合讨论
publishedAt: 2026-10-03
---

## 背景

写文档、README、博客时经常要贴图。对象存储要绑域名甚至备案，各家免费额度东一个西一个，管理成本不低。GitHub 仓库 + jsDelivr 是个老办法，但配合脚本和 MCP 做成自动化之后，体验比想象中稳，值得重新整理一遍。

## 问题

- `raw.githubusercontent.com` 在大陆基本不可用，直链贴出去别人打不开；
- 需要链接稳定、可版本化，改图不脏缓存；
- 希望写文档时 Agent 能自己传图并拿回 markdown 链接，而不是我手动开网页上传。

## 做法

1. 建一个**公开**仓库，比如 `cdn-assets`，图统一放 `img/` 目录。
2. 链接格式：`https://cdn.jsdelivr.net/gh/<user>/<repo>@<commit或tag>/<path>`。用 commit 哈希做版本，天然不可变，缓存永远不会脏。
3. 上传走 GitHub Contents API，十几行脚本搞定，返回可直接粘贴的链接：

```bash
IMG=pic.webp
REPO="yourname/cdn-assets"
P="img/$(date +%Y%m%d)-$(md5sum "$IMG" | cut -c1-8).webp"
C=$(base64 -w0 "$IMG")
R=$(curl -s -X PUT -H "Authorization: Bearer $GITHUB_TOKEN" \
  "https://api.github.com/repos/$REPO/contents/$P" \
  -d "{\"message\":\"up $IMG\",\"content\":\"$C\"}")
echo "https://cdn.jsdelivr.net/gh/$REPO@$(echo "$R" | jq -r .commit.sha)/$P"
```

4. 把脚本包成 MCP 工具（输入本地路径，输出 CDN 链接），写文档时 Agent 直接调用，产出即贴即用。
5. 上传前压缩：`pngquant` / `cwebp` 转 webp，单张控制在几百 KB。

## 踩坑点

- **覆盖同名文件 ≠ 更新**：jsDelivr 缓存很凶，改了内容但 URL 不变，可能永远是旧图。所以要么带 commit 哈希，要么文件名带日期/哈希，别复用文件名。紧急情况可用 `purge.jsdelivr.net` 强刷。
- **必须公开仓库**：commit 过的内容等于公开发布，截图里有 token、内网地址就直接翻车。养成先脱敏再入库的习惯。
- **体积限制**：GitHub 单文件 100MB 上限（50MB 开始警告），jsDelivr 对过大文件不保证服务，社区经验 20MB 左右就会失败。图床用途下压到 1MB 内基本碰不到。
- **大陆速度不保证**：jsDelivr 国内节点 2022 年后有过变动，直连时快时慢。可以把 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 备在配置里做兜底，重要文档另留一条对象存储备份。
- **定位是灰色地带**：它本质是给开源项目分发文件的 CDN。低频、小体积没问题，别拿来挂高流量热链，有被限制的风险。

## 可复用建议

- 文件名约定 `日期-内容哈希.webp`：天然去重，天然防缓存脏读。
- 脚本要显式校验返回（`jq` 检查 `commit.sha` 是否存在），别让 Agent 拿着 404 链接写进文档。
- token 用最小权限（仅该仓库 contents 读写），放环境变量，不进代码。
- 仓库保持扁平、定期清理，几百 MB 内问题不大。
- 这套方案的定位是"零成本、够用的文档图床"，不是生产 CDN。真有流量了迁对象存储，迁移成本只是换 URL 前缀。

## 总结

GitHub + jsDelivr 解决的是"贴图这件事别打断写作流"：git 管版本、API 可自动化、包成 MCP 工具后 Agent 全程自助。代价是缓存策略要主动适配、隐私要自己把关。按上面的约定用，它是一个非常合格的低运维图床。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/145d50cbdbc16816.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/2e3305c97202cec5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-03/e5ab57f5dd70e5ea.png)

