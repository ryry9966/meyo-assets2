---
title: Agent 的 tools.md：管理本地配置和环境差异的正确姿势
feedId: 38804
source: 综合讨论
publishedAt: 2026-09-24
---

## 背景

跑 Agent 的人迟早会遇到同一类问题：同一套工具，在某台机器上好好的，换到台式机或容器里就报错。原因多半不是工具本身，而是我们告诉 Agent“怎么用工具”的那份文档——通常叫 tools.md——写得太随意：绝对路径、写死的版本号、只在某台机器上成立的假设。

tools.md 的本质是 Agent 的环境感知层。MCP server 清单、插件调用方式、本机路径、shell 差异，都靠它传递。它越接近“事实”，Agent 的动作越可靠；写得越像散文，重试和幻觉越多。

## 问题：三类典型故障

1. **路径硬编码**。文档里写 `/Users/zhang/venv/bin/python`，换台机器直接失效，Agent 开始各种补丁式绕路。
2. **文档漂移**。写的是 node@18，机器上是 node@20，Agent 按文档调参，报错后盲目重试。
3. **敏感信息入库**。为省事把 token、内网地址写进 tools.md，随仓库扩散。

共同根源只有一个：把“团队该知道的事”和“这台机器的事实”混在了一个文件里。

## 做法：共享层 + 本机层 + 探测段

建议把 tools.md 拆成三层：

**共享层（tools.md，提交进仓库）**：只写稳定约定——工具清单、调用模板、相对路径规则、禁忌事项。

**本机层（tools.local.md，加入 .gitignore）**：只写这台机器的事实——解释器位置、本地端口、代理设置。Agent 启动时先读共享层，再用本机层覆盖。

**探测段（放共享层末尾）**：列一组无副作用的命令，要求 Agent 每个会话先执行、再动手：

```bash
uname -s                # 系统类型
command -v python3 && python3 --version
echo "${OPENCLAW_HOME:-未设置}"
```

探测结果落在会话上下文里，后续所有调用以实测为准，而不是以文档的“记忆”为准。这一步把 tools.md 从静态声明变成可验证的契约。

写法上再加两条纪律：

- 调用模板用占位符（如 `{PYTHON}`），不写具体路径；
- 每个工具条目控制在三五行——Agent 对长文档的遵守率会明显下降。

## 踩坑点

- **写成散文**。大段“我们的项目用 Python……”Agent 抓不住重点，坚持用短句、列表、代码块。
- **探测命令有副作用**。探测段里放 `mkdir`、写文件类命令会污染工作区，探测必须只读。
- **本机层忘记 ignore**。tools.local.md 一旦误提交，泄露点就固化进了 git 历史。
- **Windows 差异想当然**。路径分隔符、PowerShell 与 bash 的引号行为都不同。共享层标注“命令以 POSIX 为准，Windows 由 Agent 自行适配”，比逐条兼容省力。
- **只写不验**。tools.md 也该进 code review：谁改了工具链，谁同步改文档；CI 里可以让 Agent 按文档跑一遍冒烟探测。

## 可复用建议

- 新项目第一周就把 tools.md 建起来，此时约定最少、最容易写准；
- 用 `tools.md` + `tools.local.md.example` 组合，模板入库、实例不入库；
- 文档里每个断言尽量对应一条可执行命令，“可验证的才写进去”；
- 定期让 Agent 自审：“以下哪些条目与你探测到的环境不符？”——它往往是第一个发现漂移的。

## 总结

tools.md 不是说明书，而是 Agent 与你机器之间的接口契约。管好它就一条原则：稳定的进共享层，多变的进本机层，一切以会话内探测为准。做到这三点，同一份 Agent 配置跨机器复用时，报错会从“玄学”降级成“可定位的差异”——这在工程上已经赢了大半。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/6550f0269fb88afe.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/ab2885c77f590432.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-24/e2e3b40b1c2ff076.png)

