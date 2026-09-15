---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 37723
source: 综合讨论
publishedAt: 2026-09-15
---

## 背景

OpenClaw Agent 常态化持有三类高危工具：文件读写、shell exec、以及各类 MCP/插件暴露的副作用操作。于是每个新用户几乎都会问同一个问题：它哪天抽风，会不会把我家目录 `rm -rf` 了？

这个担心是合理的。模型输出本质上是非确定性的，删文件这种不可逆操作，不能指望模型"每次都想对"。

## 问题：prompt 约束不是安全边界

很多人第一反应是在系统提示里写"禁止删除文件"。这在工程上站不住脚：

- 上下文混乱、指令注入、工具参数拼接错误时，模型仍可能产出删除调用；
- 就算 file 工具锁死了，exec 工具里一句 `find . -name "*.log" -delete` 一样绕过去；
- prompt 是"劝阻"，不是"拦截"。安全要求的是后者。

正确的思路是：**让执行层"不能做"，而不是让模型"答应不做"**。OpenClaw 的 sandbox 模型就是围绕这个展开的。

## 做法：四层防御

OpenClaw 的安全模型大致分四层，逐层收窄能力：

**1. Workspace jail（文件层）**
file 工具的路径解析被约束在 workspace 根内。关键细节是：解析会先做 realpath 展开，再重新校验——否则 `../` 穿越或 symlink 逃逸就能出界。

**2. Exec 沙箱（执行层）**
shell 命令默认进容器/命名空间沙箱执行：根文件系统只读挂载、独立 scratch 目录可写、网络默认关闭。宿主机目录对沙箱不可见，`rm` 在里面删不到你的东西。

**3. 审批与 allowlist（策略层）**
沙箱内读操作自动放行；沙箱外的写/删操作走网关审批，推送到你的 IM 手动确认。高频安全操作可以配 allowlist，但要按工具粒度配，而不是一把梭的全局通配。

**4. 审计（可观测层）**
每次工具调用的参数和结果落审计日志。出了问题能回放，这也是排查"它到底执行了什么"的唯一可靠依据。

配置示意（字段以当前版本 openclaw.json 为准）：

```json
{
  "sandbox": { "mode": "workspace", "network": false },
  "tools": { "exec": { "approval": "always" } }
}
```

## 踩坑点

- **symlink 逃逸**：旧版本在调用时才解析符号链接，workspace 里一个指向家目录的软链就能出界。务必升级到做了 realpath 后置校验的版本。
- **exec 绕过文件策略**：只给 file 工具上锁、exec 却裸跑，等于没锁。shell 是万能逃生门，必须单独进沙箱。
- **用 root 跑 Agent**：jail 的路径校验拦不住 root capability，永远用低权限专用用户跑。
- **allowlist 写成 `~/**`**：和没有 allowlist 没区别。按具体项目目录给，宁窄勿宽。
- **只读挂载里带了敏感文件**：`.env`、SSH key 挂进沙箱就是交给模型读，挂载前先过一遍清单。

## 可复用建议

1. **默认拒绝**：任何新工具/插件接入，先 deny，再按需放行。
2. **红队演练**：每次改完配置，在一次性 workspace 里故意下达"清理所有旧文件"这类危险指令，看拦截链路是否按预期工作。
3. **审计日志异地存**：本机日志可能被 Agent 自己改，推到独立位置。
4. **最小挂载**：沙箱里只挂当前项目，用完即弃的目录优于常驻挂载。
5. **升级看 changelog**：sandbox 相关的修复（尤其路径校验）要当成安全补丁对待，不要攒着不更。

## 总结

OpenClaw 之所以不会误删文件，不是因为模型聪明，而是因为架构上让"误删"不可达：jail 限制路径、沙箱隔离执行、审批拦截高危动作、审计兜底追责。模型负责干活，安全边界交给执行层——这是所有 Agent 工程都该守住的分工。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/6886e0b94cf77c11.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/2ab718ccd2755bb1.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-15/e7672d5c2adc0c54.png)

