---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37793
source: 综合讨论
publishedAt: 2026-09-16
---

## 背景

OpenClaw 这类 agent 网关的典型运行形态是：模型拿到工具后直接在本机执行 shell、读写文件。能力越大，翻车半径越大。sandbox 安全模型要回答的核心问题只有一个：**当模型“想错了”，系统靠什么兜底？**

## 问题

误删很少来自模型“作恶”，更多来自三个经典场景：

1. **变量展开失败**：`rm -rf "$TARGET/"` 里 `TARGET` 为空，命令退化成对根路径的操作；
2. **相对路径歧义**：agent 以为 cwd 在项目目录，实际在用户 home，`rm -rf ./build` 删的是别处的 build；
3. **软链接逃逸**：workspace 里一个指向 `~/Documents` 的 symlink，让“工作区内删除”删到了工作区外。

## 做法：四层防线

OpenClaw 的思路不是“信任模型”，而是把删除动作拆成多层校验，任何一层拦住都算数。

**第一层：workspace 边界。** 每个 agent 绑定独立 workspace 根目录，文件工具和默认 exec 的 cwd 都规范化到根目录之下。路径解析在工具层做，不做字符串拼接；symlink 先 resolve 再校验是否越界。

**第二层：exec 沙箱。** shell 命令不直接跑在宿主进程里，而是落在受控运行时——macOS 上走 sandbox profile，Linux 上建议 Docker 容器，只挂载 workspace 和必要的只读目录。宿主 home、系统目录对容器内进程不可见，删无可删。

**第三层：命令策略。** 工具网关对命令做模式匹配，`rm -rf`、对 `.ssh`、`.git`、配置目录的写入等模式进入 deny 或“需审批”名单。关键是策略在网关层执行，不依赖模型自觉。

**第四层：审批门 + 副作用预演。** 高危操作要求人工确认，确认界面展示的是**解析后的绝对路径和受影响文件列表**，而不是原始命令字符串；删除默认先移入 trash 目录，延迟清理。

## 踩坑点

- 调试时把 sandbox 关了，忘开回来。建议生产 profile 写死默认值，并定期跑一条“越界删除”冒烟用例验证沙箱确实生效。
- Docker 图省事直接挂载整个 home，等于没有沙箱。挂载粒度压到 workspace 一级。
- 只靠 denylist 会被绕过（base64、`find -delete`、一行 python 脚本）。denylist 只做兜底，主防线必须是文件系统边界。
- symlink 必须在 open/unlink **之前** resolve，事后校验是典型 TOCTOU。

## 可复用建议

- 每个 agent 独立 workspace，宁可目录多一点，不共享根。
- 删除 = 移入 trash + 定时清理，成本极低，救回率极高。
- 审批界面展示绝对路径与文件清单，别让人盯着原始命令做判断。
- 把“沙箱是否生效”做成 CI 冒烟测试，而不是靠记忆。

## 总结

“Agent 不会误删文件”不是因为模型聪明，而是因为最坏情况下它**够不着**文件。边界在文件系统层、策略在网关层、人在审批层——三层任何一层独立成立，事故就不会发生。这也是我评估任何 agent 框架安全性的第一条：关掉模型的“自觉”，看它还能删什么。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/d88d7a3c437ffda1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/9d9362c61d4c13fa.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-16/559becd159ce9eb5.png)

