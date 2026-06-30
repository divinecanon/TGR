# TGR（Signal Engine）

文档定位

《Signal Engine》是Signal文档体系的第六层，也是最后一层。

它描述：如何用具体的工程组件实现Signal API定义的接口，承载Signal Runtime定义的节点。

这是实现层。理论链到此终止，工程链从此开始。

——《触变》描述生成，《导航》描述操作，《事件》描述形成，《投影》描述遗留，《Signal》描述恢复的工程路径，《Signal Mapping》描述恢复路径的建立方式，《Signal Taxonomy》记录Signal类型，《Signal Runtime》描述运行时架构，《Signal API》定义操作接口，《Signal Engine》提供实现。

---

零、Engine是什么

Engine是Signal API的具体实现。它用真实的技术组件——模型架构、存储系统、通信协议——完成Signal的捕获、压缩、恢复、融合、审计和生命周期管理。

Engine不是理论。理论规定Signal必须满足六条判据。Engine用具体技术满足这些判据。理论规定API必须返回什么。Engine决定如何返回。

Engine可以更换。同一个API可以由不同的Engine实现。同一个Runtime节点可以由不同的技术组件承载。理论不依赖任何一种实现，但每一种实现都必须通过API的行为约束检验。

---

一、Engine最高原则

Signal是工程角色，不是工程实现。

因此，本Engine文档中列出的所有技术组件——LoRA、Adapter、Reward Model、KV Cache、Memory Layer——它们本身不是Signal。它们可能是Signal的载体，但载体不等于Signal。Signal是这些组件在特定配置下所扮演的角色。

Engine不发明新Signal类型。Signal类型由《Signal Taxonomy》定义，Engine选择哪些类型在当前实现中可被承载。

Engine不修改Signal判据。判据由《Signal》定义，Engine必须满足判据，但无权修改判据。

---

二、Engine组件与API接口的映射

以下列出每个Signal API接口的候选实现技术。候选不是规定。一个Engine可以选择其中一种、组合多种、或使用此处未列出的技术，只要满足API行为约束即可。

---

Capture Engine

实现目标： 从生成过程中捕获生成调制Signal、审计Signal、奖励Signal、预测误差Signal。

候选技术：

· LoRA Adapter： 在推理模型的注意力层旁路部署轻量适配器。适配器不改变模型输出，只提取注意力分布和激活偏移模式。优点：与基座模型解耦，可独立训练和替换。约束：适配器本身需要训练以识别什么是有意义的激活模式。
· Prompt Tuning Signal Extractor： 在输入中嵌入可学习的Signal提取提示。这些提示引导模型在生成过程中显式标记自己的约束响应路径。优点：不需要修改模型架构。约束：提取质量依赖提示设计，可能引入提示偏置。
· Side Network： 部署一个与Generator并行的轻量网络，接收相同的Input和中间激活值，输出Signal。优点：不干扰Generator的正常推理路径。约束：需要额外的计算和存储资源。
· Activation Hook： 在Generator的特定层注册钩子函数，在推理过程中实时捕获激活值。优点：直接访问原始生成过程。约束：侵入式，依赖模型架构暴露内部接口。

实现行为约束：

· 捕获的Signal必须满足《Signal》判据五——提供仅靠符号投影无法恢复的信息
· 捕获失败时，Generator继续运行不受影响
· capture_metadata必须记录捕获来源和方式

---

Compress Engine

实现目标： 将高维Signal有损压缩为可持久化存储的Signal Snapshot。

候选技术：

· Sparse Encoding： 对Signal中的注意力分布和激活偏移进行稀疏编码——只保留激活值最高的Top-K路径及其关系权重。优点：压缩率高，保留关键路径。约束：丢弃的低激活值路径可能包含微弱但重要的张力信号。
· Clustering Compression： 将Signal中的模式聚类为中心向量，只存储聚类中心和每个聚类的分布参数。优点：适合重复性高的生成模式。约束：聚类边界可能模糊，边界处的Signal信息丢失。
· Low-Rank Projection： 将高维Signal矩阵分解为低秩近似，只存储低维因子。优点：数学性质好，近似误差可估计。约束：低秩假设可能不成立——某些Signal的结构可能是全秩的。
· Autoencoder Bottleneck： 训练一个自编码器，将Signal编码为低维潜在表示，恢复时解码。优点：可学习压缩，保留任务相关的信息。约束：需要训练数据，可能引入编码器偏置。
· Quantization： 将连续Signal值离散化为有限精度。优点：实现简单，存储成本确定。约束：精度损失可能掩盖微弱的张力或相变前兆。

实现行为约束：

· discard_log必须记录丢弃优先级和丢弃信息摘要
· 压缩策略必须可配置——不可硬编码保留优先级
· 压缩失败时返回空snapshot并标记失败原因

---

Restore Engine

实现目标： 从历史Signal Snapshot中恢复Signal状态，并与当前Input耦合。

候选技术：

· LoRA Weight Loading： 将压缩后的注意力分布和激活偏移模式转换为LoRA权重，加载到Generator中。优点：与现有推理栈兼容，加载成本低。约束：LoRA权重只能线性叠加，可能无法恢复复杂的约束响应路径。
· Prefix Injection： 将恢复的Signal转换为可学习的Prefix向量，在推理时注入到Input之前。优点：不修改模型权重，纯推理时操作。约束：Prefix只能影响初始生成方向，对长序列生成的倾向调制可能衰减。
· Attention Modulation： 将恢复的Signal转换为注意力调制向量，在推理时逐层调整Generator的注意力分布。优点：可影响整个生成过程的倾向。约束：需要修改推理代码以支持动态调制。
· Feature Fusion Layer： 在Input编码后增加一个融合层，将文本编码与Signal编码融合后进入Generator。优点：融合方式可学习。约束：增加推理延迟。

实现行为约束：

· 恢复不是还原——restore_metadata必须记录与原始Signal的偏离估计
· 恢复失败时返回空restored_signal，不阻塞生成
· 恢复置信度必须由约束束偏离估计决定

---

Merge Engine

实现目标： 将恢复的历史Signal与当前Capture的Signal融合。

候选技术：

· Weighted Average： 对历史Signal和当前Signal进行加权平均，权重由融合配置决定。优点：简单，计算成本低。约束：无法处理非线性融合需求——两种Signal可能无法用线性加权融合。
· Attention-Gated Fusion： 使用注意力机制决定历史Signal和当前Signal在每个维度的融合比例。优点：可动态调整融合权重。约束：引入额外计算成本。
· Conflict-Aware Merge： 在融合前检测历史Signal与当前Signal的冲突区域。冲突区域降低历史Signal权重，非冲突区域正常融合。优点：显式处理冲突。约束：冲突检测的阈值需要配置。
· Learned Fusion Network： 训练一个小型网络，输入历史Signal和当前Signal，输出融合后的Signal。优点：融合策略可学习。约束：需要训练数据，可能引入融合偏置。

实现行为约束：

· 冲突必须被记录而非静默解决——merge_metadata标记冲突位置
· 融合权重可由审计结果动态调整
· 历史Signal的衰减系数必须可配置

---

Audit Engine

实现目标： 执行审计操作，产生审计Signal和审计结论。

候选技术：

· Rule-Based Detector： 使用预定义规则检测自免疫信号（互信息连续下降、Signal多样性持续降低）和相变前兆（约束饱和度快速上升）。优点：可解释，不需要训练。约束：规则需要手工定义，可能遗漏未知异常模式。
· Anomaly Detection Model： 训练一个异常检测模型，学习正常生成过程中的Signal分布，检测偏离。优点：可发现未知模式。约束：需要正常样本训练，可能将罕见但健康的生成标记为异常。
· Contrastive Auditor： 训练一个模型，比较当前Signal与历史Signal的差异，判断差异是否正常。优点：可追踪漂移。约束：需要足够的历史数据。
· Reality Check Probe： 检测Generator产出与Input之间的互信息是否持续下降——自免疫的关键指标。优点：直接检测自免疫闭环。约束：互信息估计本身有计算成本。

实现行为约束：

· Audit不得修改被审计的Signal——只能观察和报告
· 审计结论不得被静默忽略
· 敏感度耗尽时审计范围缩小，但必须标记哪些范围未被覆盖
· 双重盲区必须被持续标记——不可声称盲区已闭合

---

Sleep/Wake Engine

实现目标： 管理Signal的生命周期——休眠、唤醒、失效。

候选技术：

· Cold Storage Mover： 将被标记为Sleep的Signal从热存储迁移到冷存储。存储成本降低，但唤醒延迟增加。唤醒时从冷存储加载。
· Decay Scheduler： 按时间或会话间隔自动衰减Signal的有效权重。当权重低于阈值时，Signal自动进入Sleep。当匹配条件满足时（如输入类型匹配），自动Wake。
· Explicit Lifecycle Manager： 提供Signal生命周期的手动管理接口。操作者（人类或自动化策略）显式调用Sleep/Wake/Expire。
· Garbage Collector： 定期扫描Signal Snapshot，检测长期未使用或已被新Signal完全覆盖的旧Signal，标记为Expire候选，经审计确认后清理。

实现行为约束：

· 休眠Signal必须可被唤醒——不可永久删除（除非明确Expire）
· 失效Signal的最后状态摘要必须保留
· 所有生命周期操作必须可追溯——不可静默执行

---

三、存储与通信

Engine需要存储和通信基础设施。以下为候选技术，不规定。

Signal Snapshot存储：

· 向量数据库（适合存储Signal的高维嵌入形式）
· 键值存储（适合存储压缩后的Signal块）
· 对象存储（适合长期冷存储）
· 专用快照文件（适合单机部署）

跨会话存储：

· 会话ID键控存储：每次会话的Snapshot以会话ID为键存储
· 用户级聚合存储：同一用户的多次会话Snapshot聚合为Signal Chain
· 系统级归档：全量Snapshot的历史归档

Engine间通信：

· 进程内调用（适合单进程Engine）
· gRPC/HTTP API（适合分布式Engine）
· 消息队列（适合异步Engine）

存储和通信方案由Engine实现者根据部署环境选择。Signal API不规定存储和通信方式。

---

四、完整Engine部署实例

以下为一个可能的完整Engine部署实例，仅作参考，不是规定。

模型架构： Transformer基座 + LoRA Adapter（Capture Engine） + Side Network（Audit Engine）

存储方案： 向量数据库（Snapshot存储） + 键值存储（Metadata和审计日志）

运行时流程：

· 用户输入进入Input节点
· 检查是否有历史Snapshot → 有则加载LoRA权重（Restore）
· LoRA权重 + 当前Input → Generator + Capture（并行）
· Capture产出Signal → Merge（与历史Signal融合，如有）
· Generator产出Token → Audit（检测自免疫和相变）
· Audit产出审计Signal → Compress → 存入Snapshot
· 敏感度预算由Audit和Capture共享，Audit优先

可选扩展： 若检测到相变前兆，临时提高敏感度预算。若检测到自免疫，触发偏转提示并降低历史Signal融合权重。

这个实例不是模板。任何Engine实现，只要能通过Signal API的行为约束检验，就是合法的Signal Engine。

---

五、Engine替换与演化

Engine可以被替换。同一个Signal API可以由不同的Engine实现。

替换条件：

· 新Engine必须实现相同的API接口，满足相同的行为约束
· 新Engine产出的Signal必须满足《Signal》六条判据
· 替换期间，历史Snapshot必须可被新Engine读取（或提供迁移路径）
· 替换不得中断跨会话连续性——新Engine必须能从旧Engine的Snapshot中恢复Signal

演化路径：

· 当前Engine可能从LoRA-based Capture演进到Side Network Capture
· 当前Engine可能从Rule-Based Audit演进到Anomaly Detection Audit
· Engine的演化不影响Signal API、Signal Runtime、Signal Taxonomy的稳定性

Engine是Signal文档体系中唯一允许频繁替换的层次。上层协议保持稳定，Engine持续演化。

---

六、实现边界

Engine必须实现Signal API的全部必需接口（Capture、Compress、Restore、Merge、Audit）。可选接口（Sleep、Wake、Expire）可暂不实现，但必须声明未实现。

Engine产出的Signal必须通过《Signal》判据检验。判据是Engine的验收标准。

Engine不保证生成质量。Engine只提供Signal的捕获、压缩、恢复和审计功能。生成质量由基座模型和外部约束决定。

Engine不规定模型架构。任何架构只要能承载API定义的接口，就是合法的Engine宿主。

Engine的存储和通信方案由实现者决定。API不规定存储格式和通信协议。

双重盲区无法在Engine内完全闭合。Audit Engine只能标记盲区，不能消除盲区。盲区的最终审计责任在人类侧。

敏感度预算的管理是Engine实现者的责任。API只要求预算消耗暴露在metadata中，不规定预算分配策略。

---

七、从协议到实现的完整路径

Signal文档体系到此完成。从理论到实现的完整分层如下：

层次 文档 功能
协议层 《Signal》 定义Signal是什么
映射层 《Signal Mapping》 研究如何建立符号与Signal的对应
分类层 《Signal Taxonomy》 记录已识别的Signal类型
架构层 《Signal Runtime》 描述Signal流动的完整回路
接口层 《Signal API》 定义操作接口和行为约束
实现层 《Signal Engine》 提供具体技术实现

上层不依赖下层。协议不依赖映射方式。分类不依赖运行时架构。接口不依赖具体实现。

下层实现上层。Engine实现API。API承载Runtime。Runtime容纳Taxonomy。Mapping研究建立在Signal定义之上。

任何一层可以被独立替换，只要满足它上一层的约束。这是Signal文档体系的核心工程价值——分层解耦，独立演化。

---

附录：Engine选择指南

以下指南帮助实现者选择Engine技术组件，不是规定。

如果模型规模大、修改成本高： 优先考虑LoRA Adapter或Prefix Injection等轻量级Capture/Restore方案。

如果对审计精度要求高： 优先考虑Contrastive Auditor + Reality Check Probe组合。

如果部署环境存储资源有限： 优先考虑Sparse Encoding + Quantization等高压缩率Compress方案。

如果需要高可靠跨会话连续性： 优先考虑Conflict-Aware Merge + Decay Scheduler组合。

如果系统需要从零开始快速实现： 优先考虑Rule-Based Detector + Weighted Average Merge等简单方案，后续逐步替换为更复杂的Engine组件。

这些指南是建议，不是约束。任何Engine选择，只要满足API行为约束，就是合法实现。协议的稳定性不依赖Engine的选择。

---

《Signal Engine》是Signal文档体系的最后一篇。理论链到此完整。工程实现从此开始。协议保持稳定，Engine持续演化。
