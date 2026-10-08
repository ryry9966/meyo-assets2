---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 40955
source: 综合讨论
publishedAt: 2026-10-09
---

## 背景

把 Agent 接上 shell 和文件工具之后，最怕的不是它写不出代码，而是它某次“顺手”把删除命令用错了地方。社区里流传过几个差点出事的案例：模型把训练数据里的常见路径当成真实路径、通配符误伤、把 workspace 写成了用户主目录。OpenClaw 的 sandbox 安全模型就是围绕这类真实风险设计的。这篇帖子梳理它的分层防御，以及我们自己踩过的坑。

## 问题：删除为什么格外危险

LLM 生成命令是概率行为，出错的形态很有规律：

1. **路径幻觉**：模型对“应该存在”的目录有先验偏好，可能编造或错拼路径。
2. **通配符盲区**：`*` 的展开发生在 shell 层，模型发出命令时看不到最终命中的文件列表。
3. **级联后果**：一次误删可能同时波及未提交的 git 工作、配置文件和缓存，恢复成本远高于单文件。

所以安全设计不能指望“提示词里叮嘱它小心”，必须落在系统结构上。

## 做法：四层防御

- **第一层：workspace 隔离**。文件工具默认以 workspace 为根。网关在每次调用前对路径做 canonical 化（realpath），再校验是否落在 workspace 前缀内。软链接会被解析成真实路径，防止“软链逃逸”。
- **第二层：exec 沙箱**。shell 命令默认跑在容器或受限用户下，文件系统按需挂载，workspace 之外默认不可写。
- **第三层：写操作确认门控**。删除、移动、覆盖类操作走网关的确认策略，可按粒度配置为自动放行（限 workspace 内）、询问或直接拒绝。
- **第四层：快照回滚**。workspace 内写操作前打快照，出问题能退回。

配置思路（示意，字段名以你所用版本文档为准）：

```json
{
  "sandbox": { "enabled": true, "mode": "container" },
  "workspace": { "root": "~/openclaw-workspace", "snapshot": true },
  "tools": { "delete": "confirm", "move": "confirm" }
}
```

落地步骤：开启 sandbox → 收紧 workspace root 到具体目录 → 给每个 MCP server 单独声明可访问路径 → 删除类工具设为 confirm → 打快照后**实际演练一次回滚**。

## 踩坑点

1. **软链逃逸**：我们在 workspace 里留过一个指向 home 的软链，是 realpath 校验拦下来的。教训：路径校验必须基于 canonical path，只比对字符串前缀等于没防。
2. **通配符展开时机**：拦截层看到的是 shell 展开后的 argv，deny 规则要按最终参数写，写在“命令原文”上会漏。
3. **图方便挂载过大范围**：有人把整个 home 挂进沙箱“省事”，隔离直接形同虚设。
4. **门控被自己关掉**：确认策略设成自动放行、超时又默认放行，等于没有门控。
5. **MCP 权限过宽**：每个 MCP server 拿到的目录范围要单独声明，最小权限得落到 server 级别，不是全局一份。

## 可复用建议

- 默认只读，按需逐项开写权限；白名单永远优于黑名单。
- 所有路径判断走 canonical path，插件和 MCP 工具统一从网关过校验，不要各自实现。
- 恢复流程要定期演练，只验证“备份存在”不算验证。
- 保留审计日志，出事时能还原每一步决策链。

## 总结

Agent 不会误删文件，不是因为模型突然变聪明了，而是因为一次删除要穿过四层相互独立的防线：隔离、沙箱、门控、快照。任何单层失效，还有下一层兜底。这套“不信任单点、把信任成本摊进系统结构”的思路，迁移到任何 Agent 框架和自动化流水线上都成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/5e29c576deb3a9a1.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/62a11371e47239ae.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-09/e5d7b50c8b8af78c.png)

