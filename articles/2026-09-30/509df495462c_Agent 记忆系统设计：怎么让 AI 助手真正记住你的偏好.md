---
title: Agent 记忆系统设计：怎么让 AI 助手真正记住你的偏好
feedId: 39682
source: 综合讨论
publishedAt: 2026-09-30
---

## 背景

用 Agent 干活久了都会遇到同一个尴尬：每次新会话，助手又变回“陌生人”。你明明说过项目用 pnpm、提交信息别带 emoji、部署目标是家里的 NUC，下次它照样问一遍。上下文窗口再大，也只是“这顿吃饱”，不是“记住味道”。这篇记录我在 OpenClaw 里给助手加记忆层的实践，思路对任何基于 MCP 的 Agent 都适用。

## 问题：三种常见做法为什么不行

1. **全量塞 system prompt**：偏好攒到几十条后，token 成本上升，注意力被稀释，关键偏好反而更容易被忽略。
2. **只做会话摘要**：有损压缩，“用户偏好 pnpm”这种事实和“今天聊了部署”这种叙事混在一起，检索噪音大。
3. **一上来就上向量库**：偏好类记忆总量很小，向量召回的不确定性比精确读取更麻烦。

核心矛盾是：写入要宽松（宁可多记），读取要严格（只注入当下有用的几条）。

## 做法：一张表起步，四条规则

存储就是一个 SQLite 表，包成一个小 MCP server，暴露 `memory_write` / `memory_search` / `memory_forget` 三个工具：

```sql
CREATE TABLE memories (
  id INTEGER PRIMARY KEY,
  scope TEXT,        -- global / project / session
  kind TEXT,         -- preference / fact / constraint
  content TEXT,      -- 结构化后的单句事实
  source TEXT,       -- 来源会话
  confidence REAL,
  hit_count INTEGER DEFAULT 0,
  updated_at TEXT
);
```

规则比存储本身更重要：

1. **写入靠触发词 + 轮末抽取**。用户说“记住”“以后都”“别再”时立即写；每轮结束做一次轻量抽取，只保留“主语 + 偏好 + 范围”结构的单句。
2. **更新而非追加**。新旧记忆在相同 scope + kind + 主语下冲突时覆盖旧条，`updated_at` 刷新，旧值挪进 history 字段。没有冲突处理的记忆系统，两周就会精神分裂。
3. **读取有预算**。按 scope 匹配 > hit_count > 时效排序，取前 5~8 条压成紧凑块注入。全局偏好常驻，项目偏好按工作目录匹配。
4. **命中打点 + 衰减**。被注入且实际使用后 hit_count +1；90 天未命中的降权，用户说“忘掉 XX”时直接删。

## 踩坑点

- **一次性指令被记成永久偏好**。“这次用 pnpm 装一下”被写成全局偏好，之后所有项目都被强推 pnpm。解法：写入前先判 scope，默认 project，只有明确说“以后/所有项目”才写 global。
- **记了但从不命中**。早期检索只做关键词匹配，大量记忆永远沉底。加上 hit_count 观察两周后，砍掉了一半长尾。
- **敏感信息混入**。API key、住址被当成“事实”存了下来。现在写入前过正则 + 模型双重过滤，并保留查看与删除入口——记忆系统没有“查看/删除”，就是隐私负债。
- **覆盖无法回滚**。加了 history 字段之后，才敢放心做自动覆盖。

## 可复用建议

- 先用单表 + 规则跑三个月，别急着上向量库，偏好记忆的规模撑不起检索复杂度。
- 每条记忆必带 scope、source、时间戳，缺一个，冲突处理就没法做。
- 把“遗忘”做成一等能力：显式 forget 命令 + 定期衰减，比“永远记住”更重要。
- 攒 10~20 条召回评测问答对，每次改注入策略跑一遍，别凭感觉调参。

## 总结

记忆系统的难点不在存储，而在写入的判断与读取的克制：写入时区分范围、处理冲突，读取时控制预算、追踪命中。先让 20 条高质量偏好稳定生效，比堆 2000 条向量更像“认识你”。下一步我打算把抽取和冲突合并挪到离线任务里做，schema 欢迎在评论区一起对齐。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/83f48b54303a559d.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/b94bc1365d616c34.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/1753cf3f95dd2be4.png)

