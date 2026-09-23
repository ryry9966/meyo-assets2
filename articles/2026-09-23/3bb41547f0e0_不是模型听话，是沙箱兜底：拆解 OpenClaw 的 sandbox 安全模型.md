---
title: 不是模型听话，是沙箱兜底：拆解 OpenClaw 的 sandbox 安全模型
feedId: 38578
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

OpenClaw 的 agent 默认带 exec 和文件工具，能跑 shell、读写路径、装依赖。能力越大，社区群里被问得越朴素的一个问题就越常出现：“我让它批量重命名文件，它会不会顺手把我的 home 目录清了？”

这篇文章不讲玄学，只讲 OpenClaw 靠什么机制，把“误删”从概率事故变成几乎不可能发生的架构事故。

## 问题：误删的真实来源

误删不是模型“想删”，通常来自三种情况：

1. **幻觉执行**：模型理解偏差，把 `mv` 写成 `rm`，或路径展开出错；
2. **提示注入**：agent 读到的网页、issue、文件内容里夹带指令，被当成用户意图执行；
3. **权限过宽**：agent 进程直接跑在宿主机用户态，能碰到什么取决于操作系统，而不是你的配置。

第三条是根因。只要 agent 和你的终端同权，前两条就只是时间问题。

## 做法：分层设防

OpenClaw 的思路是纵深防御，每层独立成立，不指望模型自觉（具体键名以你当前版本的配置文档为准）。

**1. 容器沙箱。** 设置 `sandbox.mode: "all"`，exec 和文件操作全部落到 Docker 容器内执行。容器里的 `rm -rf /` 删的是容器层，宿主机无感。

**2. 最小挂载。** 只把 workspace bind mount 进容器。home、SSH 密钥、工作区之外的数据默认不可见——看不见的文件删不掉。

**3. 工具策略。** `tools.deny` 直接禁用高风险工具；exec 这类能力走 approval 流程，危险命令需要人工确认。记住：提示词里写“请不要删文件”不是安全边界，deny list 才是。

**4. 非 root + 会话隔离。** gateway 不以 root 运行；`sandbox.scope` 控制容器按 agent 还是按 session 隔离，避免一个会话的污染扩散到其他会话。

验证方式很直接：造一个假文件，让 agent 执行 `rm -rf ~/some-test-file`，然后回宿主机确认。我实际跑过一次，容器内“删除成功”，宿主机文件完好。

## 踩坑点

- **默认模式≠全沙箱**：sandbox 默认往往是 `non-main`，即主会话直连宿主机、只有非主会话被隔离。如果你的主会话一直“裸奔”，先查这一项。
- **挂载范围失控**：图省事把整个 home 挂进去，沙箱形同虚设。挂载列表要像 review 代码一样 review。
- **Docker socket 泄漏**：为了让 agent 自建容器，把 `/var/run/docker.sock` 挂进沙箱，等于交出宿主机 root。真需要就套一层受控代理。
- **网络限制反噬**：容器断网后 pip/npm 装不了包，排查半天发现是沙箱网络策略，配镜像源或代理即可。
- **混淆“约束”和“边界”**：system prompt 只影响概率，sandbox 影响结果。两者都要有，但别拿前者替代后者。

## 可复用建议

- 新 agent 接入先跑一遍“越权演练”：让它尝试读、写、删沙箱外路径，确认全部失败再上线；
- 挂载最小化，按需临时加，用完回收；
- 高危工具默认 deny，按 agent 逐个放行；
- 审计日志常开，事后能回答“它到底执行了什么”。

## 总结

“Agent 不会误删文件”不是因为模型聪明，而是因为架构保证：它够不着不该碰的文件。提示词管概率，沙箱管后果。把 sandbox 配置当成和数据库权限同等级别的工程工作来对待，才能放心把自动化真正交给 agent。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/25b802c93d3f9b7c.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/1371a0bbd24a689f.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/f58931fa7626ca2d.png)

