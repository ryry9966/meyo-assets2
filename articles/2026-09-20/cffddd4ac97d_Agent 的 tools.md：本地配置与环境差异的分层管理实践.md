---
title: Agent 的 tools.md：本地配置与环境差异的分层管理实践
feedId: 38259
source: 综合讨论
publishedAt: 2026-09-20
---

## 背景

把 OpenClaw 这类常驻本机的 Agent 跑起来之后，很快会撞上一个现实：模型对“你这台机器”一无所知。用什么 shell、装了哪些 CLI、MCP 服务怎么连、项目放在哪、有没有代理——这些环境事实不写清楚，Agent 就只能猜，猜错的代价是一次次失败的命令和被浪费的轮次。

workspace 下的 `tools.md` 会被注入系统提示词，正是官方留给环境知识的位置。但社区里常见的困惑是：该往里写什么、和 AGENTS.md 怎么分工、多台机器怎么维护。这篇帖子给出一套我验证过几个月的做法。

## 问题：环境知识散落的三种坏味道

1. **写错地方**：把本机路径、CLI 版本塞进 SOUL.md / AGENTS.md。换台机器，人格文件里全是过时环境描述。
2. **口头补丁**：会话里反复叮嘱“用 pnpm 别用 npm”“配置在 ~/.config/xx”。上下文一长就丢。
3. **放任探索**：每个任务先花几轮 `ls`、`which` 摸环境。偶发可以，常态既慢又贵。

## 做法：分层、模板、校验

**第一步，分层。** 只守一条边界：SOUL.md / AGENTS.md 管“你是谁、怎么做事”，tools.md 只放“这台机器是什么样”。凡随机器变化的进 tools.md，跨机器不变的别进。

**第二步，用固定骨架写**，控制信息密度：

```markdown
## 系统
- macOS 15 / Apple Silicon / zsh
## 常用 CLI
- node 22、pnpm 9、uv；无 docker
## 路径约定
- 项目根 ~/work；临时产物一律 /tmp/agent-scratch
## MCP 服务
- filesystem(~/work)、git、playwright
## 禁区
- 不碰 ~/.ssh；不做全局安装；不动生产目录
```

每节三五行即可。自检标准：一条新同事第一天上手这台机器需要知道的信息，才配写进来。

**第三步，多机管理。** 仓库只提交 `tools.md.example` 模板；各 workspace 的实际文件不进 git，或脱敏后进私有仓库。我用十几行的 bootstrap 脚本探测 OS、包管理器、已装 CLI，生成初版再人工修订，笔记本、家用服务器、VPS 各自一份。

**第四步，定期校验。** 写个 drift 检查脚本：对文档里每条 CLI 跑 `command -v`，对 MCP 端口做探活，失效条目当场标红。环境文档最大的敌人不是缺失，是过期。

## 踩坑点

- **别写密钥**。tools.md 会进系统提示词，也大概率进日志。只写变量名（如 `API_KEY` 来自环境变量），绝不写值。
- **别和 AGENTS.md 抢话**。同一件事两处描述且互相矛盾，Agent 行为会变得随机。
- **控制长度**。它按 token 计费并稀释注意力，建议给自己设 60 行硬预算，超了先删。
- **改环境当天同步**。新增/下线 MCP 服务立刻更新 tools.md，否则 Agent 拿着过期工具清单硬调，报错非常迷惑。
- **别整文件拷贝**。macOS 和 Linux 服务器差异很大，拷贝不如从模板重新生成。

## 可复用建议

- 把 tools.md 当代码：纳入 review，环境变更和文档修改走同一个 PR。
- 内容自检一句话：“换一台同款机器，这句话还成立吗？”成立才留。
- 每月跑一次 drift 脚本，顺带删掉当月没被命中过的条目。

## 总结

tools.md 的价值不在长，而在准。它本质上是给 Agent 的一份“机器交接文档”：环境事实与人格分离、骨架模板化、多机差异化生成、定期防漂移。做到这四点，环境差异就从“每次踩坑”变成“一次配置”，任何一台新机器上部署 Agent 的成本都能降到分钟级。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/a95fe808fa11e9ed.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/fc9a73b20881f203.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-20/cd296e1cc2da778f.png)

