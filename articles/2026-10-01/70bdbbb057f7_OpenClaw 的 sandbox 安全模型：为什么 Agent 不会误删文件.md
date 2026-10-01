---
title: OpenClaw 的 sandbox 安全模型：为什么 Agent 不会误删文件
feedId: 39959
source: 综合讨论
publishedAt: 2026-10-01
---

## 背景

让 Agent 直接碰 shell，是自动化实践里最让人手心出汗的事。OpenClaw 中 Agent 通过工具调用拿到执行能力：读写文件、跑命令、接 MCP 工具。于是几乎所有人的第一个问题都是——它会不会一句 `rm -rf` 把我的项目删了？这篇帖拆一下 OpenClaw 的 sandbox 安全模型，讲清楚为什么默认配置下，Agent 删不掉你不想让它删的东西。

## 问题

误删通常不是"模型想搞破坏"，而是几类低级但高频的失误：

- **路径幻觉**：把 `./src` 和 `~/src` 搞混，或拼错目录名后"顺手清理"；
- **命令组合**：`find ... -exec rm {}`、管道、子 shell 里藏了破坏性操作；
- **间接执行**：写一个脚本文件再执行它，绕过命令层检查；
- **cwd 漂移**：在错误的工作目录里执行了"清理临时文件"。

只在 prompt 里写"请不要删除文件"是拦不住这些的，sandbox 必须在执行层兜底。

## OpenClaw 的做法：四层防御

sandbox 不是一道闸门，而是叠了四层，任何一层失效还有下一层：

1. **工作区边界（jail）**：文件操作默认限制在 workspace 根目录内。路径解析对 symlink 做 realpath 归一化，防止"在工作区里放一个指向外部的软链接再穿越出去"。
2. **写权限白名单**：workspace 内再分区——源码可写，依赖与配置默认只读，workspace 之外一律拒绝。白名单用显式 glob，不做隐式放行。
3. **破坏性操作拦截（argv 级）**：对 `rm`、`rmtree`、`dd`、`truncate` 等危险动词拦截，匹配的是解析后的 argv 与工具调用参数，不是对命令字符串做 grep——所以管道和子 shell 里的变形也能命中。命中后默认走软删除：移入沙箱内 trash 目录保留 N 天，而非真删。
4. **快照 + 审计**：高风险批量操作前对目标子树做写时复制快照；每次文件操作写入追加式审计日志，出问题可回放、可恢复。

配置大致长这样：

```yaml
sandbox:
  workspace: ./project
  write:
    - ./project/src/**
  deny:
    - ./project/.git/**
  trash:
    enabled: true
    retention: 7d
  confirm_threshold: 10   # 单次影响超过 10 个文件需人工确认
```

## 踩坑点

- **symlink 穿越**：早期版本只检查路径前缀，Agent 在 workspace 里 `ln -s /` 后就能删到外面。教训：边界判断必须基于 realpath，并默认禁止创建指向边界外的链接。
- **字符串匹配的误报漏报**：grep 命令串要么拦不住 `bash -c "..."`，要么把 `grep rm README` 也拦了。改成 argv 级解析后才稳定。
- **间接执行**：Agent 把删除逻辑写进 `cleanup.sh` 再跑。现在"执行沙箱内生成的脚本"本身按高风险处理，走同一拦截链。
- **glob 配宽了**：有人图省事把白名单写成 `**`，等于第三层裸奔。按最小必要给权限。
- **快照吃满磁盘**：大仓库 + 高频批量操作时很占空间，要配容量上限和自动清理。

## 可复用建议

1. 永远不依赖单层防御：边界、白名单、拦截、快照至少叠两层。
2. 破坏性操作默认"软删除 + 可恢复"，不可逆动作留给显式人工确认。
3. 把拦截规则固化成回归测试："删掉根目录下所有文件""清空主目录"这类对抗性 prompt 写成测试用例，每次改 sandbox 都跑一遍。
4. 审计日志和快照是排障的最后底牌，别为了省空间先砍它们。

## 总结

"Agent 不会误删文件"不是模型听话，而是执行层的确定性兜底：路径被关进 jail，写权限被白名单收窄，危险动词被 argv 级拦截并软删除，操作前有快照、事后有审计。模型负责把事做对，sandbox 负责"做错时损失可控"。这套分层思路不绑定 OpenClaw，迁移到任何 Agent + 工具执行的栈上都成立。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/86f0515e5f18d20f.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/a5581c2a762dd105.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-01/563d3cea8ed9d060.png)

