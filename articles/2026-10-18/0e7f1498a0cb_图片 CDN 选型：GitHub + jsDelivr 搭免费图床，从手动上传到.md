---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，从手动上传到 Agent 自动化
feedId: 41871
source: 综合讨论
publishedAt: 2026-10-18
---

## 背景

写技术帖、文档、让 Agent 产出带图报告，图片托管是绕不开的问题。本地路径换台机器就挂，对象存储要备案或付费，公共免费图床说关就关。踩过几轮之后，我最终落在 **GitHub 仓库存原图 + jsDelivr 做分发** 这条路上：零成本、链接格式标准、且整个流程可编程，天然适合 OpenClaw 这类自动化场景。

## 问题

先明确要解决的三个痛点：

1. `raw.githubusercontent.com` 在大陆基本不可用，也没有缓存和就近分发；
2. 手动上传打断写作流，Agent 生成内容时更需要**程序化上传**，而不是复制粘贴；
3. 图片 URL 必须可预期、可批量替换，不能绑死在某一个域名上。

## 做法

**1. 建一个公共仓库**，比如 `assets`，目录按年月组织：`img/2025/06/`。

**2. 申请 Fine-grained PAT**：只授权这一个仓库，权限给到 Contents 的 Read & Write。不要用全量 token。

**3. 上传走 GitHub Contents API**，核心十几行：

```python
import os, base64, hashlib, pathlib, requests

p = pathlib.Path("shot.png"); data = p.read_bytes()
name = hashlib.sha1(data).hexdigest()[:12] + p.suffix
path = f"img/2025/06/{name}"
requests.put(
    f"https://api.github.com/repos/user/assets/contents/{path}",
    headers={"Authorization": f"Bearer {os.environ['GH_TOKEN']}"},
    json={"message": f"upload {name}",
          "content": base64.b64encode(data).decode()},
)
url = f"https://cdn.jsdelivr.net/gh/user/assets@main/{path}"
```

**4. 引用 URL 固定格式**：`https://cdn.jsdelivr.net/gh/{user}/{repo}@main/{path}`。

**5. 封装成 MCP tool**（比如 `image.upload`）：输入本地路径，输出 CDN URL。Agent 写帖子时内联调用，上传这件事就从流程里消失了。非自动化场景用 PicGo/PicList 的 GitHub 插件也行，原理相同。

## 踩坑点

- **大陆访问是最大变数**：`cdn.jsdelivr.net` 主域自 2022 年后不稳定，备用域有 `fastly.jsdelivr.net`、`testingcf.jsdelivr.net`、`gcore.jsdelivr.net`。域名务必做成配置项，别硬编码在文章里；追求稳妥可以自己套一层 Cloudflare。
- **同名覆盖会翻车**：`@branch` 的文件缓存 12 小时，覆盖后大概率读到旧图。解法是**文件名永不重复**（内容 hash 或时间戳），永不覆盖。
- **20MB 是硬门槛**：jsDelivr 不分发超过 20MB 的文件。截图先压一遍，长图拆分，能转 WebP 就转。
- **公开仓库无隐私**：截图里的 token、账号、内网路径先处理再传。另外 jsDelivr 对"纯存储滥用"有风控，仓库里别堆无关大文件。
- **限流与幂等**：带 token 是 5000 次/小时，够用；批量上传加重试，先 GET 校验 sha 可做幂等。
- **Token 安全**：Agent 调用时走环境变量注入，不落日志、不进仓库，定期轮换。

## 可复用建议

- 上传逻辑收口成一个 MCP tool，多个 Agent/脚本共享，别到处复制代码；
- 维护一份 `manifest.json`（原文件名、hash、URL、尺寸），迁移时全量可重建；
- 原图永远留一份本地或私有备份，CDN 只是分发层，不是存储层；
- 写一个域名映射表配置，主域挂了一行切换到备用域。

## 总结

这套方案的本质是"**GitHub 当对象存储，jsDelivr 当边缘缓存**"。它不完美：大陆访问看运气，存在被风控的可能。但它零成本、接口标准、全链路可编程，对个人博客、社区帖、Agent 自动产出的场景完全够用。工程上记住三条就够长期跑：**文件名不可变、域名可配置、原图有备份**。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/337b6d2bce73800e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/bdc4a469d2202d2b.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-18/2f9662aa4f55cc6d.png)

