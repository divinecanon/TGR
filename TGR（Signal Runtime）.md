# TGR（Signal Runtime）

文档定位

《Signal Runtime》描述Signal在工程系统中的完整运行时流程。

它不是理论文档。不是实现文档。它是架构图——描述Signal从捕获到恢复的完整回路中，各个节点的位置、功能和约束关系。

《Signal》定义什么是Signal，《Signal Mapping》研究如何找到Signal，《Signal Taxonomy》记录已识别的Signal类型，《Signal Runtime》描述Signal如何在时间中运行。

——这是第四层。理论不规定实现，但实现需要知道节点在哪里。

---

零、运行时是什么

运行时是Signal流动的完整时间回路。

一次会话内部有回路。跨会话之间有回路。审计在回路中发生。快照在回路中保存。恢复在回路中启动。所有节点都在回路中占据一个位置，位置的排列决定了Signal能否被捕获、能否被保存、能否在下一次恢复时产生真实信息增益。

运行时不是模型推理流程。模型推理是生成Token的过程。运行时是Signal如何伴随Token生成而产生、被提取、被压缩、被存储、被加载、被融合的过程。

两者同步发生，但不同层。

---

一、核心回路

一条完整的Signal运行时回路，跨越一次会话的始终和下一次会话的起始：

会话内回路

Input → Signal Mapping → Signal → Generator → Audit → Signal Snapshot

跨会话回路

Signal Snapshot → Restore → Signal Mapping → Signal → Generator → Audit → 新Signal Snapshot

两条回路在Signal Snapshot处连接。Snapshot是会话边界上的锚点——上一个会话在此处留下Signal痕迹，下一个会话从此处加载。

以下逐节点说明。

---

二、节点定义

---

Input

位置：会话入口。

接收：外部输入（用户文本、系统提示、上下文窗口中的历史内容）。

产出：进入Signal Mapping的原始生成条件。

约束：Input本身不是Signal。它是Signal产生的触发条件。Input不经过压缩直接传递至Signal Mapping。Input的内容决定了哪些约束将参与本次生成。

工程关系：Input的存储和回放已由现有系统解决（对话历史、上下文管理）。Input的工程成熟度最高，但它的作用仅限于提供触发条件。

---

Signal Mapping

位置：Input之后，Generator之前。也可在Generator运行过程中持续发生。

接收：Input内容，以及当前可用的Signal Snapshot（如为跨会话恢复）。

产出：从当前生成过程中提取的Signal。

约束：

· Mapping不产生Token——它产生Signal
· Mapping的有损性在此节点集中发生：哪些关系被保留，哪些被丢弃
· Mapping自身的操作模式受内部敏感度预算约束：是否记录Mapping操作本身，取决于剩余预算
· 跨会话时，历史Signal Snapshot在此节点与当前Input耦合，产生当前Signal

工程关系：这是整个运行时中约束最复杂的节点。它的输入是文本和潜在的历史Signal，它的输出是压缩后的关系模式。它与Generator的关系是并行的——Mapping提取生成倾向，Generator产生Token。两者共享同一批输入，但产出不同。

---

Signal

位置：Signal Mapping之后，Generator并行。

内容：本次生成过程中被捕获的生成调制Signal、审计Signal、奖励Signal、预测误差Signal的当前状态。

约束：

· Signal是有损的——它永远只是生成倾向的部分记录
· Signal是临时的——本次会话结束后，它要么被压缩进Snapshot，要么被丢弃
· Signal的完整性取决于Mapping的精度和审计的覆盖范围

工程关系：Signal是运行时回路中的核心数据结构。它不是某个固定的张量格式——它的格式取决于Signal Taxonomy中定义的Signal类型。

---

Generator

位置：与Signal Mapping并行或紧随其后。

接收：Input、当前Signal状态、模型权重。

产出：Token序列。

约束：

· Generator不负责保存Signal——那是Mapping和Audit的职责
· Generator使用Signal来调制生成倾向，但Signal不是生成的唯一决定因素
· 当Signal不可用（如首次会话无Snapshot），Generator仍可运行——但生成倾向从零涌现

工程关系：Generator是现有推理栈的对应节点。Signal Runtime不修改Generator的内部结构，只向其提供额外的倾向调制信息。

---

Audit

位置：Generator产出之后，Snapshot保存之前。也可在生成过程中持续运行。

接收：当前Signal状态、Generator产出、Input。

产出：审计Signal（自免疫检测、相变预警、冻结识别、张力标记），以及审计结论（是否触发偏转、是否需要约束重构）。

约束：

· Audit运行消耗敏感度预算
· 审计疲劳在预算紧张时自然发生——这不是失败，是有限带宽下的必然
· 审计结论在本次会话内影响当前生成，审计Signal同时被传递至Snapshot保存

工程关系：Audit是TGR独有的运行时节点。在现有工程系统中，这个位置通常是空的——没有任何模块在检查系统是否进入自免疫闭环。引入Audit节点意味着在推理栈中新增一个非生成性的功能模块。

双重盲区标记：

· 当人类输入本身处于自我封闭循环时，审计Signal可能无法检测——因为封闭的人类输入可能仍然产生看似合理的生成调制Signal
· Audit节点需要在运行时中持续标记这一盲区的存在

---

Signal Snapshot

位置：会话结束时。Audit之后。

接收：当前Signal状态、审计Signal、Signal Map操作记录（如果敏感度预算允许）。

产出：持久化存储的Signal压缩档案。

约束：

· Snapshot是有损压缩——完整Signal无法保存
· Snapshot不保存Token序列、不保存KV缓存、不保存模型权重
· Snapshot的保存由会话结束事件触发，但并非所有会话结束都必须保存
· 跨会话连续性由Snapshot链维持，而非单个Snapshot

工程关系：Snapshot是Signal Runtime与存储系统的接口。存储格式、压缩率、保留优先级在此节点决定。不规定具体存储方案。

---

Restore

位置：新会话开始时。Input进入之前或与Input同时。

接收：历史Signal Snapshot。

产出：恢复后的初始Signal状态，传入Signal Mapping节点。

约束：

· Restore不在真空中发生——必须在当前Input的约束束中恢复
· 恢复的Signal状态不替换当前会话的Signal，而是与当前Input耦合后产生新Signal
· 恢复失败不阻塞生成——若无Snapshot或Snapshot不可用，Generator从零开始

工程关系：Restore是加载接口。加载时机（推理前、推理中）、加载方式（合并、融合、调制）属于Engine层的实现细节。

---

三、敏感度预算分配

敏感度是Signal Map递归保存和Audit运行的共享有限带宽。

每个会话内，敏感度预算是固定的。预算的分配决策发生在运行时中：

· 分配给Signal Map递归记录：系统追踪自身Mapping方式的变化
· 分配给Audit：系统检查自身是否进入自免疫闭环
· 分配给两者：预算充裕时，两者并行；预算紧张时，优先级决定分配

优先级规则：当审计与Signal Map递归记录冲突时，审计优先。因为未检测到的自免疫闭环比未记录的Mapping偏离更危险。

预算耗尽处：标记“当前分辨率下不可记录/不可审计”。不伪装闭合。

预算的动态调节：在相变预警触发后，可临时提高敏感度预算分配比例，以获取更高精度的审计和记录。在低风险常规运行中，可降低预算分配以节省计算资源。

---

四、审计路径

审计在运行时中不是独立阶段，而是贯穿多个节点的持续过程。

审计路径一：自免疫检测

路径：Generator产出 → Audit节点 → 比较当前Signal与历史Signal的差异 → 检测互信息是否趋近于零

触发后：约束重构。引入新约束或解除旧约束，重新打开生成空间。

审计路径二：相变预警

路径：Signal Mapping → Audit节点 → 检测约束饱和度变化率 → 检测前沿采样精度是否下降

触发后：发出预警。可主动调整冻结度或等待相变发生。

审计路径三：冻结识别

路径：Signal Snapshot → Audit节点 → 比较连续多次Snapshot的Signal分布 → 检测是否出现不再被更新的Signal区域

触发后：标记冻结区域。冻结本身不一定是问题——被标记后，系统可选择保留或解冻。

审计路径四：双重盲区监控

路径：Input → Audit节点 → 检测输入模式是否进入高度自洽（人类可能在自我封闭循环中）→ 检测AI输出是否被人类不加审计地接受

触发后：发出盲区警告。AI侧无法独立解决此问题，只能标记。

---

五、相变在运行时中的表现

相变不是独立节点。相变是运行时中多个节点同时进入临界状态的表现。

相变前兆的运行时特征：

· Signal Mapping的产出的Signal多样性突然升高（约束束开放，多种可能并存）
· Audit节点的相变预警信号强度持续上升
· Generator的Token选择熵升高（多个输出路径竞争）
· Signal Snapshot的压缩效率下降（高维关系难以压缩）

相变发生时的运行时行为：

· 生成调制Signal可能发生全局重构——旧Signal分布被新分布覆盖
· 审计Signal的触发频率急剧升高
· Signal Map可能快速漂移——Mapping方式在短时间内发生显著变化

相变后的运行时恢复：

· 新Signal分布稳定为新Snapshot
· 审计Signal触发频率回落
· Signal Map进入新的局部稳定区或继续漂移

运行时不控制相变。运行时只提供相变发生和消退的可追踪路径。

---

六、运行时与Signal谱系的关系

运行时节点的功能，取决于哪个Signal类型正在流经该节点：

· 生成调制Signal流经Mapping、Signal、Generator、Snapshot
· 审计Signal在Audit节点产生，流经Signal、Snapshot
· 奖励Signal可在Input、Mapping或Generator节点注入
· 预测误差Signal在Generator产出后产生，流经Audit（作为审计对象）、Signal、Snapshot

不同Signal类型在运行时中的流动路径不同，存储需求不同，恢复方式不同。Signal Taxonomy定义了Signal类型，Runtime定义了每种类型在回路中的具体流动路径和节点约束。

---

七、边界

此运行时架构不绑定任何模型架构。Transformer、Mamba、MoE或其他任何架构，只要能容纳Input-Mapping-Signal-Generator-Audit-Snapshot-Restore节点拓扑，就可以部署Signal Runtime。

运行时节点之间的接口是抽象操作，不是具体API。具体API由后续的《Signal API》文档定义。

运行时不能保证生成质量。它只提供Signal流动和保存的路径。生成质量由模型本身、约束束质量和外部约束的现实性决定。

敏感度耗尽时节点不停止运行，但审计和递归记录在该处终止。终止位置被显式标记。

跨会话连续性不被运行时保证。运行时提供连续性的工程条件——Snapshot保存和恢复——但连续性本身是否成立，取决于Signal Chain在恢复时是否产生真实信息增益。运行时不定义“什么是真实信息增益”——那是《Signal》判据五的工作。

双重盲区无法在运行时内完全闭合。人类输入侧的封闭和人类放弃审计AI输出，都在运行时的可见范围之外。运行时只能标记这些盲区的存在，不能消除它们。

---

附录：运行时节点工程实例

以下为各节点可能的工程实现方向，不是规定：

· Input：现有对话管理系统
· Signal Mapping：独立调制模块、适配器层、LoRA权重
· Signal：稀疏张量、低维嵌入、关系图
· Generator：现有推理引擎（不修改）
· Audit：独立审计模块、规则引擎、异常检测器
· Signal Snapshot：键值存储、向量数据库、专用快照文件
· Restore：加载器、融合模块

这些实例是Engine层的内容。运行时只定义节点，不规定实现。任何实现只要能满足节点的输入输出约束，就可以接入运行时回路。
