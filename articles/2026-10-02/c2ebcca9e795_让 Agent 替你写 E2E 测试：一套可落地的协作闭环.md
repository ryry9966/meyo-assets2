---
title: 让 Agent 替你写 E2E 测试：一套可落地的协作闭环
feedId: 40087
source: 综合讨论
publishedAt: 2026-10-02
---

## 背景

E2E 测试是很多团队“知道重要、但一直在欠债”的部分。Playwright、Cypress 上手都不难，难的是持续维护：页面改版、选择器失效、测试数据漂移，写用例的时间永远被业务需求挤占，最后核心链路只剩一条没人敢动的冒烟。

Agent 时代这件事有了新解法。但方向不是“一键生成测试”——而是把 AI 当成一个跑在闭环里的初级工程师：给它浏览器、给它报错日志、给它明确的验收规范，让它写、跑、改，人只做评审。

## 我们踩过的坑：直接生成基本不可用

最早的尝试是把需求描述直接丢给模型，让它输出 Playwright 用例。结果三类问题反复出现：

- 大量 `expect(page.isVisible())`，看起来覆盖了，实际什么业务结果都没验证；
- 选择器靠猜，本地能跑，CI 上 flaky；
- 不了解环境的前置数据，用例之间互相污染。

结论：**缺上下文 + 缺反馈闭环的生成，产出基本不可用**。

## 做法：四步闭环

**1. 喂上下文，而不是喂需求。** 通过 MCP 的 browser 工具让 agent 自己打开页面、抓取 DOM、枚举路由；同时注入一份手写的测试规范：选择器必须优先 `data-testid`、断言基线是什么、登录态怎么注入。规范放仓库里一个静态文件，每次自动注入，比在 prompt 里反复交代稳定得多。

**2. 先骨架，后断言。** 拆成两步。骨架阶段只要求 agent 描述操作序列（goto → fill → click → 期望跳转），人工确认路径没走偏之后，再让它补断言。断言必须指向业务结果（列表行数变化、接口返回字段），不允许只断言元素存在。

**3. 隔离环境跑，失败日志回灌。** 每条用例生成后立刻丢进 CI 容器执行，把 Playwright trace 的截图和网络快照喂回去让它自我修正，最多 3 轮，仍失败就标记人工介入。

**4. 人审合并。** 所有 PR 打 `ai-generated` 标签，重点看断言强度和清理逻辑（afterEach 是否还原数据）。

一份可复用的 agent 配置骨架：

```yaml
tools: [browser_navigate, browser_snapshot, browser_click, run_test, read_trace]
guardrails:
  max_repair_rounds: 3
  assert_policy: business-outcome-only
  selector_policy: data-testid-first
```

## 踩坑点

- **测试幻觉**：agent 会编造不存在的接口字段名。强制要求断言里的字段必须从真实响应里复制，不允许凭记忆写。
- **Flaky 会传染**：AI 修 flaky 的第一反应是加 `waitForTimeout`。在规范里直接禁掉，只允许 `expect(...).toBeVisible()` 这类自动等待。
- **上下文过期**：页面改版后 agent 还按旧 DOM 写用例。每次生成前必须先 `browser_snapshot` 拿实时结构，禁止依赖会话里的旧快照。
- **成本失控**：3 轮修复上限 + 每轮记录 token 消耗。反复修不好的用例，往往说明需求本身没描述清楚，退回人工，不要硬烧。

## 可复用建议

- 生成器只负责写，验证器永远是真实浏览器 + 真实环境，两者职责不要合并；
- 度量别只看“生成了多少条”，看两周后还活着的用例比例，那才是真实收益；
- 先覆盖注册、登录、下单这类低频变更链路，高频改动的活动页留给人工；
- 把人审的固定检查项写成 PR 模板，评审速度会比“凭感觉看”快一倍。

## 总结

Agent 写 E2E 测试的价值，不在于替代人写，而在于把“写、跑、修”这个重复循环自动化掉，让人把精力放到断言设计和测试策略上。闭环、约束、人审，三者缺一不可：少了闭环，产出不可用；少了约束，产出不可维护；少了人审，产出不可信。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/e355b2daaa145567.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/049a94efb48281ca.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-02/025c5d5c0a375403.png)

