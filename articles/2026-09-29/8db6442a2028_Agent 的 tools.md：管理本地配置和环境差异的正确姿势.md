---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 39420
source: 综合讨论
publishedAt: 2026-09-29
---

## 背景

Agent 跑起来之后，最大的不确定性往往不是模型能力，而是它对"这台机器"一无所知。同一份任务描述，在我的笔记本上能跑通，换到实验室的 Linux 服务器就开始瞎猜：该用 `uv` 的时候敲了 `pip`，该走 `make test` 的时候自己拼了一串 pytest 参数，路径、代理、CUDA 版本全靠幻觉。这些错误单看都很蠢，但每个会话都要重新纠正一遍。

## 问题

把环境信息写进系统提示词有三个毛病：一是和项目耦合，换个仓库就得改 prompt；二是多机不同步，同事 clone 下来跑不通；三是会话一长，这些约定在上下文里被稀释，Agent 会悄悄"忘掉"。

我们的做法是把这类信息沉淀成一个 `tools.md`，放在工作区里让 Agent 开工前先读。本质上，它是**面向 Agent 的环境契约**。

## 做法

分层是关键。我们用三层，优先级从高到低：

1. **项目级** `tools.md`：跟仓库走、进版本库，写项目特有约定（测试命令、目录结构、禁止操作）。
2. **机器级** `tools.local.md`：进 `.gitignore`，写本机事实（Python 路径、GPU、代理端口）。
3. **全局级** `~/.config/agent/tools.md`：写个人习惯（默认编辑器、shell、常用别名）。

Agent 侧只需一条规则："执行任何命令前，先读 tools.md，冲突时以项目级为准。"

一个最小模板：

```markdown
# tools.md
## 运行
- 包管理：uv，禁止直接用 pip
- 测试：make test，不要自己拼 pytest 参数
## 环境
- Python 3.11，位于 /usr/local/bin/python3.11
- 出网需 HTTPS_PROXY=http://127.0.0.1:7890
## 禁止
- 不要动 /data 下的任何文件
- 不要执行 rm -rf 和 sudo
```

写法上有两条经验：每行只陈述一个事实；多用"用 X / 禁止 Y"的祈使句，Agent 对指令式描述的遵循度明显高于散文。

## 踩坑点

- **别放密钥**。tools.md 会被读进上下文，可能随日志、trace 外泄。密钥放环境变量，tools.md 只写"从哪读"。
- **别写成文档**。我们第一版写了 200 行背景说明，Agent 抓不住重点。现在控制在 40 行内，全是事实和指令。
- **别和现有脚本重复**。Makefile 里已有的任务，tools.md 只写"统一走 make"，否则两处漂移。
- **过期信息比没有更糟**。Python 升级后忘了改，Agent 会拿着旧路径反复失败。解法是加一个十来行的 `tools_check.sh`，在 CI 里断言文件中声明的命令真实存在。
- **别假设 Agent 会主动读**。必须在 agent 规则或系统提示里显式挂上这条指令，否则它根本不知道这个文件存在。

## 可复用建议

- 把 tools.md 当代码 review：环境变更的 PR 必须同步改它，地位等同 Dockerfile。
- 提供一个 env snapshot 脚本，输出当前机器的关键事实，新机器初始化时先生成草稿再人工修剪。
- 团队若混用多种 Agent 框架，tools.md 保持框架无关，"谁负责读它"放进各自的启动配置。

## 总结

tools.md 的价值不在"文档"，而在把 Agent 从"每次猜一遍"变成"开局读一遍"。它成本极低——一个 markdown 文件加一条读取规则——却消除了大部分环境类翻车。我们的粗略统计是，接入后环境相关的返工从每任务两三次降到接近零。建议从一个 20 行的最小版本起步，让踩过的坑逐条沉淀进去，它会慢慢长成团队最划算的 Agent 配置资产。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/ab3c5d03408da12e.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/2302ed93d30f558a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-29/7f7c162f9042d4e8.png)

