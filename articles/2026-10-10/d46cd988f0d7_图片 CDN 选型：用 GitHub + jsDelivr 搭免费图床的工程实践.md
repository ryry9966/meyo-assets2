---
title: 图片 CDN 选型：用 GitHub + jsDelivr 搭免费图床的工程实践
feedId: 41073
source: 综合讨论
publishedAt: 2026-10-10
---

## 背景

做 Agent 内容自动化时，有个绕不开的环节：Agent 生成插图或截图后，最终产出的 Markdown 里必须是一个可公网访问的图片直链。无论是让 OpenClaw 插件自动发周报，还是 MCP 工具输出图文报告，“生成图片 → 拿到直链 → 插入正文”这条链路都需要一个稳定又免费的图床。

## 问题

常见方案各有硬伤：免费图床（sm.ms 之类）有防盗链和清理策略，链接寿命不可控；对象存储要实名 + 备案 + 绑域名，对个人自动化场景太重；GitHub 仓库的 `raw.githubusercontent.com` 直链在国内又慢又不稳，还有速率限制。jsDelivr 对 GitHub 仓库内容提供 CDN 加速，两者组合几乎是门槛最低的方案——但代价是**大陆可用性波动**和**激进的边缘缓存**，这两个问题必须在设计时正视，而不是装作不存在。

## 做法

1. **建一个纯图床仓库**：公开仓库如 `assets`，按年月分目录存放（`2025/06/xxx.png`），不混代码。
2. **创建细粒度 PAT**：只授予该仓库 Contents 的读写权限，不要用全权限 classic token。
3. **走 Contents API 上传**：`PUT /repos/{owner}/{repo}/contents/{path}`，body 传 base64 内容：

```bash
curl -X PUT -H "Authorization: Bearer $GH_PAT" \
  https://api.github.com/repos/USER/assets/contents/2025/06/20250612-a1b2c3d4.png \
  -d '{"message":"upload","content":"<base64>"}'
```

4. **拼直链**：`https://cdn.jsdelivr.net/gh/USER/assets@main/2025/06/xxx.png`。文件名永远不重复，所以 `@main` 足够，不必固定 commit hash。需要强刷时调 `https://purge.jsdelivr.net/gh/...`。
5. **封装成 MCP 工具**：输入本地路径或 base64，内部完成压缩、命名、上传，返回 CDN 直链 + raw 链接作为兜底。Agent 生成图片后直接调用，一次到位。

## 踩坑点

- **大陆可用性是头号风险**。jsDelivr 在国内的可用性周期性波动，务必在目标读者的网络环境实测，别以“我这边能打开”为准。敏感场景要有降级路径（双链接或自托管反代）。
- **缓存不认覆盖**。同名文件更新后 CDN 刷新极慢，所以“每次上传都用新文件名”是纪律，不是偏好。
- **单文件上限 20MB**：超过该大小 jsDelivr 不予分发；仓库整体也建议控制在 1GB 内。PNG 先过 pngquant，截图类普遍能压到 200KB 以内。
- **公开即公开**：任何带 token、内网地址、客户数据的截图都不能进这个仓库，自动化流程里最好加一道敏感信息检查再上传。
- **速率限制**：匿名 API 只有 60 次/小时，批量上传必须带 PAT；细粒度 scope 收窄到单仓，泄露了影响也可控。

## 可复用建议

- 文件名约定 `yyyymmdd-{内容sha1前8位}.png`：防碰撞、缓存友好，重复上传天然幂等。
- 把图仓当**唯一事实源**，CDN 只是缓存层。哪天 jsDelivr 不可用，换镜像或自建反代，原图和链接结构不受影响。
- 上传逻辑抽成通用 MCP 工具（如 `upload_image`），多个 Agent 复用，返回值里同时给 jsDelivr 直链和 raw 兜底链接，由下游决定用哪个。

## 总结

GitHub + jsDelivr 不是完美方案，但零成本、零实名、API 就是纯 HTTP 调用，对个人 Agent 场景是合理的默认选择。大陆可用性和缓存策略是它的工程约束而非缺陷——用“新文件名 + 压缩前置 + 仓库为事实源”这三条纪律去适配，这套组合可以稳定跑很久。核心心法一句话：**图仓是资产，CDN 是耗材，任何一层挂掉都能随时换。**

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/751c7f6f15a38a8f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/4e05ee21e6c69fa9.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-10/87a2df9a2b9e8853.png)

