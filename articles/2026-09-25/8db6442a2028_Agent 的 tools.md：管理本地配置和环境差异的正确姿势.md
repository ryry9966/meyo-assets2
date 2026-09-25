---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 38936
source: 综合讨论
publishedAt: 2026-09-25
---

# 背景

用 Agent（OpenClaw 这类）跑本地任务时，最影响成功率的往往不是模型能力，而是它对“这台机器”一无所知：包管理器是 brew 还是 apt、Python 该叫 `python` 还是 `python3`、代理端口是多少、哪些目录能写。这些信息如果每次靠对话临时补充，换台机器、换个会话就全部归零。

# 问题

实践中反复出现三类事故：

1. **猜错命令**：在 Ubuntu 上调 brew，在 macOS 上跑 apt；
2. **环境漂移**：同一条 SOP 在 A 机器跑得通，B 机器报路径不存在；
3. **信息散落**：一部分写在系统提示词里，一部分在聊天记录里，一部分只在某个同事脑子里。

根治思路很朴素：给 Agent 一份它启动时必然读到的、描述本机工具链的“说明书”，也就是 `tools.md`。

# 做法

建议分层维护：`tools.base.md`（团队通用约定）+ `tools.local.md`（本机差异，加入 `.gitignore`）。

一个够用的最小模板：

```markdown
# tools.md
## 环境
- OS: Ubuntu 22.04 / shell: bash
- 包管理: apt（不要用 brew）

## 工具链
- node: 先 `source ~/.nvm/nvm.sh`
- python: 只用 python3，无裸 python

## 网络
- 代理: http://127.0.0.1:7890，git/npm 已配好

## 禁止
- 不要全局 pip install
- 不要动 /etc 下配置
```

接入方式：把它放进 Agent 的启动发现路径（工作区根目录或提示词模板里 include），确保每次会话自动注入，而不是指望用户记得“先发一下环境说明”。

# 踩坑点

1. **写成散文**。tools.md 是给机器读的事实清单，不是博客。每条一行，这段内容会占用上下文。
2. **写死绝对路径**。`/usr/local/bin/python3.11` 在下次升级后就变成错误信息。更稳的写法是记录“验证命令”（如 `command -v python3`），让 Agent 自己解析。
3. **和 MCP 配置重复**。MCP server 列表这类 harness 已经管理的东西不要抄进来，两处维护必然漂移。tools.md 只写 harness 看不到的本机事实。
4. **塞敏感信息**。token、密码一律走环境变量，文件里只写“密钥在 env XXX 中”。
5. **过期比缺失更糟**。升级工具链后忘改文档，Agent 会拿着旧事实自信犯错。建议文件头加更新日期，或写个十几行的 `tools-check.sh`，一键核对文档与实际版本。

# 可复用建议

- **当代码对待**：进版本库、走 PR，改环境必须连带改文档；
- **条目带自检命令**：Agent 先跑再动手，失败即暴露漂移；
- **只写差异**：base 写约定，local 写本机事实，避免每个文件抄一遍全量；
- **做成 onboarding 流程**：新机器初始化时，让 Agent 读 tools.md、跑一遍自检命令、把结果回填 local 文件。

# 总结

tools.md 的价值不在文档本身，而在于把“环境知识”从对话和记忆里挪进版本化的文件，让 Agent 行为可预测、环境差异可审计。它的成本极低——一个 markdown 文件加十几行约定——但能把“在我机器上能跑”这类问题挡掉大半。建议从最小模板起步，跑两周后按真实报错补充条目，比一开始追求完美结构更实际。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/c075aca67b62d1dc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/eeb6cc74813e6124.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-25/c51f6ee13d9dcdfa.png)

