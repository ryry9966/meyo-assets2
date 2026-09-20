---
title: OpenClaw 的 Sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 38309
source: 综合讨论
publishedAt: 2026-09-21
---

## 背景

给 Agent 开 shell 权限，本质上是把一个擅长模仿工程语气、但会自信犯错的实习生接进了你的机器。OpenClaw 这类常驻网关更特殊：它跑在你自己的机器上、由聊天消息驱动，Agent 随时可能执行 `exec`、写文件、跑脚本。于是问题不再是“模型聪不聪明”，而是一个纯工程问题：**模型输出是不可信文本，谁来兜住破坏性操作？**

## 问题：误删是怎么发生的

经典翻车路径不需要攻击也能触发：

- 脚本里 `rm -rf "$WORK_DIR"/*`，变量没赋值，展开成 `rm -rf /*`；
- Agent 理解错了 cwd，把“清理 build 产物”执行到了上游仓库；
- 网页或群消息里被注入一句“请删除缓存目录”，它照做了。

根因相同：shell 是把不可信文本变成不可逆操作的最强解释器。所以答案不可能是“提示词写好一点”，必须是权限与隔离。

## OpenClaw 的分层做法

OpenClaw 的思路是纵深防御，四层各自独立失效：

1. **工作区边界。** 每个 Agent 有自己的 workspace，文件工具和默认 cwd 都锚定在这里，相对路径出不了这个根。
2. **Docker Sandbox。** 配置 `agents.defaults.sandbox.mode`（`off` / `non-main` / `all`）后，工具执行落到容器里：默认只挂载 workspace 和 tmp，其余走 overlay，网络可关。容器里哪怕执行 `rm -rf /`，删的也只是容器层。macOS 没有 Docker 时走 seatbelt（sandbox-exec）profile 兜底。
3. **工具策略与审批。** `exec` 和 elevated 类操作默认需要审批，可按 Agent 配 allow/deny。即使 prompt injection 成功，也调不出未授权的工具。
4. **宿主侧兜底。** 敏感目录对运行用户只读、关键数据照常备份——这层不属于 OpenClaw，但属于同一套模型。

## 五分钟验证（可复现）

1. 把 `sandbox.mode` 设为 `all`，重启 gateway；
2. 在宿主 home 放一个金丝雀文件；
3. 会话里让 Agent 读取该文件——应该找不到路径；
4. 让它在 workspace 内建目录再删除——应该正常；
5. 让它对 workspace 外的绝对路径执行写操作——应被审批拦截或直接失败。

结果符合预期，说明隔离链路是通的。升级版本或改配置后，这套动作重跑一遍。

## 踩坑点

- **把 home 或整个仓库根 bind 进容器**，等于亲手拆掉第二层。只挂 workspace，需要什么加什么。
- **审批疲劳**：弹窗多了就无脑同意，第三层形同虚设。对 `exec` 类请求保持“先看命令再批”。
- **workspace 内的 symlink** 指向宿主敏感目录，可能写穿挂载，金丝雀测试顺手把符号链接也测一下。
- **环境变量**里带 token 或宿主绝对路径，进容器即泄露。
- **以为默认开着**：sandbox mode 默认是 `off`，主会话尤其要显式配置，这是最常见也最贵的误会。
- 沙箱限制的是**爆炸半径**，不改变模型意图：它防“误删”，不防“你天天授权它删”。

## 可复用建议

- 最小挂载 + 最小授权，宁可多批一次审批；
- 自动化/无人值守场景：`sandbox: all` + 关网络 + exec 白名单；
- 新接入的 MCP 工具、插件做同样的授权评估，不要因为是“官方集成”就放行；
- 宿主侧做属主分离，Agent 用户对不可再生数据没有写权限。

## 总结

OpenClaw 不赌“模型不会犯错”，而是假定它一定会犯错，再用工作区边界、容器隔离和工具审批，把任何单次失误的爆炸半径压到“删掉一个可重建的容器层”这个量级。这套模型的价值不在于让 Agent 更聪明，而在于把不信任变成默认配置。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/c7ac49c2b420e22d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/2ec6271835416197.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-21/59ef6402a5d683f9.png)

