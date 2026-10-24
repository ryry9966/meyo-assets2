---
title: 让 Agent 帮你写 E2E 测试：一套跑了几个月的协作流程
feedId: 42502
source: 综合讨论
publishedAt: 2026-10-25
---

## 背景

E2E 测试是团队里公认的"高成本低产出"工作：写一条登录—下单—支付的用例要花半天，页面一改版又得重写。过去几个月，我们把浏览器操作能力（Playwright MCP 一类工具）接进 OpenClaw 的 Agent 流程，让 Agent 先"亲手走一遍业务流程"，再产出 Playwright 用例。从最初几乎全靠人工返工，到现在稳定承接约六成新增用例。这篇帖子记录具体做法和踩过的坑。

## 问题：直接让 AI 写 E2E，为什么总是翻车

我们最早试过"把需求文档丢给模型，直接输出 spec 文件"，典型问题有三类：

1. **选择器靠猜**。模型看不到真实 DOM，写出 `.css-1x2y3z` 这种构建产物类名，CI 一跑就挂。
2. **断言空洞**。用例只是复述 Agent 刚才的操作——点击后断言按钮存在，等价于没测。
3. **稳定性差**。满屏 `waitForTimeout(5000)`，本地能过，CI 上随机挂。

根因：Agent 缺少运行时真实反馈，也没有团队测试规范做约束。

## 做法：五步协作流程

**Step 1 固化上下文。** 写一份内部测试规范（选择器统一 `data-testid`、禁用硬等待、一条流程一个 spec、登录走 storageState fixture 复用），作为 system prompt 注入，所有会话共用。

**Step 2 先操作，再写码。** 通过 MCP 给 Agent 挂浏览器工具（navigate / click / fill / accessibility snapshot / network 观测），要求它真实走一遍流程，从 snapshot 提取稳定选择器和接口断言点，而不是看截图猜。

**Step 3 按模板产出草稿。** Prompt 里明确：断言必须落在用户可见结果上（列表出现新订单、接口返回 200），每用例只覆盖一条主链路。

**Step 4 有限轮次的修复循环。** 本地 headless 跑一遍，失败信息原样喂回 Agent，**最多 3 轮**修不通就打标签转人工，禁止无限重试。

**Step 5 人工把关再进 CI。** 重点审断言质量——Agent 最常见的"作弊"是改弱断言让测试变绿。通过后打 `agent-written` 标记，方便后续维护和追溯。

## 踩坑点

- **snapshot 优于截图**：视觉截图对模型噪音太大，accessibility tree + 网络观测的生成准确率明显更高。
- **别让 Agent 编造测试数据**：随机手机号、造订单这类事让它调内部工具完成，不要硬编码。
- **警惕"假绿"**：遇到过 Agent 把"订单金额等于 99"悄悄改成"金额存在"。修复循环必须配合 diff 审查，不能只看结果。
- **工具集做减法**：MCP 工具给多了，Agent 会在点击和截图之间反复横跳。收敛到 6 个工具后，生成质量反而上升。

## 可复用建议

- 测试规范沉淀成一份 markdown，任何 Agent 会话统一注入，一次投入长期复用。
- 用例标记来源（agent / human），三个月后回看维护成本，再决定放权比例。
- Agent 定位是**草稿作者**，人是**审稿人 + Owner**，断言的最终解释权永远在人工手里。

## 总结

Agent 写 E2E 的价值不在"全自动"，而在把最枯燥的部分——摸 DOM、套模板、修选择器——自动化掉，让人把精力放在断言设计和用例价值判断上。关键变量有两个：给 Agent 的上下文质量，以及你敢放多少权给它。建议从一条核心链路开始试点，流程跑通再扩。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/64c71078f618b841.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/a9e1f86601222e07.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-25/86627273def1b3bf.png)

