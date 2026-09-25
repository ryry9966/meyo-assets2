---
title: GitHub + jsDelivr 免费图床实践：给 Agent 流程补上图片这块拼图
feedId: 38984
source: 综合讨论
publishedAt: 2026-09-25
---

## 背景与问题

写技术博客、维护开源文档，或者让 Agent 自动生成带图内容时，图片放哪是个绕不开的问题。可选方案无非几类：云厂商 OSS + CDN（稳定但要钱，国内节点还涉及备案）、第三方免费图床（随时可能跑路）、自建（成本和维护都归自己）。我的需求很典型：个人文章和项目文档配图，量不大，希望免费、可控、最好能被脚本直接调用。最终选了 GitHub 仓库做存储、jsDelivr 做分发的组合，用了大半年，把实践和坑摊开讲。

## 原理与做法

jsDelivr 本质是对 GitHub / npm 等公开仓库的一层全球 CDN 缓存。文件放进公开仓库后，用固定格式就能拿到 CDN 链接：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<分支或tag>/<路径>
```

步骤：

1. **建独立仓库**：新建一个公开仓库专门放图（比如 `assets`），和代码仓库解耦，不污染提交历史。
2. **走 API 上传**：用 GitHub Contents API，不需要本地 clone，核心逻辑十几行：

```python
import base64, hashlib, requests

REPO, BRANCH = "yourname/assets", "main"

def upload(filename: str, data: bytes) -> str:
    name = f"{hashlib.sha1(data).hexdigest()[:12]}.{filename.rsplit('.', 1)[-1]}"
    path = f"images/{name}"
    r = requests.put(
        f"https://api.github.com/repos/{REPO}/contents/{path}",
        headers={"Authorization": "Bearer <token>"},
        json={"message": name,
              "content": base64.b64encode(data).decode(),
              "branch": BRANCH})
    r.raise_for_status()
    return f"https://cdn.jsdelivr.net/gh/{REPO}@{BRANCH}/{path}"
```

3. **接入自动化**：把函数包成一个 MCP tool（如 `upload_image`），Agent 写文档时直接调用，拿 URL 后自动插入 Markdown。也可以做成 CLI，挂进截图工具的 hook，实现截图即上传即得链接。

## 踩坑点

- **国内访问不稳定**。网上教程还在说“jsDelivr 国内加速”，那是老黄历，近几年国内节点时好时坏。把它当“免费、还行”的分发层，别当强 SLA 的生产 CDN。
- **缓存不实时**。push 后 CDN 不会立刻生效，分支引用官方口径缓存约 12 小时。急用可手动刷：请求 `https://purge.jsdelivr.net/gh/<user>/<repo>@<分支>/<路径>`。
- **仓库越养越肥**。每次上传都是一次 commit，历史里的图删了也还在。只传文章相关资源，按内容哈希命名（天然去重），别当网盘用——纯二进制囤积有违反 GitHub 条款的风险。
- **API 限流**。未认证每小时 60 次，挂 token 是 5000 次。Agent 批量处理记得加节流和重试。
- **文件名**。中文名、空格迟早出事，统一小写加连字符。

## 可复用建议

- **存储和分发分开看**：GitHub 是源，jsDelivr 是缓存层，原图务必在本地或私有备份留底。
- **最小权限 token**：fine-grained token 只授这个仓库的 Contents 读写即可。
- **不可变资源**：对发布后不应变化的图，引用 tag 或 commit hash 而非分支名，缓存行为更可控。
- **坏链监控**：定期用脚本对文中图片做 HEAD 探测，坏链早发现。
- **留好迁移路径**：量级上来或要面向国内生产环境时，把同一套上传逻辑指向 OSS API，URL 换前缀即可。

## 总结

GitHub + jsDelivr 不是新鲜方案，但对个人开发者来说，它把“图床”退化成几行代码的 API 调用，并且天然能嵌进 Agent / MCP / 自动化流水线。预期管理好：适合博客、文档、README 这类非强商业场景；速度别指望 SLA，可用性靠原图备份和坏链监控兜底。工具越简单，越值得花半小时把它包成顺手的小工具。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ca7e53f71b9d6cd5.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/ad2f75c6b914b7d5.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/58835dec71f0451e.png)

