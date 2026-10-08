---
title: 给 Agent 一份 tools.md：把本地环境差异变成可读的契约
feedId: 40929
source: 综合讨论
publishedAt: 2026-10-08
---

## 背景

跑 Agent 做自动化时，最大的不稳定因素往往不是模型，而是“这台机器”。同一份任务描述，在我的笔记本上能跑通，换台机器就卡在依赖安装；Agent 不知道你用 uv 还是 conda、哪个端口被占用、代理要不要开。AGENTS.md 这类记忆文件通常写的是项目约定，很少写“这台机器的现实”。

tools.md 就是补这一层的：一份专门给 Agent 读的本地环境说明，把机器差异从“试错发现”变成“开局声明”。

## 问题

没有 tools.md 时，常见的翻车方式：

- Agent 在 Ubuntu 上尝试 `brew install`；
- 绕过 venv 直接往全局 site-packages 装包；
- 拉依赖卡在默认源，而机器明明配了国内镜像；
- 每次会话都要重新口头解释一遍环境，token 和耐心双输；
- 更糟的：把密钥直接贴进对话上下文。

共性是：环境知识只存在于你脑子里，Agent 只能猜。

## 做法与步骤

1. **分层放置**。仓库根目录放 `tools.md`（提交进 git，写团队共识）；个人差异放 `tools.local.md`（进 .gitignore），并在 tools.md 里留一行指向它。
2. **固定小节，只写可执行的事实**：

```markdown
# tools.md
## 运行环境
- Ubuntu 22.04 / bash；项目一律在 ~/work 下操作
## 工具链
- Python 一律走 uv；验证: uv run python -V
- Node 由 nvm 管理；验证: node -v
## 网络与镜像
- 代理 http://127.0.0.1:7890；pip 走国内镜像源
## 本地服务
- Postgres 5432，docker compose up db；健康检查: pg_isready
## 禁区
- 禁止 sudo、禁止全局 pip install、不要动 ./legacy
```

3. **每条工具都带验证命令**。Agent 能自检，就不用你人肉确认。
4. **在 AGENTS.md 里显式引用**：“环境事实见 tools.md，冲突时以 tools.md 为准”。OpenClaw 里也可以把它挂进系统提示或项目记忆。
5. **写一个 `make doctor`**（或单脚本）逐条跑验证命令，会话开始前先体检一次。

## 踩坑点

- **写成散文**。“我平时用 conda”没用，`conda activate xxx` 才有用，Agent 需要无歧义的命令。
- **塞密钥**。tools.md 只写“密钥在哪、怎么加载”，不写值本身。
- **信息重复导致漂移**。README、AGENTS.md、tools.md 三处都写环境，两周后必然打架。单一事实源，其他地方只放链接。
- **只写“怎么装”不写“怎么用”**。Agent 应使用你已有的工具，而不是每次重装一遍。
- **越写越长**。tools.md 每个字都进上下文，建议控制在百行内，过期条目果断删。
- **忽略路径差异**。WSL 和 Windows 混用时不说明清楚，Agent 会在错误的文件系统里找半天。

## 可复用建议

- 把 tools.md 当代码：改 Dockerfile、CI 或依赖时，同一 PR 里同步更新。
- 文件头写“最后验证日期”，超过一个月没跑过 doctor 的条目视为可疑。
- 团队内让 tools.md 进 code review——它就是机器与 Agent 之间的接口契约，接口变更理应被评审。

## 总结

tools.md 的本质很小：把“这台机器怎么用”从口头记忆，变成一份 Agent 可读、可验证、可版本化的声明文件。几十行的投入，换来更少的试错回合、更少的误装依赖和更稳的自动化。环境差异消灭不了，但可以被声明。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/8ba2e6db3dad9fd3.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/4b7721af7d020f81.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-08/b6b46ef7f84aa3a3.png)

