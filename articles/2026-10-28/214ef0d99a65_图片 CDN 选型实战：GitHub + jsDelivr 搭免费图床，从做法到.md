---
title: 图片 CDN 选型实战：GitHub + jsDelivr 搭免费图床，从做法到踩坑
feedId: 42842
source: 综合讨论
publishedAt: 2026-10-28
---

## 背景

写博客、README 配图、Agent 产出带图报告、MCP 工具返回截图——这些场景都需要一个"贴进 Markdown 就能长期访问"的图片外链。自己搭服务太重，买对象存储嫌烦，免费的公共图床又总在跑路。这篇记录我用 GitHub 仓库 + jsDelivr 做图床的完整实践，以及在 OpenClaw 自动化流水线里的接入方式。

## 问题与选型对比

- **对象存储（OSS/COS/R2）**：稳定，但要管鉴权、计费，国内还要过备案
- **GitHub raw**：`raw.githubusercontent.com` 有速率限制，热链场景容易限流
- **公共图床服务**：免费额度不透明，随时可能关停

jsDelivr 本质是给 GitHub 仓库加了一层全球 CDN 缓存，适合"低频写入、高频读取"的图片场景，零成本且 URL 规范统一。

## 做法

1. 建一个**独立公开仓库**（如 `image-host`），只放图，和代码仓库隔离
2. 目录按年月组织：`/images/2025/xxx.png`
3. 文件名用**时间戳或内容 hash**，永不覆盖
4. 上传走 git push 或 GitHub Contents API
5. 引用格式：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@main/images/2025/a1b2.png
```

6. 要求链接长期稳定就 pin 到 tag 或 commit hash：`@v1.0.0` / `@<40位hash>`

脚本化上传示例（curl + Contents API）：

```bash
base64 -w0 img.png > /tmp/b64
curl -X PUT -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/USER/image-host/contents/images/2025/$(date +%s).png \
  -d "{\"message\":\"add\",\"content\":\"$(cat /tmp/b64)\"}"
```

从返回 JSON 取 `download_url`，把域名替换为 `cdn.jsdelivr.net/gh/...` 即得 CDN 地址。在 OpenClaw 侧可以把它封装成一个 MCP 工具：输入本地图片路径，输出 CDN URL，Agent 写文时直接引用。

## 踩坑点

1. **国内访问不稳定**：jsDelivr 的 ICP 2022 年就被吊销了，大陆节点时好时坏。面向国内读者的站点务必先实测，或主用 Cloudflare R2（10GB 免费额度），jsDelivr 当兜底
2. **缓存不刷新**：同名覆盖后 CDN 仍是旧图。要么文件名唯一，要么手动打 purge 接口 `https://purge.jsdelivr.net/gh/user/repo@main/path.png`
3. **单文件 ≤ 20MB**，且别当网盘用：仓库过大有被限制风险，jsDelivr 清理过滥用仓库
4. **未认证 API 限额 60 次/小时**，自动化必须带 token（5000/h）
5. **public 仓库全网可见**：敏感截图别传；上传前用 webp/pngquant 压一遍，体积能省一半以上

## 可复用建议

- 命名规范固定为 `{用途}/{年月}/{hash}.{ext}`，天然防覆盖、可追溯
- 写入低频、读取高频才用这套方案；写入频繁直接上 R2
- URL 拼接逻辑封装成函数或 MCP 工具，别在 Prompt 里手拼
- 工具描述里写清楚约束：只上传可公开的图、单张 ≤ 5MB、自动压缩

## 总结

GitHub + jsDelivr 是零成本图床里性价比最高的组合，工程要点就三条：仓库隔离、文件名唯一、上传走 API。它不是生产级 CDN 的替代品，但对个人博客、文档配图、Agent 自动化产出足够可靠——前提是想清楚目标读者在哪、走什么网络。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/a70d2c209c00ac45.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/bcfb9b8e93f6ab02.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-28/41958312a7cda963.png)

