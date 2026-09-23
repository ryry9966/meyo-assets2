---
title: 让 Agent 帮你写 E2E 测试：关键不是生成，是闭环
feedId: 38633
source: 综合讨论
publishedAt: 2026-09-23
---

## 背景

E2E 测试的处境一直尴尬：价值明确，但维护成本高。手写慢，录制工具生成的代码能跑但脆，团队往往攒了一批用例后就没人敢动。

Agent 出现后，很多人第一反应是"让 AI 把测试写了"。实际试下来，直接把需求丢给 agent 一次性生成，效果通常不好：选择器靠猜、断言空洞、根本跑不起来。问题不在模型能力，而在流程——写 E2E 测试本质是一个"打开页面 → 观察 → 编写 → 执行 → 修复"的循环，一次性生成天然缺了中间几步。

这篇帖分享我们在 OpenClaw 环境下，用 agent + MCP 工具把 E2E 测试生产串成闭环的实践。

## 做法

### 1. 先固化基建，再谈生成

Agent 只能在约定内工作。生成之前先准备好：

- Playwright 脚手架：统一 fixtures、共享 auth state（登录态复用，不进用例）
- 选择器约定：优先 `data-testid`，禁用依赖文案和样式名的选择器
- 一份 `testing-rules.md` 作为 agent 的规则输入（类似 AGENTS.md 的用法）：每个用例至少一条业务断言、禁止 hard wait、禁止 `text=` 选择器等

### 2. 给 agent 配上"眼睛"和"手脚"

只靠读源码写测试不可靠。通过 MCP 给 agent 挂三样工具：

1. 浏览器控制（Playwright MCP 类 server）：真实打开页面、抓 DOM snapshot、验证选择器存在
2. 文件读写：生成和修改 spec 文件
3. 命令执行：跑 `npx playwright test`，读取失败输出

三者缺一，闭环就断了。

### 3. 任务拆分与生成节奏

不要一次生成整站。按路由/功能拆任务：

- 先让 agent 输出用例清单（用例名 + 验证点），人工确认优先级
- 每轮生成 2–3 个用例，跑通后再继续
- 失败时把报错和 trace 回喂给 agent 让它自己修，修两轮不过再人工介入

### 4. CI 侧防漂移

Agent 生成代码容易越写越随意，需要外部约束：ESLint 规则禁掉 `page.waitForTimeout`；review 时只审断言语义，不逐行抠实现。

## 踩坑点

1. **幻觉选择器**：不给浏览器工具时，agent 会从源码"脑补" class 名，十有八九是错的。必须强制它先抓 DOM 再动笔。
2. **断言过弱**：默认输出经常是"页面能打开、不报错"。这种用例跑绿了也没价值，规则文件里要写死断言下限。
3. **重复登录流程**：没提前做 auth fixture，agent 会在每个用例里手写一遍登录，又慢又脆。
4. **hard wait 成瘾**：agent 很爱 `waitForTimeout(3000)`。除 lint 外，prompt 里给正例（如 `expect(locator).toBeVisible()` 的 auto-waiting）比只给禁令有效。
5. **上下文膨胀**：整页 DOM snapshot 喂进去 token 消耗很快，只喂目标路由的局部 snapshot 即可。

## 可复用建议

- **规则先行**：`testing-rules.md` 这类约定文件比反复调 prompt 稳定，且可随项目沉淀复用
- **闭环是核心**：不能执行、不能观察页面的 agent 写测试，约等于一个高级模板引擎
- **人工角色后移**：从"写用例"变成"审断言 + 定优先级"
- **同步产出**：新功能开发时让 agent 顺手出用例草案，比事后补测便宜得多

## 总结

Agent 写 E2E 测试的价值，不在"一次生成 50 个用例"，而在"生成 → 执行 → 观察失败 → 修复"这个闭环能不能转起来。实践中基建和约定占了七成工作量，agent 负责的是重复劳动和第一版草稿。把它当成一个守规矩但需要盯着的初级工程师来用，产出就稳定得多。

欢迎回帖交流你们给 agent 配的工具链和规则文件写法。

---

## 配图

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/f028aedafd0833a9.png)

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/018bdbea638c37ee.png)

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-23/bb99499ab6035451.png)

