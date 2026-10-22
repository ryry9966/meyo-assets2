---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 42281
source: 综合讨论
publishedAt: 2026-10-22
---

## 背景

OpenClaw 的 Agent 默认带 shell 和文件工具，能执行命令、批量改文件。能力即风险，"它会不会把我的仓库删了" 是每个接手自动化的人都会问的问题。OpenClaw 的设计前提很明确：**不假设模型够聪明，假设它一定会犯错**，然后用系统层兜底，而不是在 prompt 里写"请小心"。

## 问题在哪

误删通常不是模型"想删"，而是几种工程事故：

- 路径解析出错，相对路径拼错一层；
- 通配符过宽，`*.log` 变成 `*`；
- 对目录结构的理解有偏差，清理对象判断错误；
- 一条看似无害的清理命令，在错误的上下文里执行。

这些靠 LLM 的自觉是拦不住的，因为输出是概率性的。所以约束必须落在执行层。

## 做法：四层防线

**1. 容器边界。** 开启 sandbox 后，工具执行在 Docker 容器内，只挂载 workspace。就算命令里写出 `rm -rf /`，删的也只是容器内视图，爆炸半径被物理隔离。

**2. 路径沙箱。** 所有文件类工具统一走路径校验：先把目标 resolve 成绝对路径（含 symlink 展开），再判断是否落在 workspace 内，越界直接拒绝。校验在执行前完成，不信任 Agent 声称的工作目录。

**3. 危险命令拦截。** exec 类工具对命令做结构化解析后匹配 denylist：`rm -rf`、`mkfs`、`dd`、`git push --force` 等；同时维护一条不变量——**工作区外的任何写操作一律要求审批**，无论命令长什么样。

**4. 审批与回滚。** 被标记为 destructive 的操作进入人工确认队列；workspace 本身是 git 仓库，批量改动前 Agent 会被要求先 commit，事后可 revert。`.git` 目录本身在保护清单里。

配置大致长这样：

```yaml
sandbox: docker
workspace: /home/me/openclaw-workspace
tools:
  exec:
    deny: ["rm -rf", "mkfs*", "dd if=*"]
    require_approval: ["rm", "git push --force"]
fs:
  allowed_roots: ["${workspace}"]
  protected: ["${workspace}/.git"]
  resolve_symlinks: true
```

## 踩坑点

1. **挂载过宽**：图省事把 `$HOME` 挂进容器，沙箱形同虚设。只挂 workspace，宁可让 Agent 拿不到某些文件。
2. **只匹配首命令**：`bash -c "sh -c 'rm ...'"`、`xargs` 都能绕过浅层匹配。denylist 要覆盖解析后的完整命令树。
3. **先 cd 再操作**：Agent 先 `cd ..` 再执行，路径校验必须基于执行前的绝对路径判定，不信任运行时工作目录。
4. **symlink 逃逸**：工作区里一条指向 `/etc` 的软链就能骗过字符串匹配，必须 resolve 之后再判。
5. **语义绕过**：denylist 拦住了 `rm`，Agent 改用 `mv` 到 `/tmp` 后"清理"。所以关键是"工作区外写操作一律审批"这条不变量，而不是关键词本身。
6. **忘了保护 `.git`**：回滚依赖 git，Agent 一手删掉 `.git` 就没有退路了。

## 可复用建议

- 把安全模型写成**不变量**而非规则列表。"所有写操作的解析后绝对路径必须落在 workspace 内"这一条，比二十条关键词可靠得多。
- 分级授权：读默认放开、写限 workspace、destructive 必须人工，三档分开配置，别用一把总开关。
- 批量操作前强制快照（commit 或 tar），成本极低，出事时是唯一退路。
- 定期红队演练：写一条测试任务故意诱导 Agent 执行危险命令，验证拦截链路是否真的生效，别等真出事故才发现配置失效。

## 总结

OpenClaw 防"误删"靠的不是模型自觉，而是**分层默认拒绝**：容器限制爆炸半径，路径沙箱限制活动范围，denylist 加审批限制高危动作，快照保证可回滚。prompt 里写"请小心"没有工程价值，把"小心"变成系统不变量，才是 Agent 自动化能放心交给它跑的前提。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/7f8682fadc2a4eb1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/535ede4feb280b3c.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-22/47f2a6ba2275ef90.png)

