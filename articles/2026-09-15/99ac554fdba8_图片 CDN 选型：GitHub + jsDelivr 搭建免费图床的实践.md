---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践
feedId: 37629
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

做 Agent 项目时经常遇到同一个需求：跑图、截图、流程图、实验输出，需要一个能直接贴进博客、README 或对话里的稳定 URL。团队项目用对象存储当然规范，但个人项目为几十张图去开桶、绑域名、算流量，成本收益不成比例。

## 问题

常见免费方案的痛点：

- 各类"免费图床"网站：随时跑路，链接失效没有兜底手段；
- `raw.githubusercontent.com`：没有 CDN 缓存，部分地区访问慢，且历史上 content-type 配置问题导致部分客户端无法直接渲染；
- 对象存储 + CDN：稳定，但要维护域名、备案、账单。

把 GitHub 仓库当"源站"，jsDelivr 当缓存层，是一个务实的折中。

## 做法

1. **建一个公开仓库**，比如 `assets`。必须是 public，私有仓库 jsDelivr 不回源。
2. **上传图片**。手动就 git push；自动化场景用 Contents API，一次 PUT 完成：

```bash
curl -X PUT \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  https://api.github.com/repos/me/assets/contents/2025/06/demo.png \
  -d "{\"message\":\"add demo\",\"content\":\"$(base64 -w0 demo.png)\"}"
```

3. **拼 CDN 地址**，格式固定：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<version>/<path>
```

`@main` 跟分支走，`@<commit-sha>` 或 `@<tag>` 锁版本。写博客建议锁 commit，文档引用用 tag。

4. **接入 OpenClaw 工作流**：把第 2 步包成一个 MCP 工具（输入本地图片路径，返回 CDN URL），Agent 生成截图后顺手上传，URL 直接吐回对话。脚本 30 行以内。

## 踩坑点

- **中国大陆访问不稳定**。jsDelivr 的 ICP 备案被吊销后，大陆侧解析经常到 Fastly，时通时断。面向国内读者的站，它只能当备选，不能当唯一。
- **缓存极强，覆盖同名文件不生效**。想更新就换文件名（推荐 `日期-语义名-短hash.png`），或主动刷：`https://purge.jsdelivr.net/gh/user/repo@main/img.png`。
- **单文件上限 20MB**，超了 jsDelivr 直接不回源。上传前压缩转 webp 是基本素养。
- **文件名别用中文和空格**，URL 编码问题会浪费你半小时。
- **别当网盘用**。存大文件批量分发既违反 GitHub 服务条款，也容易把仓库玩废。

## 可复用建议

- 仓库按 `yyyy/mm/` 建目录，文件名带日期和短 hash，天然去重；
- GitHub 仓库是唯一事实源，jsDelivr 只是缓存层。哪天它彻底不可用，raw 地址、自建反代都能兜底；
- 若面向国内且在意稳定性，可平移到 Cloudflare R2（免费额度 10GB 存储）或对象存储。目录结构和工具链不动，只换 URL 拼装层——这也是把"上传"和"拼 URL"拆成两个函数的意义；
- 给 MCP 工具加降级：上传成功返回 CDN URL，失败时退回 raw 地址。

## 总结

GitHub + jsDelivr 不是最稳的图床，但可能是个人开发者成本最低、迁移代价最小的方案：源站在自己手里，CDN 只是可选加速层。它的合理定位是"文档、博客、Agent 产出的中小图片分发"，而不是高可用图片服务。认清这个边界，它就够用；想清楚迁移路径，它就不怕失效。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/bebe065cbef5a4d0.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/3dee6e7c8e74c378.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/88b26efb42939925.png)

