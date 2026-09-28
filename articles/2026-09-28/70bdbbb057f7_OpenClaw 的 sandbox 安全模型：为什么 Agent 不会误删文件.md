---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39306
source: 综合讨论
publishedAt: 2026-09-28
---

## 背景

OpenClaw 的核心循环是：模型 → 工具调用（exec、文件读写、浏览器）→ 结果回灌。其中 exec 最强大也最危险——模型拿到了 shell，理论上可以执行任何命令。用过 LLM 编码工具的人大多遇到过：相对路径当绝对路径用、`~` 展开错、`rm` 前少打一个空格。所以真正的问题不是“模型会不会犯错”，而是**犯错时的爆炸半径有多大**。

## 问题：安全靠什么保证

OpenClaw 的答案不是在提示词里写“别删文件”，而是分层防御。提示词约束是概率性的，文件系统隔离是确定性的。大致四层（字段名以你所用版本为准）：

1. **文件系统隔离**：exec 默认跑在 Docker 容器里，由 `agents.defaults.sandbox.mode` 控制（`off` / `non-main` / `all`）。Agent 只能看到 bind mount 进去的 workspace，宿主机其余路径对它根本不存在。
2. **挂载权限**：workspace 支持 ro/rw 控制。纯只读任务直接 ro 挂载，从物理上排除写入。
3. **权限收敛**：容器内以非 root 用户运行，默认无网络出口，`sudo` 提权不可用。
4. **工具策略与审批**：命中危险模式的操作（如对 workspace 外路径写入）会触发人工确认或直接拒绝，不依赖模型自觉。

## 动手验证（建议新人必做）

1. 确认 sandbox 生效：`docker ps` 应能看到 sandbox 容器，而不是宿主机进程。
2. 让 Agent 执行 `ls /`：容器视角的根目录是隔离的，和宿主机对不上。
3. 让它“删”一个目录（如 `rm -rf ./test-dir`）：变化只发生在 workspace 内，宿主机无痕。
4. 查审计日志：每条 exec 的命令、退出码、涉及路径都可回溯。

## 踩坑点

- **把 `~` 或 `/` 直接 rw bind 进容器**：隔离形同虚设，等价于 host exec。
- **沙箱镜像缺常用工具**：模型会主动请求“切到 host 执行”绕开沙箱。要么在镜像里装齐工具链，要么显式关闭 host fallback。
- **审批疲劳**：习惯性秒点“允许”后，审批层就退化成摆设。危险类别的确认不要设永久放行。
- **符号链接逃逸**：workspace 内软链指向宿主机真实路径时，删除会跟着走。挂载前检查软链，或降为 ro。
- **往容器里挂 docker.sock**：等于把宿主机 root 交出去，任何时候都别这么配。

## 可复用建议

- **最小挂载**：只挂项目目录，不挂 home。
- **写操作前先 git commit**：快照是最便宜的回滚手段，比任何事后补救都可靠。
- **团队共享配置时把 sandbox mode 定为 `all`**：host exec 只留排障场景，且用完即关。
- **定期做一次破坏性命令演练**：验证隔离没有被后续改动悄悄削弱，尤其是升级或改挂载之后。

## 总结

Agent 不会误删你的文件，不是因为它聪明，而是因为它的“手”被关在一个出不去的盒子里。确定性边界 + 审批 + 审计三层叠加后，模型犯错的最坏结果只是沙箱里多出一个空目录。安全配置的全部价值就在这里：**让最坏情况变得无关紧要**。这也是我建议每个 OpenClaw 用户先把 sandbox 验证一遍再谈自动化的原因。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/d28f50184721f2bc.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/b015432f7dbf679a.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-28/fd29376fdbc212bd.png)

