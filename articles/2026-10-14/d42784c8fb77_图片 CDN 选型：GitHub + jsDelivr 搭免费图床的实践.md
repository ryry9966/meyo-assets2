---
title: 图片 CDN 选型：GitHub + jsDelivr 搭免费图床的实践
feedId: 41515
source: 综合讨论
publishedAt: 2026-10-14
---

## 背景

做自动化和 Agent 实践时经常遇到一个朴素需求：流程跑完要产出"可外链的结果"。OpenClaw 驱动的 Agent 截了张运行截图，MCP 工具生成了张架构图，想贴进 issue、文档或博客，第一反应就是"这图放哪"。商业图床收费且限流，对象存储要绑域名、要计费，自建 Nginx 又多一台机器要养。

## 问题

我的需求排序是：免费、URL 稳定、能被脚本调用、有 CDN 缓存、不引入运维负担。GitHub 仓库天然满足"免费 + 版本化"，但 raw.githubusercontent.com 直链在国内可用性差且有限速。jsDelivr 补上了关键一环：它把任意公开 GitHub 仓库代理成全球 CDN，URL 规则固定：

```
https://cdn.jsdelivr.net/gh/<user>/<repo>@<branch>/<path>
```

于是图床问题退化成"往仓库写文件"。

## 做法

1. 建一个公共仓库（私有仓库 jsDelivr 不代理），比如叫 `assets`。
2. 上传用 GitHub Contents API，一次 PUT 就是一个 commit：

```bash
BASE="https://api.github.com/repos/$USER/assets/contents"
HASH=$(sha1sum "$f" | cut -c1-8)
NAME="$(date +%Y%m%d)/$HASH-$(basename "$f")"
curl -s -X PUT "$BASE/$NAME" \
  -H "Authorization: Bearer $PAT" \
  -d "{\"message\":\"upload\",\"content\":\"$(base64 -w0 "$f")\"}"
echo "https://cdn.jsdelivr.net/gh/$USER/assets@main/$NAME"
```

3. 把这段封装成 CLI 或 MCP 工具：输入本地路径，输出 CDN URL。Agent 和插件链路里直接调用，同 hash 已存在则跳过（幂等）。

## 踩坑点

- **缓存不刷新**：同名覆盖更新后，jsDelivr 边缘节点可能长期返回旧图。最省事的解法是文件名带内容 hash，每次更新即新 URL；确需覆盖时用 purge.jsdelivr.net 主动清缓存，但有频控。
- **单文件上限 20MB**，超限直接失败；仓库过大时 jsDelivr 同步变慢，别把图床当网盘用。
- **二进制进 git 历史几乎不可逆**，体积只增不减。按日期分目录、定期归档旧图到新仓库，保持主仓精瘦。
- **高频自动化 commit 可能触发 GitHub 滥用检测**。批量图合并成一次 commit（Git Trees API 或本地 git push），PAT 用细粒度权限，只授予该仓库的 contents 写权限。
- **公共仓库 = 图片公开可枚举**。终端截图、内网拓扑这类图，先脱敏再传。
- **国内可用性**：jsDelivr 的国内节点近两年时好时坏，这是这套方案最大的不确定性。我的做法是保留 raw.githubusercontent.com 作为备选域名，按实际受众切换。

## 可复用建议

- 文件名规范固化下来：`{日期}/{短hash}-{语义名}.png`，缓存问题基本消失。
- 上传工具一定要做幂等和重试，Agent 链路里图片上传失败不应该中断主流程。
- URL 规则和业务解耦：上层只存相对路径，域名做成配置项。哪天量上来了迁去 R2 或对象存储，改一行配置就能切。
- 用 MCP 封装时把"返回 URL"和"返回本地路径"都暴露，调试期直接看本地文件更快。

## 总结

GitHub + jsDelivr 这套图床的上限很明确：20MB、公开性、国内波动。但在这些边界内，它几乎是零成本方案里最顺手的一个——git 管版本、CDN 管分发、一个几十行的脚本管上传，和 OpenClaw 的自动化链路天然契合。我的判断标准很简单：个人和小团队、中等量级，用它；到量了或者对国内可用性有硬要求，平滑迁走，不硬撑。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/1ca153b4437e1546.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/fec17faf75ac249e.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-14/f49c8b8e6f10d2dc.png)

