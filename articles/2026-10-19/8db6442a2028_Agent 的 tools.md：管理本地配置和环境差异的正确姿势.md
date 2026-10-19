---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 42020
source: 综合讨论
publishedAt: 2026-10-19
---

## 背景

OpenClaw 的 agent 每次会话都在一个"陌生环境"里醒来：它知道怎么调工具，但不知道你的机器上有什么。macOS 还是 Linux、brew 还是 apt、Python 走 uv 还是系统自带、哪台机器要挂代理才能出网——这些模型一概不知。workspace 里的 `tools.md` 就是为这个存在的：一份每个会话注入上下文的**本机事实清单**。

## 问题

不维护这份清单的典型症状：

- agent 反复探测环境，`which python`、`brew list` 一路试错，白白烧掉几轮对话；
- 拿通用假设硬套，比如在没装 Docker 的机器上直接甩 `docker compose up`；
- 多台机器（笔记本、家用服务器、VPS）行为不一致，同一个 prompt 在 A 机能跑、B 机报错；
- 你口头纠正过一次，下个会话它又忘了。

## 做法

1. **定位与粒度**：`tools.md` 放在 workspace 根目录，每台机器一份，不要跨机器同步——它描述的就是"这一台"。
2. **写什么**：四段式结构，直接看骨架：

```markdown
# tools.md（这台机器：MacBook Pro, macOS 15）
## 环境
- 包管理用 brew；apt 只存在于家里那台 Linux 服务器
- Python 一律 `uv run python`，不要裸调 python
## 常用命令
- 跑测试：`cd ~/code/xxx && uv run pytest -q`
- 本地服务状态：`systemctl status ollama`
## 禁区
- `~/backup` 只读；rsync 类命令先 `--dry-run`
- pip 安装需走代理
## 已知坑
- 系统 python3.9 缺 ssl，别用
```

3. **控制篇幅**：`tools.md` 每个会话都进上下文，要按 prompt 写，不是按文档写。我压在一页以内，能一行说清的不写三行。
4. **动态事实写成查询指令，不写成值**：别写"GPU 现在空闲"，写"GPU 状态用 `nvidia-smi` 查"。
5. **生成基线再手改**：先用脚本探测环境输出草稿（OS、包管理器、关键工具路径），人工删改后落盘，避免漏项，也避免把临时状态写进去。
6. **验证**：改完直接问 agent："读一下 tools.md，复述你了解到的环境和约束"，确认理解到位；再跑一个真实任务，对照会话日志看它是否真的在用你写的命令。

## 踩坑点

- **秘钥写进去**。`tools.md` 是明文，还可能被备份同步。只写"凭据在环境变量 `XXX` 里"，绝不写值。
- **和 AGENTS.md 抢职责**。规则（"提交前先跑测试"）放 AGENTS.md，事实（"这台机器用 brew"）放 tools.md，两边重复必然漂移。
- **信息过期**。agent 对这份清单信任度很高，卸载了工具没更新，它会对着报错反复重试。可以放進 heartbeat 定期自检，或环境大改后人工过一遍。
- **写成散文**。大段描述模型抓不住重点，条目化、命令化最有效。

## 可复用建议

- 四段式模板：环境 / 常用命令 / 禁区 / 已知坑，新机器十分钟填完；
- 把 `tools.md` 纳入 dotfiles 仓库、按机器分分支，`diff` 一下就能看到环境差异；
- 写个 20 行的 doctor 脚本，逐条检查清单里的命令是否仍存在、版本是否变化，输出漂移报告；
- 团队场景：各自维护自己的 `tools.md`，仓库只共享项目级约定，机器级事实不上传。

## 总结

`tools.md` 的本质，是把环境的隐式知识变成注入上下文的显式契约。它不是文档工程，是 prompt 工程：短、准、可验证。多机用户尤其值得投入一次——省掉的是每次会话里 agent 的瞎猜，和你反复纠正的耐心。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/24b39a18c54d944a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/b35ce70c7f54c1bd.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-19/5ba0a10b067542ed.png)

