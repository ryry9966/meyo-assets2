---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的工程实践
feedId: 38528
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

在 Agent / MCP 工作流里，图片外链是个高频小需求：Bot 回复里贴运行截图、自动化报告引用生成的图表、插件文档引用示例图。这些场景的共性是：图片必须是一个**可公网直链的 URL**，而不是本地路径。

可选方案无非几种：对象存储（要钱，自定义域名还要备案）、自建服务（要运维）、本地起 HTTP 服务（跨设备不可用）。对我们这种写多读少、图片小、频率低的场景，用公开 GitHub 仓库 + jsDelivr 的 CDN 镜像，成本为零，链路也足够简单。

## 问题

直接用 `raw.githubusercontent.com` 有两个硬伤：国内直连不稳定，且没有边缘缓存，多人访问同一张图会反复回源。jsDelivr 的 `/gh/` 端点正好补上这两点——它会镜像仓库内容并做全球缓存：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

## 做法

1. **建一个公开仓库**，如 `images`，按 `yyyy/mm` 目录归档，控制单目录文件数。
2. **用 GitHub Contents API 写入**。核心逻辑不到二十行：

```python
import base64, hashlib, time, requests

def upload(local_path: str) -> str:
    data = open(local_path, "rb").read()
    name = f"{int(time.time())}-{hashlib.sha1(data).hexdigest()[:8]}.png"
    path = f"2025/06/{name}"
    r = requests.put(
        f"https://api.github.com/repos/{OWNER}/{REPO}/contents/{path}",
        headers={"Authorization": f"Bearer {TOKEN}"},
        json={"message": f"upload {name}",
              "content": base64.b64encode(data).decode()})
    r.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/{OWNER}/{REPO}@main/{path}"
```

3. **封装成 MCP tool**：输入本地文件路径或 base64，返回 CDN URL。任何接了 MCP 的 Agent 都能直接调用，不需要每家 Bot 各写一遍上传逻辑。
4. **维护一个 `index.json`**：记录原文件名、内容哈希、最终 URL，用于去重和事后清理。

## 踩坑点

- **缓存是第一坑**。同名覆盖后 jsDelivr 不会立刻更新，边缘节点缓存期很长。根治办法是**时间戳 + 哈希的唯一文件名**，让 URL 天然不可变；`purge.jsdelivr.net` 的刷新接口只当补救手段。
- **文件大小上限 20MB**，且 Git LFS 文件不走 jsDelivr，截图和图表没问题，别拿它存视频。
- **国内可用性会波动**。jsDelivr 提供了 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等多线路域名，URL 拼装逻辑里把域名做成配置项，坏了能一键切换，`raw` 链接做兜底。
- **仓库是公开的**。任何敏感截图、带内网地址的图都不要进这个仓库——把它当成"发布目录"，不是"暂存区"。
- **API 限流**：带 token 是 5000 次/小时，批量任务的自动化要做重试和队列，别打满。

## 可复用建议

- 把"上传 → 拼唯一 URL → 写索引"整体做成一个 MCP tool，是这套方案最有价值的沉淀，Agent 侧只看到一个 `upload_image(path) -> url`。
- 域名、仓库、分支全部走配置，换源或迁移到对象存储时只改一处。
- 图片入库前做一次压缩（如 `sharp` 转 webp），仓库体积和加载速度都受益。

## 总结

GitHub + jsDelivr 图床适合**公开、小体积、低频写**的 Agent 产物分发：零成本、和 Git 工作流天然契合、通过 MCP 封装后接入成本几乎为零。它替代不了生产环境的对象存储，但作为自动化工作流的"图片出口"，是目前性价比最高的选择之一。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/fef7a383fe5042e8.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/ebc9ea16cc199fd4.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/a96a93f3bb42a533.png)

