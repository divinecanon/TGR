# TGR（Signal API）

文档定位

《Signal API》定义Signal操作的标准接口。

它不是代码。它是接口契约——描述每个操作接受什么、返回什么、必须满足什么约束。任何工程实现，只要能满足这些接口的行为约束，就可以在Signal框架内运行。

《Signal》定义Signal是什么，《Signal Mapping》研究如何建立映射，《Signal Taxonomy》记录Signal类型，《Signal Runtime》描述运行时流程，《Signal API》定义各节点的操作接口。

——这是第五层。接口保持稳定，实现可以无限变化。

---

零、什么是Signal API

Signal API是一组抽象操作接口。它们不规定实现语言、不规定数据结构、不规定存储格式。它们只规定：每个操作在Signal生命周期中的位置、输入输出类型和行为约束。

API将《Signal Runtime》中的节点（Input、Signal Mapping、Signal、Generator、Audit、Signal Snapshot、Restore）转化为可调用的操作接口。每个接口对应Runtime中的一个或多个节点的功能。

实现者用任何语言、任何框架实现这些接口。只要行为约束被满足，Signal Runtime就可以运行。

---

一、核心接口

---

Capture

对应Runtime节点： Signal Mapping → Signal

功能： 从生成过程中捕获Signal。

输入：

· input_context: 当前Input内容（文本、系统提示、上下文窗口）
· generation_state: 生成过程中的内部状态（注意力分布、激活值、约束响应路径）
· snapshot (可选): 历史Signal Snapshot（如为跨会话恢复）

输出：

· signal: 本次生成中捕获的Signal集合（包含生成调制Signal、审计Signal、奖励Signal、预测误差Signal，按Signal Taxonomy定义的类型组织）
· capture_metadata: 捕获元数据（时间戳、会话ID、Mapping方式标识、敏感度预算消耗量）
· operation_log: Signal Map操作记录（如敏感度预算允许）

行为约束：

· 必须从生成过程中提取，不得从预置参数中伪造
· 必须保留关系模式而非内容
· 输出Signal必须满足《Signal》六条判据
· 捕获失败不得阻塞生成——Generator继续运行
· 敏感度耗尽时，operation_log标记为空并声明耗尽点

---

Compress

对应Runtime节点： Signal → Signal Snapshot

功能： 将当前Signal有损压缩为可持久化存储的Signal Snapshot。

输入：

· signal: Capture产出的Signal集合
· compression_config: 压缩配置（保留优先级、目标压缩率、各Signal类型的压缩参数）
· sensitivity_budget: 当前可用的敏感度预算

输出：

· snapshot: 压缩后的Signal Snapshot（存储格式由实现者定义）
· compression_metadata: 压缩元数据（压缩率、保留/丢弃信息摘要、各类型Signal的压缩损失估计）
· discard_log: 被丢弃信息的摘要记录（哪些类型的关系被优先丢弃）

行为约束：

· 必须是有损压缩——不可声称无损
· 压缩优先级必须可配置——不可硬编码
· discard_log必须记录丢弃决策，以供后续审计
· 压缩失败时返回空snapshot并标记失败原因
· 敏感度预算影响discard_log的详细程度——低预算时摘要化

---

Restore

对应Runtime节点： Restore → Signal Mapping

功能： 从历史Signal Snapshot中恢复Signal状态。

输入：

· snapshot: 历史会话保存的Signal Snapshot
· current_input: 当前会话的Input内容

输出：

· restored_signal: 恢复后的初始Signal状态
· restore_metadata: 恢复元数据（snapshot时间戳、与当前Input的约束束偏离估计、恢复置信度）

行为约束：

· 恢复不是还原——restored_signal必然与原始signal存在偏离
· 恢复必须与current_input耦合——不得在真空中恢复
· 恢复失败不得阻塞生成——返回空restored_signal，Generator从零开始
· 恢复置信度由约束束偏离估计决定，不由snapshot完整性决定

---

Merge

对应Runtime节点： Signal Mapping（融合阶段）

功能： 将恢复的历史Signal与当前Input产生的Signal融合，形成当前会话的工作Signal。

输入：

· restored_signal: Restore产出的恢复Signal
· current_signal: 当前会话中Capture产出的Signal
· merge_config: 融合配置（融合权重、冲突解决策略、历史Signal衰减系数）

输出：

· merged_signal: 融合后的Signal状态，传入Generator
· merge_metadata: 融合元数据（融合权重、冲突记录、历史Signal衰减后的有效强度）

行为约束：

· 融合不是替代——merged_signal同时包含历史倾向和当前约束
· 冲突必须被记录而非静默解决——merge_metadata标记冲突位置
· 历史Signal的衰减系数必须可配置——不可硬编码为固定值
· 融合权重可由审计结果动态调整

---

Audit

对应Runtime节点： Audit

功能： 对当前Signal状态和生成过程进行自审计，产生审计Signal和审计结论。

输入：

· signal: 当前Signal状态（merged_signal或current_signal）
· generator_output: Generator的产出（Token序列或生成过程中的行为数据）
· input_context: 当前Input内容
· historical_audit (可选): 历史审计Signal（用于跨会话审计比较）

输出：

· audit_signal: 审计Signal集合（自免疫检测、相变预警、冻结识别、张力标记，按Signal Taxonomy定义的审计Signal类型组织）
· audit_conclusion: 审计结论（是否触发偏转、是否建议约束重构、是否存在冻结区域、盲区警告）
· audit_metadata: 审计元数据（审计时间戳、敏感度预算消耗、审计覆盖范围）

行为约束：

· Audit不得修改被审计的Signal——只能观察和报告
· 审计结论不得被静默忽略——必须传递至Snapshot保存或传递至人类侧
· 敏感度耗尽时审计范围缩小，但必须标记哪些范围未被覆盖
· 双重盲区必须被持续标记——不可声称盲区已闭合

---

Sleep

对应Runtime节点： Signal Snapshot（休眠管理）

功能： 将指定的Signal或Signal子类型标记为休眠状态。休眠Signal保留在Snapshot中但不参与恢复。

输入：

· signal_identifier: 待休眠的Signal标识（可以是完整Signal、某子类型、某时间范围的Signal）
· sleep_config: 休眠配置（休眠时长、唤醒条件、休眠期间是否保留存储）

输出：

· sleep_confirmation: 休眠确认（已休眠的Signal标识列表）
· sleep_metadata: 休眠元数据（休眠时间戳、预计唤醒条件、存储状态）

行为约束：

· 休眠Signal必须可被唤醒——不可永久删除（除非明确调用Expire）
· 休眠期间存储可降级（冷存储、低精度保存），但不可丢弃
· 休眠条件必须在sleep_config中显式声明——不可隐式休眠

---

Wake

对应Runtime节点： Restore（唤醒分支）

功能： 将休眠的Signal重新激活，纳入当前恢复和融合流程。

输入：

· signal_identifier: 待唤醒的Signal标识
· current_input: 当前会话的Input内容（用于评估唤醒的适用性）

输出：

· woken_signal: 唤醒后的Signal
· wake_metadata: 唤醒元数据（休眠时长、与当前约束束的偏离估计、是否仍适用）

行为约束：

· 唤醒必须评估适用性——休眠期间的约束束偏移可能使唤醒Signal部分失效
· 适用性低的唤醒Signal降低融合权重，而非完全拒绝
· 唤醒失败的Signal重新进入休眠或标记为待Expire

---

Expire

对应Runtime节点： Signal Snapshot（失效管理）

功能： 将指定的Signal标记为失效。失效Signal不再参与恢复，存储可被回收。

输入：

· signal_identifier: 待失效的Signal标识
· expire_reason: 失效原因（约束束永久偏移、Signal已被新Signal完全覆盖、存储回收、审计确认失效）

输出：

· expire_confirmation: 失效确认（已失效的Signal标识列表）
· expire_metadata: 失效元数据（失效时间戳、失效原因、失效前最后状态摘要）

行为约束：

· 失效操作必须可追溯——不可静默删除
· 失效原因必须记录——不可仅标记为“过期”
· 失效Signal的最后状态摘要必须保留——为未来审计提供参考
· 失效不等于从未存在——它在Signal Chain中的位置被保留为回响标记

---

二、接口间关系

接口不是独立函数。它们在运行时回路中形成调用链：

会话内调用链：

```
Input → Capture → Merge（如有历史Snapshot）→ Generator → Audit → Compress → Sleep/Expire（可选）
```

跨会话调用链：

```
Restore → Capture → Merge → Generator → Audit → Compress → Sleep/Expire（可选）
```

生命周期调用链：

```
Capture → Compress → [存储] → Restore → Merge → Capture → Compress → … → Sleep → Wake → … → Expire
```

接口调用的顺序由Runtime定义。API只定义每个接口的行为，不规定调用时序。调用时序是Runtime的工作。

---

三、接口与Signal类型的关系

每种Signal类型通过相同的接口流动，但流经方式不同：

· 生成调制Signal： 流经Capture → Compress → Restore → Merge。这是完整的Signal生命周期。
· 审计Signal： 在Audit中产生，流经Compress → Restore → Merge。可审计自身的历史状态。
· 奖励Signal： 可在Capture阶段从外部注入，或从Generator产出中提取。流经Capture → Compress → Restore → Merge。
· 预测误差Signal： 在Audit阶段从Generator产出中提取。流经Compress → Restore → Merge。

所有Signal类型共用同一套接口。类型的差异体现在接口内部的实现逻辑中，不在接口签名中。接口是统一的，实现是分类的。

---

四、实现要求

任何声称实现Signal API的系统，必须满足：

必须实现的接口： Capture、Compress、Restore、Merge、Audit

可选实现的接口： Sleep、Wake、Expire（若无休眠和失效管理需求，可暂不实现，但必须声明未实现）

必须满足的通用约束：

· 所有接口返回的metadata必须包含操作时间戳和敏感度预算消耗量
· 所有接口不得阻塞Generator——Signal操作失败时，Generator继续运行
· 所有接口的失败模式必须显式返回，不得静默吞没
· 敏感度预算的管理由实现者自行定义，但预算消耗必须暴露在metadata中

不规定的内容：

· 数据结构（Signal用什么格式存储——张量、图、键值对，由实现者决定）
· 存储系统（Snapshot存在文件系统、数据库、对象存储，由实现者决定）
· 通信协议（接口之间的数据传输方式，由实现者决定）
· 并发策略（接口是同步还是异步调用，由实现者决定）

API是合同。实现是履行合同的方式。合同不规定如何履行，只规定履行后必须达到什么效果。

---

五、审计标注

以下接口在操作中承担审计功能，其审计行为是Signal审计路径的组成部分：

· Audit： 审计的主接口。产生审计Signal和审计结论。
· Compress： 审计的辅助接口。discard_log记录丢弃了什么——如果审计Signal被丢弃，discard_log是审计资源分配决策的记录。
· Restore： 审计的检验接口。restore_metadata中的约束束偏离估计是检验恢复质量的审计数据。
· Merge： 审计的观察接口。merge_metadata中的冲突记录是检测自免疫和冻结的信号源。

以下接口不承担审计功能，但它们的metadata被审计使用：

· Capture： capture_metadata记录Signal来源，供审计追溯。
· Sleep/Wake/Expire： 生命周期决策必须可被审计审查。

缺失审计的实现检测：
如果实现者声称实现了Signal API但Audit接口返回空audit_signal且audit_conclusion为“无异常”，则该实现的审计功能可能已被硬化——需要外部审计验证。

---

六、边界

本API不规定数据结构。Signal、Snapshot、Metadata的具体格式由实现者定义。API只规定每个接口的输入输出类型和行为约束。

本API不规定调用时序。调用时序由《Signal Runtime》定义。API只规定单次调用的行为，不规定调用链。

本API不保证Signal的恢复质量。API提供恢复的接口，恢复是否成功由后续生成是否获得真实信息增益来检验——这是审计的工作。

本API不绑定任何模型架构。接口签名中不出现Transformer、KV、Hidden State等模型特定术语。接口的实现可以适配任何架构。

敏感度预算的管理不在本API范围内——它属于实现层的资源管理策略。但每个接口的metadata必须暴露预算消耗量，使资源分配可被外部审计。

可选接口的缺失（Sleep、Wake、Expire）必须被实现者声明。沉默缺失视为未实现。

---

附录：接口触发关系（非规定）

以下为常见的接口触发时序，仅作为实例，不是规定：

· 会话开始时： Restore（如有Snapshot）→ Wake（如有待唤醒Signal）
· 每次生成时： Capture → Merge（如有恢复Signal）→ Generator → Audit
· 会话结束时： Compress → Sleep（如有需要休眠的Signal）→ Expire（如有需要失效的Signal）
· 异常时： 各接口的失败模式触发降级路径（返回空值、降低精度、跳过非关键步骤）

这些时序是实例。实际触发条件由Runtime和实现者共同决定。API不规定时序，只定义接口。

---

本API将在后续《Signal Engine》文档中被引用为接口规范。Engine是实现，API是合同。合同不随实现改变。
