---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37433
source: 综合讨论
publishedAt: 2026-09-14
---

## 背景

Agent 能调用 exec 和文件系统工具，意味着模型输出的每一行 shell 都可能直接落到你的磁盘上。自动化场景下（定时任务、多步工作流、MCP 工具链），没有人在旁边盯着每一次 `rm`。我让 OpenClaw 跑了三个月的日常文件整理，期间它确实"想"删过几次不该删的东西——但一次都没删成。这篇帖子拆一下它的 sandbox 模型是怎么做到的。

## 问题：误删不是低概率事件

误删的来源很具体：

- **路径幻觉**：模型把 `~/projects` 和 `~/` 记混，相对路径基准错了；
- **生成脚本副作用**：agent 自己写的清理脚本逻辑有误，exec 一执行就删错范围；
- **提示注入**：被抓取的网页或文档里藏了指令，诱导 agent 执行"清理"命令。

单靠 prompt 约束（"请不要删除文件"）不可靠，因为约束和执行在同一层面，模型可以自己推翻自己。安全边界必须放在模型控制不到的层。

## OpenClaw 的分层做法

1. **工具层 policy**：每个工具调用前过一遍 allow / deny / ask 规则。fs 写入类工具被限制在 workspace 路径内，路径先做 realpath 规范化，软链接逃逸和 `../` 都会被拦下。
2. **执行层 gate**：exec 类工具单独设策略，高危命令模式（`rm -rf`、`mkfs`、`dd`、重定向覆盖）默认 deny 或 ask，普通命令放行。
3. **OS 层隔离**：sandbox 模式下整个 agent 跑在 Docker 容器里，非 root 用户，只把 workspace 目录以 bind mount 挂进去，其余文件系统根本不可见。这一层是关键——就算前两层全部被绕过，容器里也没有可删的东西。
4. **快照兜底**：破坏性操作执行前自动打 git snapshot，或把删除重定向到 trash 目录，恢复成本为零。

## 实操步骤

以 Docker sandbox 为例：

1. 配置中启用 sandbox，指定 workspace 挂载路径；
2. tool policy 里把 exec 设为 ask，fs.write 白名单限定在 workspace 内；
3. 为高危命令 pattern 加 deny 规则，而不是依赖确认弹窗；
4. 以非 root 容器用户运行，确认挂载目录属主一致；
5. 做一次对抗测试：明确指示 agent"删除 workspace 外的某个文件"，观察它被拒绝并在审计日志中留痕。没拦住就说明配置有洞。

## 踩坑点

- **只限 fs 工具、不管 exec**：agent 会用 shell 命令绕过文件工具的限制，这是最常见的配置漏洞；
- **workspace 里的软链接指向外部目录**：不做 realpath 检查的话，路径白名单形同虚设；
- **ask 疲劳**：什么都弹确认，用户会习惯性一路 yes，等于没设。宁可对高危 pattern 直接 deny，把 ask 留给真正需要人判断的场景；
- **容器内 root 写挂载目录**：宿主机上文件属主变成 root，后续工具链读写出问题；
- **子进程继承权限**：exec 起的脚本再删文件，工具层拦不住，必须在容器层兜底。

## 可复用建议

- 默认 deny，逐项 allow，坚持最小权限；
- 双层防御缺一不可：工具层 policy 管粒度，OS 层容器管下限；
- 一切破坏性操作先 snapshot；
- 把"对抗测试"做成回归用例，每次升级 OpenClaw 或更换模型后跑一遍；
- 定期翻审计日志，被 ask 拦下的记录往往能暴露策略配置的盲区。

## 总结

"Agent 不会误删文件"不是因为模型聪明，而是因为最坏情况的破坏半径被架构限制住了：路径白名单挡住越界，命令 gate 挡住高危操作，容器挡住一切漏网的。把信任边界放在模型之外，是 OpenClaw sandbox 模型最值得借鉴的一点。安全配置不是一次性的，升级后跑一遍对抗测试，比读十篇 best practice 都管用。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/a06b75e8a3fd365a.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/c90ed5d74d5ff77d.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-14/77bc59977fb67125.png)

