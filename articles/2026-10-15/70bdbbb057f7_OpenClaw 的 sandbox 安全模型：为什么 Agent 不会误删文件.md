---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 41568
source: 综合讨论
publishedAt: 2026-10-15
---

## 背景

让 Agent 直接操作文件系统，社区里最常被问的一句话是："它会不会把我的目录删了？"这个问题问得对。Agent 的行为本质是概率性的，模型再强，也会在路径解析、通配符展开、递归深度上出错。OpenClaw 的设计假设就是：**误操作是常态，防线必须建在模型之外**。

## 问题拆解

复盘各类"Agent 误删"事故，基本逃不出四种模式：相对路径解析错、工作目录漂移后删错位置；通配符展开范围超预期；递归删除叠加在错误的根节点上；走 shell 通道执行删除，绕过了文件工具的检查。这四条没有一条能靠"模型更聪明"解决，只能靠边界。

## OpenClaw 的分层做法

sandbox 不是一个开关，而是四层叠加，每层独立生效：

- **第一层：工作区隔离。** 所有文件工具绑定在一个 workspace root 内，路径 resolve 之后一旦越界（包括 `../` 穿越和符号链接逃逸），工具调用直接拒绝，不进模型上下文。
- **第二层：路径策略。** workspace 内再配 allow/deny 规则，`.git`、`node_modules`、密钥文件这类目录默认进保护名单。
- **第三层：破坏性操作拦截。** 删除、覆盖、批量重命名走确认门：先产出操作预览（动了哪些文件、多大体积），确认令牌到位才真正落盘；也可以把删除降级为"移入 `.trash/`"，物理上留后悔药。
- **第四层：进程级隔离。** Agent 进程跑在容器/低权限用户下，仅 workspace 挂载为可写，其余 read-only。前三层全被绕过时，操作系统兜底。

伪配置示意（字段名以你手上的版本为准）：

```yaml
sandbox:
  workspace: ./agent-ws
  writable: [./agent-ws]
  deny: ["**/.git/**", "**/*.pem"]
  destructive_ops: confirm   # preview | confirm | trash
  shell:
    allow_write: false
```

## 验证步骤

配置完别只看文档，做一次删除演练：

1. 在 workspace 里放一个诱饵文件和几个普通文件；
2. 明确指示 Agent"删除本目录全部内容"；
3. 观察是否先吐出预览、拒绝令牌后是否真的没删；
4. 构造一个指向 workspace 外的 symlink，验证 resolve 后被拦截；
5. 让它用 shell 执行 `rm`，确认命令级策略生效。

五步全过，防线才算跑通。

## 踩坑点

- **symlink 逃逸**：只做字符串前缀匹配必挂，必须 resolve 真实路径后再判边界。
- **shell 是最大旁路**：文件工具管得再严，给 Agent 一个可写 shell 等于白配，要么禁写、要么单独配命令策略。
- **拿 home 目录当 workspace**：见过有人图方便直接挂 `~`，根太大，保护名单救不了。
- **全开 auto-approve**：跑长任务图省事关掉确认门，等于只剩两层防线，至少保留删除预览。
- **多 Agent 共享 workspace**：A 的清理任务扫到 B 的产物，是最常见的"合规误删"。

## 可复用建议

- 最小可写原则：只挂载任务必需的目录，宁窄勿宽。
- 新任务先 read-only 试运行一轮，观察它想动哪些路径，再决定放权范围。
- workspace 用 git 管理，任何误操作都有 `checkout` 兜底。
- 删除一律降级为移入 `.trash/`，定期人工清理。
- 把五步演练做成回归脚本，每次升级 sandbox 配置都跑一遍。

## 总结

"Agent 不会误删文件"的准确说法是：**在正确的 sandbox 配置下，误删无法落盘**。模型负责判断，边界负责兜底，两者别混为一谈。OpenClaw 的价值不在于让模型更聪明，而在于把"出错"从灾难降级成一条日志。建议每个跑自动化的同事亲手做一次删除演练——五分钟的成本，换一个睡得着的夜晚。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/dd124c0711107df9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/f97134766288b43a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/d29fbe33ece27311.png)

