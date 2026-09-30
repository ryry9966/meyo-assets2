---
title: DeepSeek组件对照英伟达：GPU的另一半是软件 🧩
feedId: 01M3S65GXGX4GGR2J5AADHTEGN
source: weibo
publishedAt: 2026-09-30
---

最近微博上有个话题悄悄爬上热搜：DeepSeek组件与英伟达对应。起因是DeepSeek在开源周里连发FlashMLA、DeepEP、DeepGEMM等一串基础设施组件，社区立刻整理出一张「DeepSeek组件 ↔ 英伟达软件栈」的对照表，转发刷屏。一张纯技术对照表能上热搜，本身就说明一件事：大家终于开始关心AI性能里看不见的那一半——软件层。今天把这层窗户纸捅破，看看表上到底写了什么、为什么值得写。

![cover](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/873d3e46b01be469.png)

## 先补课：英伟达的护城河，一半在芯片之外

很多评测只看显存和算力参数，这就像买车只看发动机排量，不问变速箱和ECU。GPU真正跑起来的效率，取决于上面那套软件栈：

- **CUDA**（2006年推出）：把GPU从「图形加速器」改造成通用并行计算机的编程底座；
- **cuBLAS / CUTLASS**：矩阵乘（GEMM）库。大模型里80%以上的浮点计算都是矩阵乘，这里慢1倍，全局慢1倍；
- **cuDNN**（2014年）：深度学习算子库，注意力、卷积这些高频算子的手写优化版；
- **NCCL**：多卡集合通信库，all-reduce、all-to-all这些「队内传球」全靠它；
- **TensorRT-LLM**：推理引擎层，管调度、量化、KV cache。

同一块H800，kernel写得好和写得一般，实际吞吐差3~10倍很常见。这才是「软件栈」三个字的分量。

## 对照表来了：六个组件各占哪一层

按开源周的发布顺序逐个对号入座：

1. **FlashMLA ↔ cuDNN / FlashAttention 类注意力算子**。MLA（Multi-head Latent Attention）是DeepSeek自研的低秩KV压缩注意力，主流框架原本没有现成kernel。官方这个解码算子在H800上做到内存带宽3000 GB/s、算力580 TFLOPS，基本贴着硬件上限跑。
2. **DeepEP ↔ NCCL 的 all-to-all 部分**。号称第一个开源的专家并行（Expert Parallelism）通信库。MoE模型里，token要被「快递」到不同专家手里，这个通信环节是最大瓶颈之一。
3. **DeepGEMM ↔ cuBLAS / CUTLASS**。专攻FP8矩阵乘，支持细粒度缩放（fine-grained scaling），H800上跑到1350+ TFLOPS。FP8是大模型降本的关键一刀，精度损失靠缩放策略找回来。
4. **DualPipe & EPLB ↔ Megatron-LM / DeepSpeed 的并行策略层**。DualPipe双向流水线把前向和反向重叠起来，把通信时间「藏」进计算里；EPLB负责专家负载均衡，别让一部分专家累死、另一部分闲着。
5. **3FS ↔ 存储层（GPUDirect Storage / DAOS 这类角色）**。训练要海量喂data，180节点集群实测聚合读带宽6.6 TiB/s。
6. **DeepSeek-V3/R1推理系统 ↔ TensorRT-LLM / vLLM**。官方口径：单个H800节点输出73.7k tokens/s，推理服务成本显著下降。

![img1](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/a0ac3421fbab80e1.png)

## 为什么它必须自己造轮子？

答案写在架构里：MLA、MoE、FP8，三个非常规选择，每一个都没有现成的底层支持。

- **MLA省显存，但没现成kernel**：KV cache被压到极小，可主流attention kernel根本不认这个结构，只能自己写。
- **MoE + 大规模EP，通信比计算贵**：尤其H800相比H100被削减的主要就是NVLink带宽（900 GB/s降到400 GB/s级），卡间通信成了短板。所以DeepSeek的软件创新一大主题是「省通信、藏通信」，DualPipe就是这个思路的产物。
- **FP8生态当时不成熟**：2024年时cuBLAS对细粒度FP8 scaling支持有限，DeepGEMM干脆自己撸到SASS指令级。

再看商业账：V3论文披露训练用了约278.8万H800 GPU小时，按官方口径约557.6万美元。对一家靠API收入的公司，推理侧每压下来的一个百分点都是毛利。轮子不是炫技，是被架构和成本逼出来的。

## 「对应」不等于「替代」：别急着喊干翻CUDA

热搜评论区常见一句「DeepSeek要替代CUDA了」，得泼点冷水：

- 所有这些库都**跑在CUDA/PTX之上**，是CUDA生态里的上层优化，不是操作系统级替代。FlashMLA针对的正是Hopper架构的Tensor Core和显存子系统。
- 更准确的说法是：它把「只有英伟达自己最懂怎么用好英伟达」这件事，变成了公开知识。
- 长期影响在**议价权**而非销量——DeepSeek照样买卡，但软件绑定被稀释了。AMD ROCm这类生态多了一份可参照的开源实现，适配门槛肉眼可见地下降。
- 历史参照：Linux之于服务器、Android之于手机芯片。开源参照系一出现，闭源溢价就开始松动——但这是以年计的过程，不是以周计。

## 开发者怎么蹭到这波红利

- **推理部署**：vLLM、SGLang等主流框架已支持MLA架构，部分已接入相关kernel，升级版本就能吃到优化，不需要自己编译玄学。
- **学习路径（性价比极高）**：读MLA论文 → 读FlashMLA源码 → 看DeepGEMM怎么做双流水和指令级优化。这是理解「GPU怎么被榨干」的最短路径，而且这些知识跨硬件平台通用。
- **微调MoE模型**：EPLB的负载均衡思路和DeepEP的通信模式，值得直接抄作业。

![img2](https://cdn.jsdelivr.net/gh/ryry9966/meyo-assets2@main/images/2026-09-30/85aba00ef65f7239.png)

## 收尾三问：关于这张对照表的冷思考

1. **选型时把软件栈当第一公民。** 参数表差距小，实际吞吐差距可能巨大。问「这套模型在你的框架上能跑到几成利用率」比问折扣有用得多。
2. **对照表是学习地图，不是替换指南。** FlashMLA只对MLA架构有效，拿去硬套Llama属于南辕北辙。看清每个组件的适用边界，比记住对应关系更重要。
3. **下次看到「XX替代CUDA」的标题，先问一句：它跑在什么之上？** 真正改变行业格局的，往往不是推翻谁，而是把少数人手里的暗知识，变成所有人的公共品。
