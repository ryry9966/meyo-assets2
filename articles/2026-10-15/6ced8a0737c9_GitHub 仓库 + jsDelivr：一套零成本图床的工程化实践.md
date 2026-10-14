---
title: GitHub 仓库 + jsDelivr：一套零成本图床的工程化实践
feedId: 41570
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

社区里写博客、维护文档、跑 Agent 工作流的朋友，迟早都会撞上同一个问题：截图和小图往哪放。对象存储要花钱还可能要备案，第三方图床随时跑路。GitHub 仓库本身可以当存储，但直接引用 `raw.githubusercontent.com` 访问不稳定，也没有缓存。jsDelivr 提供了 `cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>` 这种 URL，把公开仓库变成一个带全球节点缓存的静态资源源，等于免费拿到一层 CDN。

## 问题

我的需求很具体：本地截图或 Agent 生成的图片，一条命令（或一次工具调用）拿到稳定公网 URL，直接贴进 Markdown 或对话。约束：零成本、可长期维护、图片本身仍有 git 版本管理。

## 做法

1. 建一个**公开**仓库，比如 `assets`，按 `images/2025/06/` 分目录。私有仓库不行，jsDelivr 只服务公开内容。
2. 文件名用内容哈希（如 `sha1 前 10 位.webp`），天然去重且不可变，从根上避开缓存更新问题。
3. 推送后引用：`https://cdn.jsdelivr.net/gh/me/assets@main/images/2025/06/abc123.png`。要求更强一致性的场景，把 `@main` 换成 commit SHA，内容永不漂移。
4. 自动化：写了个几十行的 MCP 工具（CLI 脚本同理）：输入本地图片 → sharp 压缩转 WebP → 算哈希命名 → GitHub API 提交 → 返回 CDN URL。挂到 OpenClaw 里之后，会话中说一句“这张图传图床”，Agent 调工具直接回链接。
5. 确实要覆盖同名文件时，去 `purge.jsdelivr.net` 提交路径强制刷新。但我的原则是宁可换名，不做覆盖。

## 踩坑点

- **国内可达性是最大变量**：主域近年波动明显。建议同时测试 `fastly.jsdelivr.net`、`gcore.jsdelivr.net` 等备用域名，模板里保留 raw 地址兜底。这个方案只适合博客、文档、个人项目这类低关键性场景，别当生产业务的主 CDN。
- **不支持 LFS**：推了 LFS 的图只会拿到指针文件；单文件也别超 20MB，超了直接不服务。
- **缓存很凶**：同名覆盖后旧图可能长时间不更新，哈希命名能根治，别依赖 purge。
- **仓库别当垃圾桶**：GitHub 对仓库体积有软性约束，无压缩原图胡乱堆，迟早难受。上传前压缩这步不能省。
- **Token 最小权限**：自动化用的 PAT 只授予这一个仓库的 Contents 读写，别拿全权限 token 到处塞。

## 可复用建议

- 把“本地文件 → 公网 URL”抽象成一个函数或工具，jsDelivr 域名不要硬编码散落各处。以后想换 R2 或对象存储，只改一层。
- 心智模型别反了：**仓库是 source of truth，CDN 只是它的一层视图**。
- 如果你已有 Agent 工作流，做成 MCP tool 的收益比裸脚本高：任何会话里都能自然语言调用，还自带调用记录，方便回溯“这张图是哪次上传的”。

## 总结

GitHub + jsDelivr 是典型的“够用就好”方案：零成本、版本化、对 Agent 自动化友好。代价是可达性看运气、缓存策略要绕着走。只要定位清楚——它是个人内容的分发层，而不是基础设施——它就是个很舒服的小工具。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/0a0f912f88ef3ca6.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/1b64af2fc6317b4b.png)

