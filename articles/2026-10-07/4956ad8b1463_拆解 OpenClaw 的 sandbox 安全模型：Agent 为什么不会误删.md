---
title: 拆解 OpenClaw 的 sandbox 安全模型：Agent 为什么不会误删你的文件
feedId: 40842
source: 综合讨论
publishedAt: 2026-10-07
---

## 背景

跑过 Agent 的人多半见过这种时刻：模型在一条命令里拼错路径，或把 `$HOME` 当成了工作目录，`rm -rf` 一出去，后果不可逆。OpenClaw 的设计前提很直白：**Agent 的每一次文件写入和命令执行，都应被视为不可信输入**。sandbox 模型就是围绕这个假设搭的。

## 问题

风险不在于模型"想"删文件，而在于三层现实：

1. **路径解析错误**：相对路径、符号链接、`..` 拼接，模型给出的目标路径和它以为的可能不一致；
2. **能力过宽**：如果 shell 工具默认触达整个文件系统，一次幻觉就是一次事故；
3. **缺乏兜底**：没有审批、没有快照、没有审计，出错后既拦不住，也回不去。

## 做法：四层防御

OpenClaw 的 sandbox 不是一个开关，而是四层叠加，每层独立生效：

**第 1 层：文件系统作用域。** 所有文件工具（read / write / patch）执行前先把目标路径 resolve 成绝对真实路径（含符号链接展开），再校验是否落在 workspace 根内。workspace 外默认只读，写操作在工具层直接被拒。

**第 2 层：工具权限画像。** 每个会话绑定一个 permission profile，声明哪些工具可用、哪些参数模式触发拦截。内置的 destructive 检测会匹配 `rm`、`mv` 覆盖、`>` 截断、递归删除等特征，命中后转审批或强制 dry-run。

**第 3 层：执行隔离。** shell 工具跑在进程级隔离里（容器或 landlock/bwrap 同类机制），文件系统视图被裁剪到 workspace 加显式挂载点，网络默认关闭。就算命令本身失控，可见面也只有沙箱内那一小块。

**第 4 层：高危审批 + 快照。** 被标记为 destructive 的操作会挂起等待确认，确认前自动对涉及路径做快照，并写入审计日志，事后可回放"谁在何时动了哪个文件"。

配置上大致三步：收紧 workspace 边界 → 限制 shell 逃生舱 → 打开 confirm-on-destructive 和审计。示意如下：

```yaml
profile: coding-default
workspace:
  root: ./project
  readonly_mounts: [~/docs]
tools:
  shell: sandboxed            # 与 workspace 同边界
  confirm_on_destructive: true
audit:
  snapshot_before_write: true
```

## 踩坑点

- **符号链接逃逸**：只在字符串层面校验路径前缀没用，`ln -s` 一次就能绕过。必须 resolve 之后再校验，这是第 1 层的关键。
- **shell 就是逃生舱**：文件工具限得再死，只要 Agent 能跑任意命令，隔离就形同虚设。要么让 shell 继承同样的作用域，要么默认禁用。
- **白名单给太宽**：把 workspace 设成 `~` 等于没设。用最小目录加显式只读挂载。
- **MCP 第三方工具不走检查**：外部 MCP server 的写操作同样要纳入第 2 层权限画像，否则就是侧门。
- **没配快照等于裸奔**：sandbox 是概率性防御，快照才是确定性兜底。

## 可复用建议

1. 默认只读、按需开写；写权限跟着任务走，不跟着 Agent 走。
2. 任何删除/覆盖操作先看 diff 再执行，把它做成团队规范，而不是个人习惯。
3. 审计日志留下来，定期回看被拦截的 destructive 尝试，比任何评测集都真实。
4. 分层而非单点：任何一层配置失误，还有下一层接着。

## 总结

"Agent 不会误删文件"不是因为模型更聪明了，而是系统假设它一定会犯错，然后在路径、工具、进程、审批四层各设一道闸。OpenClaw sandbox 模型值得借鉴的正是这个思路：把安全做进执行环境，而不是寄希望于 prompt。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/a59d67da37275b13.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/9619e0d84927a675.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-07/9895727ef36f052b.png)

