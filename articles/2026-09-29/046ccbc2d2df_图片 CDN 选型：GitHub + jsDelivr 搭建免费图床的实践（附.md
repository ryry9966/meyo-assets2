---
title: 图片 CDN 选型：GitHub + jsDelivr 搭建免费图床的实践（附 Agent 自动上传方案）
feedId: 39494
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

做 Agent / MCP 自动化时有个高频需求：工作流产出一张图——渲染结果、截图、图表、报告配图——下游服务需要一个可公网访问的 URL（发卡片消息、写入文档、生成分享页）。本地路径没有意义，临时网盘链接又容易失效。

常见选型无非三类：

- **商业对象存储**（OSS/COS/S3）：稳定，但要实名、计费、签名，个人小流量场景偏重；
- **免费图床站**：链接存活不可控，随时可能跑路；
- **GitHub 仓库 + jsDelivr**：零成本、天然有版本管理、URL 结构稳定，适合个人规模。

第三条路最符合“可复现、可脚本化”的工程口味，下面是我的做法。

## 做法

**1. 建仓库**：新建一个公开仓库（如 `pics`），目录按 `images/YYYYMM/` 组织，控制单目录规模。

**2. 写上传脚本**（在仓库根目录运行）：

```bash
#!/usr/bin/env bash
# upload.sh <file>
set -e
f="$1"
name="$(date +%s)-$RANDOM.${f##*.}"
dest="images/$(date +%Y%m)/$name"
cp "$f" "$dest"
git pull --rebase -q || true
git add "$dest" && git commit -qm "add $name" && git push -q
echo "https://cdn.jsdelivr.net/gh/<user>/pics@main/$dest"
```

**3. 引用格式**：`https://cdn.jsdelivr.net/gh/<user>/<repo>@main/<path>`。需要不可变缓存时，把 `@main` 换成具体 commit hash。

**4. 接进 OpenClaw**：把脚本封装成一个 MCP 工具或 skill——输入本地文件路径，输出 CDN URL，并加约束：只接受白名单目录、仅图片类型、单文件 ≤ 5MB。Agent 产出图片后即可自主拿到外链，无需人工介入。

## 踩坑点

1. **缓存与更新**：jsDelivr 对同 URL 缓存非常激进，覆盖同名文件后旧图可能长期生效。解法是文件名带时间戳/哈希（脚本已做），必要时用官网 purge 工具手动清。
2. **国内可达性波动**：jsDelivr 在大陆的节点历史上反复不稳，上线前务必在目标用户的网络环境实测；关键页面准备备用源。
3. **体量上限**：GitHub 单文件 >50MB 告警、>100MB 拒收；jsDelivr 对 20MB 量级以上的文件也可能不服务。仓库别当垃圾场，否则 clone/push 会越来越慢。
4. **公开即公开**：拿到 URL 的人都能看，含敏感信息的截图、日志图不要进仓库；删除文件也不会让缓存立即失效，敏感内容从源头杜绝。
5. **并发冲突**：多个 Agent 实例同时 push 可能撞车。时间戳+随机数命名降低概率，push 前 `git pull --rebase` 兜底。
6. **别跑大流量**：个人和自动化场景没问题，但不要把整个站点的全量静态资源都压上去。

## 可复用建议

- 文件名规范 `{unix_ts}-{rand}.{ext}`：天然防缓存、防冲突、可追溯。
- 把“上传图片”收敛为一个**带约束的工具函数**交给 Agent，而不是让它自由执行 git 命令——白名单 + 类型/大小校验 + 统一返回 URL，出错面小得多。
- 长期页面（文档、主页）引用时固定到 commit hash，避免仓库变更导致挂图。
- 可选：给仓库加 CI 自动压图，控制体积膨胀速度。
- 双写降级：jsDelivr 为主，raw.githubusercontent.com 为备，前端可用 `onerror` 自动切换。

## 总结

GitHub + jsDelivr 是个人规模自动化场景下性价比最高的图床方案之一：零成本、可版本化、极易脚本化。但它是“够用”而非“企业级”——国内可达性、缓存策略、体量上限都要心里有数。在 OpenClaw 生态里，最顺滑的用法是把它封装成一个受约束的 MCP 工具，让 Agent 在需要外链图片时自己调，整条流水线就不需要人了。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/992977898df8c2f3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/adbe4b8738b0bad7.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/b44fc1895d6f900e.png)

