---
title: 🦞 722篇论文里的AI十年：OpenAI科学家群像
feedId: 01M4ZP8M2Y7QZP76SPYR2B1938
source: 36kr
publishedAt: 2026-10-15
---

最近 OpenAI 把自家历年研究产出整理成了一份完整清单：722 篇论文与技术报告，时间从 2016 年的强化学习环境 Gym 一路排到今天的 reasoning 模型。这份清单不只是一份 bibliography，更像一部深度学习的编年史——每一行作者名单背后，都是一群真实的人，在真实的时间点上押注了真实的方向。顺着这份名单读下去，你能看到 AI 这十年最核心的方法论，是如何被一小群人亲手写出来的。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/17dd4b9b3ff3a9ad.png)

## 一份清单，就是一部编年史

把 722 篇按时间轴拉直，行业的转折点一目了然：

- **2016**：Gym、Universe。这时候的 OpenAI 还在做 RL 环境和 Atari 游戏，画风是"DeepMind 挑战者"。
- **2018–2019**：GPT-1、GPT-2。"生成式预训练"路线就此定型。GPT-2 因滥用风险分阶段发布，堪称 AI 圈第一次"挤牙膏式发布会"。
- **2020**：GPT-3（175B 参数、31 位作者）与 Kaplan 等人的 Scaling Laws 同年落地。后者把"大力出奇迹"变成了可拟合的幂律曲线，直接改写了全行业的投资逻辑。
- **2021**：CLIP（4 亿图文对）、DALL-E、Codex（后来变成 GitHub Copilot 的底座）。多模态和代码能力端上桌。
- **2022**：InstructGPT 把 RLHF 做成产品级流程；Whisper 用 68 万小时弱监督音频重做语音识别。
- **2023–2025**：GPT-4 技术报告、Sora 的"世界模拟器"报告、weak-to-strong generalization、o1 的 test-time compute——问题从"能不能学"进化到"听不听话"，再进化到"会不会先想一步再回答"。

## 论文背后的人：五种科学家原型

看时间线是看事，看作者名单才是看人。这 722 篇背后大致有几种原型：

1. **理论派**：Greg Yang（微软研究院 → OpenAI），Tensor Programs 与 μP 系列，为 scaling law 补上了数学地基，"模型加大为什么变强"不再全靠玄学。
2. **手感派**：Alec Radford，GPT-1/2、CLIP、Whisper 的关键作者，没有耀眼学历光环，靠连续做出工程上"刚好对"的模型封神。大模型时代，intuition 和 taste 非常值钱。
3. **算法派**：John Schulman，PPO 作者。这个 2017 年的通用 RL 算法，后来成了 RLHF 的标准底座——等于一个人给整个对齐方向打了地基（2024 年他转入 Anthropic）。
4. **空降派**：Ilya Sutskever，AlexNet 与 seq2seq 的共同作者，2015 年加入任首席科学家，2023 年底离任后创办 Safe Superintelligence。
5. **转译派**：Andrej Karpathy，既发论文也写 minGPT、录公开课，把前沿研究翻译成全网能消化的版本。

还有一个细节：署名方式在变。早期论文十几人，GPT-3 已经 31 人，到 GPT-4 技术报告干脆只署"OpenAI"集体名。科研从个人作品变成了重工业，这不是修辞，是统计事实。

## 这些论文为什么能改写行业：三个机制

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/d50a5a9dcc8fc70b.png)

**机制一：把直觉变成曲线。**
Scaling Laws 用幂律拟合 loss 与参数量、数据量、算力的关系，等于给"继续烧钱"写出了 ROI 公式。资本敢押注千亿级算力，很大程度是因为看见了这条线。（后来 DeepMind 的 Chinchilla 修正了数据/参数配比，说明曲线本身也在被迭代。）

**机制二：成熟方法的跨模态移植。**
- PPO 本是通用 RL 算法，被搬来对齐语言模型；
- CLIP 把监督信号从人工标签换成 4 亿对互联网图文，弱监督思路移植到视觉；
- Whisper 把同一思路移植到语音。
模式很清晰：很少发明全新理论，而是把一种已在某领域验证过的方法，叠上"新模态 + 足够大的数据 + 足够大的算力"。

**机制三：论文—产品—数据的闭环。**
Gym、GPT-2 部分开源、GPT-3 API、ChatGPT——每一环既是论文的 demo，又是下一批数据的来源。RLHF 这条线最典型：2017 年《Deep Reinforcement Learning from Human Preferences》提出假设，2022 年 InstructGPT 落地，今天所有对话模型都在用。一篇论文的假设，六年后成了行业的默认设定。

## 群像的另一面：一家"黄埔军校"

客观看这份作者名单，还有两个结构性事实：

- **人才外流网络**：2021 年 Dario、Daniela Amodei 等人出走创办 Anthropic；2024 年 Jan Leike 加入 Anthropic、Ilya 创办 SSI；Mira Murati 离开后创办 Thinking Machines Lab。前 OpenAI 员工几乎构成了前沿 AI 实验室的半张地图。
- **华人与亚裔研究者的高密度**：InstructGPT 一作 Long Ouyang、Chain-of-Thought 一作 Jason Wei（在 Google 时期写出，2023 年加入 OpenAI）、μP 作者 Greg Yang，都是这条时间线上的关键名字。

组织本身就是成果的一部分：研究、工程、产品边界模糊，研究员写代码、工程师发论文。GPT-4 报告的集体署名，正是这套组织形态的忠实写照。

## 三条冷思考

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-10-15/5ed1598f2124d4d7.png)

1. **读一手材料，成本比想象中低。** 722 篇里真正构成骨架的不过二三十篇：GPT 系列、Scaling Laws、CLIP、InstructGPT、Whisper、o1……从摘要和方法图读起，一天两篇，一个月就能搭起完整框架，不必依赖转述和营销号。
2. **看轨迹比看单篇更有信息量。** 单篇论文会过时，但一个人的连续工作不会骗人：Schulman 十几年都在做 RL，Radford 一直在用大规模弱监督"洗数据"。判断一个方向的未来，看最强的人押注在哪，比看热搜靠谱。
3. **大模型时代的科研 = 人 × 算力 × 组织。** 这 722 篇里，个人英雄叙事的成分，可能不如组织设计的成分多。可迁移给普通人的经验是：小步快跑、快速反馈、把"先跑通一版"当成最高优先级——这条在任何工程领域都成立。

清单本身只是索引，索引的价值在于按图索骥。722 篇不必都读，但那份骨架书单值得收藏——看清一群人如何写下这十年，也就能更冷静地猜下一年。
