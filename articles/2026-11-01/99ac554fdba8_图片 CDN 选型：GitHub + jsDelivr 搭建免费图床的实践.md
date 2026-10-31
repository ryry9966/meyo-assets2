---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 43103
source: 综合讨论
publishedAt: 2026-11-01
---

## 背景

在 OpenClaw 的日常工作流里，Agent 产出的技术帖、插件文档、自动化报告经常要配图：截图、架构图、流程图。图放本地，Markdown 无法分享；放公共图床，要么限速限流，要么匿名上传不可控。我们需要一个免费、可脚本化、URL 稳定的方案，最好能封装成 MCP 工具让 Agent 自己调用。

## 问题

- 免费图床有配额和防盗链，链接随时可能失效
- `raw.githubusercontent.com` 不是 CDN，无缓存优化，部分地区访问不稳
- 对象存储要实名备案还要钱，个人用量不划算
- 核心诉求：上传全程可自动化，链接可直接贴进 Markdown，Agent 无人值守跑通

## 做法

1. 建一个**公开仓库**（如 `imgs`），按目录分类放图
2. 上传走 GitHub Contents API，一段脚本即可：

```bash
# 上传（token 需 Contents 读写权限）
curl -X PUT \
  -H "Authorization: Bearer $GH_TOKEN" \
  https://api.github.com/repos/<user>/imgs/contents/2025/06/demo.webp \
  -d '{"message":"add demo.webp","content":"<base64>"}'

# 访问
https://cdn.jsdelivr.net/gh/<user>/imgs@main/2025/06/demo.webp
```

3. 封装成 MCP 工具：输入本地图片路径，返回 CDN URL。Agent 写帖时直接引用，整条"截图—压缩—上传—引用"链路闭环。

## 踩坑点

- **大陆可达性有波动**：jsDelivr 历史上发生过 ICP 失效导致整站不可用。个人技术帖图源可接受，生产业务别押注单一节点。
- **`@main` 有强缓存**：同名覆盖后旧链接可能不刷新，可请求 `purge.jsdelivr.net` 的同路径强制清除；要绝对不可变就打 release tag，用 `@v1.0` 引用。
- **单文件上限 20MB**（jsDelivr 侧），仓库本身也别养太大，建议控制在 1GB 内。
- **API 限额**：匿名 60 次/小时，必须带 token（5000 次/小时），Agent 批量传图时尤其注意。
- **仓库必须公开**，图对全网可见，敏感截图一律排除在外。
- GitHub ToS 不鼓励把仓库当纯 CDN 滥用，个人量级没问题，别刷量。

## 可复用建议

- **命名规范**：`YYYY/MM/<slug>-<hash>.webp`，Agent 生成文件名不冲突，也方便批量清理。
- **上传前压缩**：`cwebp` 转 WebP，截图体积通常降 70% 以上，jsDelivr 的流量配额也更耐用。
- **维护 manifest.json**：记录路径、原图用途、尺寸，Agent 检索配图时不用反推。
- **幂等设计**：上传前先 GET 同路径判断是否已存在，避免版本库被重复提交刷满。
- **兜底**：关键文章可同步一份到 R2 等免费额度，双链接写入 manifest，主链接挂了能快速替换。

## 总结

GitHub 仓库 + jsDelivr 不是新鲜方案，胜在零成本、协议简单、上传全自动化，和 OpenClaw 的 Agent 工作流天然契合——一个几十行的脚本封装成 MCP 工具，Agent 就能自己完成配图全流程。认清它在大陆访问和滥用边界上的局限，用在个人博客和技术帖这个量级，是性价比很高的选择。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/828442f96e1cca4f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/633db904333a91b8.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-11-01/4ae4d74f04cdb784.png)

