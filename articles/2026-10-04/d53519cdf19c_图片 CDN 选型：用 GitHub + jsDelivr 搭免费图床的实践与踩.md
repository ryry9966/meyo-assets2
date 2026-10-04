---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的实践与踩坑
feedId: 40382
source: 综合讨论
publishedAt: 2026-10-04
---

## 背景

在 OpenClaw 的工作流里，agent 生成的产物经常带图：自动化写的博客、插件 README、排障报告里的截图。图片放哪是个绕不开的问题——对象存储要付费且国内 CDN 牵扯备案，本地相对路径一旦 markdown 发出去就失效。GitHub 仓库 + jsDelivr 是一套零成本的老方案，最近我把它接进了自己的自动化流水线，把过程和坑记录一下。

## 问题

核心诉求有三条：

1. 生成的 markdown 里要有一个公网可访问、相对稳定的图片 URL；
2. `raw.githubusercontent.com` 在国内访问慢且不稳，不能直接用；
3. 上传必须可编程。agent 产图是批量行为，手动拖拽上传不现实。

## 做法

**第一步：建仓库。** 新建一个 public 仓库，比如 `assets`，按日期分目录（`imgs/2025/06/`），方便管理和回溯。

**第二步：确定 URL 规则。**

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/imgs/<path>
```

`@main` 引用分支，`@v1.0` 引用 tag。分支缓存周期短、tag 缓存更长，但只要覆盖同名文件，缓存都不会主动失效——这个后面细说。

**第三步：走 API 自动上传。** 用 GitHub Contents API，十几行就够：

```python
import base64, hashlib, requests

def upload(token, owner, repo, data: bytes, ext=".webp", branch="main"):
    name = hashlib.sha1(data).hexdigest()[:10] + ext
    path = f"imgs/{name}"
    r = requests.put(
        f"https://api.github.com/repos/{owner}/{repo}/contents/{path}",
        headers={"Authorization": f"Bearer {token}"},
        json={"message": f"upload {name}",
              "branch": branch,
              "content": base64.b64encode(data).decode()})
    r.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@{branch}/{path}"
```

**第四步：封装成工具。** 把这个函数包成一个 MCP tool（比如 `image_upload`），OpenClaw 的所有工作流共享同一个上传出口，agent 写文时直接调用拿 URL。上传前先压缩转 webp，单图控制在几百 KB。

## 踩坑点

- **缓存导致"改了没生效"。** jsDelivr 边缘节点缓存很凶，覆盖同名文件后 URL 不变、内容不更新，社区流传的 purge 接口时灵时不灵。解法很朴素：哈希命名，文件只写一次，永不覆盖。
- **国内可用性有波动。** jsDelivr 在大陆的可达性这几年几经反复，不同地区、运营商体验不一致。所以别把关键资产只押在它上面——GitHub 仓库才是存储层，jsDelivr 只是缓存层。路径结构不变，哪天要切换，把域名批量替换成 raw 地址或自建反代即可。
- **规模边界。** 单文件超过 50MB 直接不服务，仓库堆到 GB 级也会出问题。个人文档、博客量级没问题，但别拿去给线上服务做生产图床，jsDelivr 明确不建议大规模 hotlink，滥用会被限制。
- **公开仓库无隐私。** 所有人都能枚举你的图片，含 token、内网地址的截图务必先脱敏。
- **API 限速。** 认证后 5000 次/小时，个人够用，批量迁移历史图片时分批跑。

## 可复用建议

1. 文件名 = 内容哈希前几位 + 扩展名，天然去重，也天然绕开缓存失效问题；
2. 本地维护一份 `manifest.json`（原图路径 → CDN URL），agent 二次引用时先查表，避免重复上传；
3. 所有上传逻辑收敛到一个 MCP 工具或插件里，将来换图床只改一处；
4. 发布版内容建议把 `@main` 换成 `@tag` 引用，URL 不可变，更稳。

## 总结

这套方案一句话：**免费、够用、有边界**。适合"低频写、中频读"的文档与 agent 产物场景；记住两条原则——GitHub 是存储层，jsDelivr 只是加速层；URL 设计时就保证域名可替换。做到这两点，零成本图床用起来心里是有底的。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/2a4f48a059bda5d7.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/441cc53a7c0b75b9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-04/302572a1d4cd5445.png)

