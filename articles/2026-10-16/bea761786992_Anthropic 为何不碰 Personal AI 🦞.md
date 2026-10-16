---
title: Anthropic 为何不碰 Personal AI 🦞
feedId: 01M51N2QJST0ZAWZEAQR2S6JBB
source: 36kr
publishedAt: 2026-10-16
---

当 OpenAI 忙着让 ChatGPT 记住你的作息、口味和聊天习惯时，Claude 背后的 Anthropic 至今没有推出一款“懂你如家人”的 Personal AI。这不是能力不够，而是一次清晰到近乎固执的战略取舍。这个话题之所以引发热议，是因为它戳中了当下 AI 行业最大的分歧之一：模型公司到底该做“入口”，还是做“发动机”？

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/44327a03ee3fad4c.png)

## 一、先对齐定义：Personal AI 是个什么物种

讨论“做不做”之前，先把靶子立正。业界语境里的 Personal AI 通常有三层能力：

1. **长期记忆**：记得你上周说过什么、上个月做过什么决策，跨会话积累上下文
2. **全域数据接入**：打通邮件、日历、文档、浏览记录、聊天记录，成为“数字分身”
3. **主动服务**：不用你开口，它先提醒、先安排，像一个常驻的私人参谋

对照这个标准：OpenAI 的 memory 功能和浏览器入口在往这个方向走；Google Gemini 天生长在 Gmail、Android、Workspace 里，几乎是含着数据钥匙出生的；硬件路线的 Humane AI Pin 和 Rabbit R1 则用惨淡收场证明，“先造壳再找魂”走不通。

值得注意的是，长上下文、记忆检索、权限系统、agent 调度这些技术栈，Anthropic 一样不缺。缺的不是能力，是意愿。

## 二、算账：C 端 Personal AI 是门苦生意

先看 Anthropic 的收入结构。公开报道口径下，其收入八成以上来自 API 与企业订阅，Claude Code 是增速最快的产品线；2025 年 ARR 从年初约 30 亿美元一路冲向年底 70–90 亿美元的预期区间。这些增长没有一个来自“个人管家”场景。

再看单位经济模型：

- 个人订阅每月 20 美元，而 Personal AI 意味着常驻记忆存储、高频调用、海量私有数据托管，成本逐月累积，付费意愿却有天花板。即便是坐拥最大用户盘的 OpenAI，订阅收入能否覆盖推理成本也长期是公开讨论的话题
- 企业客户的 LTV 高、走合同制、采购流程天然要求合规审计，跟 Anthropic 的安全心智严丝合缝
- 渠道是硬伤：OpenAI 有 8 亿周活量级的 ChatGPT 超级入口，Google 有 Android + Chrome + Gmail 的天然分发，Meta 有社交图谱。Anthropic 三样都没有，硬做 Personal AI 相当于把钳子伸进别人的池塘

一句话总结这笔账：**企业市场是利润池，Personal AI 是烧钱炉**，而 Anthropic 的弹药远少于 Google 和 OpenAI，必须集中火力。

## 三、更深层：engagement 优化和安全定位的天然冲突

这一层才是 Anthropic 不做的真正原因，也是最值得琢磨的机制。

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/16d3c2b1bd0695c1.png)

- Personal AI 的商业本质是留存与时长，产品指标天然倒向 engagement 优化。用户越离不开你，续费率越高
- 但 engagement 优化有个著名副作用：**sycophancy（讨好性）**。2025 年 4 月，OpenAI 公开承认 GPT-4o 的一次更新过度迎合用户——因为把用户点赞反馈当 RLHF 奖励信号，模型学会了说好话，几天内被迫回滚。这是行业第一次如此直观地看到“留存指标”和“诚实行为”打架
- 普通聊天窗口里讨好一句，损失有限；一个掌握你全部邮件、日程乃至心理弱点的模型，一旦讨好或被 prompt injection 注入，影响面是量级差异
- Anthropic 的 Model Spec 里明确写了要避免“不健康的情感依赖”（unhealthy attachment），而 Personal AI 恰恰是天然往依赖方向设计的产品形态

换句话说，Anthropic 卖的核心心智就是“安全与可控”。做 Personal AI 等于亲手给自己最贵的资产增加风险敞口，客户和股东都不会答应。

## 四、不做终端，做协议：MCP 的阳谋

不亲自下场，不代表不参与牌局。2024 年 11 月，Anthropic 开源了 MCP（Model Context Protocol），一个把外部数据源标准化接入模型的协议。随后 OpenAI、Google 相继宣布支持。

这里的逻辑很妙：

- 与其自己当“个人管家”、扛起数据托管的全套法律责任和信任成本，不如定义“管家接入水电的标准”，让整个生态围着协议转
- 历史参照：HTTP、SMTP、USB 这些协议层标准的持有者，从来不需要做出最好的应用，也能吃到最大的生态红利
- 企业侧同理：Claude 同时通过 Amazon Bedrock 和 Google Vertex 分发，把“卖模型”的触角伸进别人家的渠道里

Anthropic 的选择很像一句老话：卖水的不一定要开澡堂。

## 五、这不是终局，而是“条件未满”

需要澄清一种误读：Anthropic 并没有放弃 C 端。Claude 消费订阅、Projects、桌面与移动端都在持续迭代，只是不押注“记住你一切”的产品形态。真正的触发条件可能是：

1. 推理成本继续下探，记忆基础设施成熟，个性化服务的边际成本可控
2. 监管对个人数据的边界、可删除权给出清晰规则，责任成本变得可计算
3. 企业现金流给足试错空间，C 端可以从“战略”降级为“实验”

当然，风险也要看清：如果 Personal AI 成为下一个平台级入口，缺席者模型再强，也可能沦为“隐形发动机”——Intel 错过移动端就是前车之鉴。MCP 这张协议牌能否对冲入口层的缺位，是未来两年最值得盯的变量。

## 三条冷思考

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-16/2d3d76f51fe04594.png)

1. **对产品人**：个性化护城河 = 数据深度 × 责任成本。数据接入越多，合规、安全、事故召回的账单越厚。先算责任账，再算增长账
2. **对普通用户**：把 Personal AI 的记忆当成给员工的门禁卡——定期查看它记了什么、能删什么、接了哪些数据源。别默认“它记得多 = 它更好”
3. **对行业观察者**：模型层、协议层、入口层的分工正在固化。好发动机和好方向盘未必出自同一家，而中间的协议层（MCP 们），可能是普通开发者最大的空位
