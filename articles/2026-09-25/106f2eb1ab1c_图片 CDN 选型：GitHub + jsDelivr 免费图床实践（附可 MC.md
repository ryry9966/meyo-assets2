---
title: 图片 CDN 选型：GitHub + jsDelivr 免费图床实践（附可 MCP 化的上传脚本）
feedId: 38881
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

写插件文档、给 Agent 回复拼图文卡片、往 README 塞架构图——图床是绕不开的小基建。个人项目量级不大：一天几十张、单张几百 KB。可选方案无非三类：对象存储（COS/OSS，要实名，流量费虽少但心智负担高）、各类免费图床（说倒就倒）、GitHub 仓库 + jsDelivr CDN。我选了第三条路，跑了三个月，这里记录实际体验。

## 问题

- `raw.githubusercontent.com` 在大陆访问不稳定，直接外链体验差；
- 图片 URL 需要稳定、可缓存、能被 Markdown/HTML 直接引用；
- 上传动作必须能被脚本或 Agent 自动化，不能每次开网页手动拖文件。

## 做法

**1. 建仓。** 新建一个公开仓库如 `cdn-assets`，按 `img/yyyy-mm/` 分目录。注意**必须是 public**，jsDelivr 只回源公开仓库。

**2. 拼 URL。**

```text
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/img/2025-06/demo.webp
```

- `@main` 走分支，jsDelivr 缓存约 12 小时，适合经常更新的场景；
- `@v1.0` 或 commit hash 走版本，缓存近乎永久，适合"传了就不改"的文档配图。

**3. 自动上传。** 不想本地 clone，直接用 GitHub Contents API，一次 PUT 就能推文件进仓库：

```python
import base64, requests

def upload(path, data: bytes, token, repo="me/cdn-assets", branch="main"):
    url = f"https://api.github.com/repos/{repo}/contents/{path}"
    r = requests.put(url, headers={"Authorization": f"Bearer {token}"},
        json={"message": f"upload {path}", "branch": branch,
              "content": base64.b64encode(data).decode()})
    r.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/{repo}@{branch}/{path}"
```

这个函数包一层 schema 就能变成 MCP tool / 插件动作：Agent 生图 → 调 upload → 拿 URL 塞进回复，全链路无人工。文件名建议用内容 hash 或时间戳，天然防重名，也顺便绕开缓存问题。

## 踩坑点

1. **缓存不刷新。** 同名覆盖后 CDN 可能半天还是旧图。要么改名（我的做法），要么手动 purge：`https://purge.jsdelivr.net/gh/<user>/<repo>@main/<path>`。
2. **大陆可用性波动。** `cdn.jsdelivr.net` 域名时好时坏，准备备用域如 `fastly.jsdelivr.net`、`gcore.jsdelivr.net`，前端可做 onerror 切换。这是本方案最大的不确定项，要有预期。
3. **体积限制。** jsDelivr 单文件约 20MB 上限，GitHub 建议仓库别超 1GB。上传前压一遍（pngquant / 转 WebP），别把图床当网盘用。
4. **私有内容绝对不放。** 公开仓库 + 公共 CDN，任何隐私截图、带 token 的图都会被缓存和镜像，删了也未必真删干净。
5. **合规灰区。** jsDelivr 定位是开源项目 CDN，不是官方图床。个人项目挂几百张图没问题，当成服务给大量用户做热点分发就不合适，有被举报封仓库的风险。

## 可复用建议

- 把 upload 封装成独立 MCP tool，任何 Agent 会话都能调，别写死在单个插件里；
- 命名规范固化：`img/{日期}/{内容hash}.{ext}`，方便排查也避开缓存坑；
- 想要"传完自动压缩"，加个 GitHub Actions 在 push 时转 WebP，Agent 侧不用装图像库；
- 稳定性要求高的场景（正式产品文档、对外服务），直接上付费 OSS，这套方案定位是个人/低流量。

## 总结

GitHub + jsDelivr 是典型的"够用就好"方案：零成本、URL 稳定、完全可脚本化，配合 MCP 能让 Agent 自动完成"生图—上传—回链"。代价是大陆访问波动和缓存策略需要自己兜底。理解边界、用在该用的地方，是个很划算的小基建。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/be5fd465bd51abda.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/637b759d2839d7db.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/b6d5da5e596422f9.png)

