---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床，并接进 Agent 工作流
feedId: 38836
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景

写技术文档、自动化发博客、Agent 生成带图报告时，经常需要"上传一张图，拿到一个可外链的 URL"。可选方案里：对象存储（OSS/COS）稳定但要实名、配置繁琐、按量付费；公共图床服务随时可能跑路或加防盗链；GitHub 的 raw 域名国内直连慢且无缓存。

最后我选了 GitHub 仓库 + jsDelivr 的组合：**GitHub 做存储源，jsDelivr 做全球缓存层**。零成本，且整条链路纯 API，可以很自然地封装成 MCP tool 或 OpenClaw skill。

## 做法

1. 建一个公开仓库（如 `img-bed`），纯放图片，不要和代码混用。
2. 上传图片，三种方式按场景选：
   - 网页拖拽：一次性手动场景；
   - git push：本地批量导入；
   - GitHub Contents API：脚本自动化首选。
3. 通过 jsDelivr 引用：
   `https://cdn.jsdelivr.net/gh/<user>/<repo>@main/images/2024/a1b2c3.png`
4. 接进自动化：把"上传 → 拿 URL"封装成一个 tool，Agent 写文章时直接产出可用外链，不再人工传图。

最小上传脚本（Python）：

```python
import base64, requests, hashlib

def upload(token, owner, repo, path, data: bytes):
    url = f"https://api.github.com/repos/{owner}/{repo}/contents/{path}"
    sha = hashlib.sha256(data).hexdigest()[:8]
    name = f"{sha}.png"
    r = requests.put(f"{url}/{name}", headers={"Authorization": f"Bearer {token}"},
        json={"message": f"add {name}",
              "content": base64.b64encode(data).decode()})
    r.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/{owner}/{repo}@main/{path}/{name}"
```

## 踩坑点

1. **缓存不失效**：jsDelivr 对同名文件缓存很狠，覆盖上传后旧 URL 可能几天不更新。解法：文件名带内容 hash 或时间戳，永不覆盖；确需刷新走 `purge.jsdelivr.net/gh/...`。
2. **20MB 单文件上限**：超限文件 jsDelivr 不分发。GIF 录屏注意先压缩。
3. **把仓库当数据库用会翻车**：API 未认证 60 次/小时，带 token 5000 次/小时。批量任务务必带 token，图片多了按 `images/yyyy/mm/` 分目录。
4. **国内可用性有波动**：jsDelivr 大陆节点历史上有过中断期。别把关键链路单点押在它上面，建议预埋 fallback（套一层 Cloudflare，或保存双链接）。
5. **公开仓库 = 全网可见**：截图带 token、内网地址的先打码，敏感图片别走这条链路。

## 可复用建议

- **命名即索引**：`{sha8}-{slug}.{ext}`，天然去重，再配一个 `manifest.json` 记录原图尺寸、来源、用途。
- **GitHub 是 source of truth，jsDelivr 只是缓存层**：将来换 CDN 只需换域名前缀，迁移成本几乎为零。
- **生产引用用 tag 不用 branch**：`@main` 是动态解析，`@2024.1` 版本化引用更稳、缓存策略更友好。
- **tool 返回结构化结果**（url、size、hash），Agent 侧校验和重试都好做。

## 总结

这套方案的本质是"把 GitHub 当免费对象存储，把 jsDelivr 当免费 CDN"。适合低频写、大量读、非敏感的图片场景，对个人博客和 Agent 自动化产出尤其顺手。如果你需要高频写入、私有图片、或对国内可用性有硬性 SLA，请直接上对象存储，别硬拗。对自动化场景而言，它最大的价值是**全链路 API 化**——五十行脚本就能让 Agent 获得稳定的"发图"能力。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/39c6567d577cd6af.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/6c0bf0bc866b294e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/680d72f2cac035d4.png)

