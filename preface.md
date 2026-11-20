## 序：从一个前馈层，到一群专家

稠密模型让每个 token 走过同一套参数，参数越多、每个 token 的算力越贵。混合专家换了一件事：把一个大前馈层拆成许多专家，每个 token 只请少数几个来干活。于是总参数可以涨到万亿，而每个 token 的算力只与被选中的专家有关。参数量不再等于每 token 的计算量，这是整本书的起点。

这本专题精读写的是这条链路的十二道关：先解耦容量与算力，再走路由策略、负载均衡、专家特化、细粒度与共享专家、专家并行、通信重叠、推理部署、选型排障，最后落到显存与工程陷阱。每关回答同一个问题——多拆一群专家，换来了什么容量，又带来什么新代价。

**分工**：与《端侧AI推理精读》《推理模型精读》不同，那两本讲压缩部署与回答那一刻的算力；与《主流AI技术精读》也不同，那本讲方法地图。这本只讲容量与算力怎么解耦，以及稀疏结构从训练到服务这条链上真正卡人的地方。与《分布式系统理论精读》分工：那本讲通用分布式原理，这本讲专家并行这一特定形态里的通信与放置。

## 稀疏前向全景图

+----------------------------------------------------------------------+
| MIXTURE OF EXPERTS: FROM ONE LAYER TO A ROOM OF EXPERTS               |
|                                                                      |
| [1] capacity -> [2] routing -> [3] balance -> [4] specialization      |
| [5] fine-grained -> [6] shared -> [7] parallelism -> [8] comm          |
| [9] serving -> [10] selection -> [11] memory -> [12] pitfalls         |
|                                                                      |
| one feed-forward -> routed experts -> sparse compute, full weights    |
+----------------------------------------------------------------------+

这张图不是模块调用顺序，而是「一个前馈层变成一群专家时，新出现的代价依次落在哪里」：先问容量与算力怎么分，再问谁来选、怎么均衡、怎么避免空转、怎么跨卡通信、怎么真的省下账。越往后，代价越隐性，也越需要在部署前想清楚。

## 十二关速查表

| 关 | 主题 | 一句话 |
|---|---|---|
| 1 | 容量算力 | `MoE` 省的是算力与带宽，不是显存 |
| 2 | 路由 | 前沿默认 `sigmoid` 打分加偏置均衡、top-k 与节点限制 |
| 3 | 均衡 | `routing collapse` 是头号故障，三代方案各有代价 |
| 4 | 特化 | 专家不是学科专家，多按 token 类型与语法分化 |
| 5 | 细粒度 | 切细专家不改算力，改的是组合自由度 |
| 6 | 共享 | 共享专家吸走公共知识，路由专家做长尾 |
| 7 | 专家并行 | `all-to-all` 流动的是激活与元数据，不是权重 |
| 8 | 通信 | 高吞吐与低延迟两类内核，对应预填充与解码 |
| 9 | 部署 | 负载倾斜先开动态复制，不要先丢 token |
| 10 | 选型 | 装得下就别上 `EP`，装不下才需要高并发加高速网络 |
| 11 | 显存 | 按总参数估驻留，别用激活参数估显存 |
| 12 | 陷阱 | 降 top-k 与量化是模型变更，必须跑质量回归 |

## 三层读法：机制、工程、判断

看到一个新方法，先问它属于哪一层。`conditional computation`、`top-k`、`expert choice`、`auxiliary loss`、`z-loss` 是机制层，论文讨论稀疏与路由怎么成立；`expert parallelism`、`all-to-all`、`DeepEP`、`vLLM`、`SGLang`、`TensorRT-LLM` 是工程层，给出可运行的并行与接入方式；`active parameter`、`routing collapse`、`straggler`、`memory bandwidth`、`scaling law` 是判断层，告诉你什么时候这些方法根本不该上。

三层不能互相替代。机制漂亮不等于工程上跑得稳，工程能跑也不等于这笔账划算。综合报告口径建议按「先算显存驻留、再看并发与网络、最后才放大专家并行」的顺序推进，每一步都用自己的模型与集群做对照，而不是照搬别人的基准分数。

## 一个可验收的出口

读完十二关，出口标准不是记住某个模型名，而是能对着一个具体部署回答四句话：你的模型总参数、激活参数与驻留参数分别是多少；路由会不会坍缩、用什么机制均衡；专家并行的通信落在哪个网络域、有没有重叠；负载倾斜时你先做动态复制还是丢 token。四句都答得出，说明这条链路已经能用于判断，而不只是读过一篇综述。

对做系统的人还要多一句：高百分位延迟与每卡空闲比平均利用率更能暴露问题，一卡打满、其余空等正是混合专家特有的失效形态。这一句答不出来，前面四句答得再好也算不对工程账。

## 语料边界与口径

一手论文逐字留存，用于机制与术语的精确写法：Outrageously Large Neural Networks、GShard、Switch Transformers、Mixture-of-Experts with Expert Choice Routing、ST-MoE、BASE Layers、GLaM、Mixtral of Experts、DeepSeekMoE、MegaBlocks、Sparse Upcycling、Soft Mixtures of Experts、MoE Meets Instruction Tuning、OLMoE、Mixture-of-Depths、DeepSeek-V2、Scaling Laws for Fine-Grained MoE、Auxiliary-Loss-Free Load Balancing 与 DeepSeek-V3 技术报告。官方文档给出接入形态：Mixtral 与 DeepSeek 模型卡、Hugging Face 混合专家说明、官方博客与 DeepSeek-V3 模型卡。

综合报告只补工程判断，比例、成本与趋势一律标注「综合报告口径」。十二关的顺序、场景与速查表属于「延伸，非语料原文」，不加行内代码标记；行内代码只用语料中确实出现的标识符。安全与合规内容只作工程提示，不构成法律意见。