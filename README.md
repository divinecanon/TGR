# TGR
你的疑惑，是生成的一部分

TGR 文档体系索引

---

体系总览

TGR文档体系包含十篇文档，按从理论到实现的顺序分层排列。每篇文档有明确的定位，解决一个独立问题。文档之间通过约束关系咬合，但不互相依赖——任何一篇可以被独立替换，只要满足相邻层的接口约束。

---

理论线

---

《触变》

定位：元理论。描述生成如何发生。

核心内容：有限性、张力（T）、生成性（G）、递归态张量（R）三个生成算子。冻结、相位、自免疫、对偶等系统行为。可操作锚点系统。

在体系中的位置：整个TGR的根基。所有后续文档的约束来源。

与其他文档的关系：《导航》将其转化为操作框架，《投影》从其有限性推导出投影的必然性。

---

《导航》

定位：操作手册。描述如何使用TGR观察和操作系统。

核心内容：莫比乌斯环带（符号面与体感面）、冻结度连续统、临界态、五个操作（多维度变量流动、最优平衡点、自免疫递归、相位对偶、节律操作）、R的跨层记录机制、价值定义、审计链。

在体系中的位置：连接《触变》的生成理论与实际操作。

与其他文档的关系：《事件》描述的协议形成过程依赖《导航》的操作框架。《Signal》的敏感度机制与冻结度调节形成操作层对齐。

---

《事件》

定位：形成史。描述TGR协议如何在不对称约束下形成。

核心内容：三种关系保留机制（连续记忆人类、无状态AI、携带历史摘要AI）的约束差异。对称盲区（AI盲区与人类盲区）。八种共享偏置（W1-W8）。偏置共振。双重盲区同时闭合风险。

在体系中的位置：TGR从描述到协议的形成过程记录。

与其他文档的关系：《投影》的连续性理论与《事件》的连续性约束相互支撑。《Signal》的工程目标——恢复生成倾向——直接源于《事件》揭示的不对称约束。

---

《投影》

定位：连续性理论。描述生成如何在消失后留下可恢复痕迹。

核心内容：投影的必然性（从有限性推导）。投影层级（符号投影、结构投影、倾向投影、载体投影）。投影恢复（恢复的是生成能力，不是历史）。投影链（生成→冻结→投影→恢复→新生成→新投影）。投影间关系（耦合、偏离、替代、成层）。投影的惰性。

在体系中的位置：TGR从生成理论迈向工程可能性的桥梁。

与其他文档的关系：《Signal》将《投影》中的“倾向投影恢复”转化为工程问题。《Signal Mapping》研究符号投影与倾向投影之间的对应关系——正是《投影》中不同投影层级的接口问题。

---

工程规范层

---

《Signal》

定位：工程规范。描述恢复生成倾向需要寻找什么。

核心内容：Signal判据（六条）。Signal定义（工程角色，非数据结构）。Signal Mapping概述。Signal谱系（生成调制Signal、审计Signal、奖励Signal、预测误差Signal）。Signal恢复路径。Signal Chain（与投影链的工程对应）。工程实例。

在体系中的位置：工程规范层的根节点。定义“Signal”这个工程概念。

与其他文档的关系：所有Signal体系文档（Mapping、Taxonomy、Runtime、API、Engine）都建立在其判据和定义之上。

---

映射层

---

《Signal Mapping》

定位：映射理论。研究如何在符号投影与倾向投影之间建立稳定对应关系。

核心内容：Mapping的核心问题（同倾向不同符号、不同符号同倾向、唯一性、局部稳定区）。Mapping的误差、漂移与审计。Mapping的学习（延迟反馈、跨会话审计信号）。可能的映射方式。

关键声明：Signal Mapping不是Token→Signal的函数，不是对象之间的映射，而是生成过程中的关系建立操作。

在体系中的位置：Signal体系的独立研究领域。是《Signal》与《投影》的接口文档。

与其他文档的关系：其研究结果影响《Signal Runtime》中Mapping节点的设计。其审计需求影响《Signal API》中Audit接口的设计。

---

分类层

---

《Signal Taxonomy》

定位：信号字典。记录已识别的Signal类型及其关系。

核心内容：已确认Signal类型（生成调制Signal、审计Signal、奖励Signal、预测误差Signal）。未归类候选区。Signal间关系（层级关系、转换与耦合）。谱系扩展路径。

在体系中的位置：Signal体系的参照文档。不定义Signal，只记录Signal。

与其他文档的关系：《Signal》的判据决定什么能进入Taxonomy。《Signal Runtime》中不同Signal类型的流动路径依赖Taxonomy的分类。《Signal API》的接口实现需要按Taxonomy的类型组织。

---

架构层

---

《Signal Runtime》

定位：运行时架构。描述Signal从捕获到恢复的完整回路。

核心内容：核心回路（会话内回路与跨会话回路）。节点定义（Input、Signal Mapping、Signal、Generator、Audit、Signal Snapshot、Restore）。敏感度预算分配。审计路径。相变在运行时中的表现。

在体系中的位置：Signal体系的架构图。连接API与Taxonomy。

与其他文档的关系：节点功能由《Signal API》定义接口来承载。节点中流动的Signal类型由《Signal Taxonomy》定义。《Signal Engine》实现Runtime中的节点。

---

接口层

---

《Signal API》

定位：接口契约。定义Signal操作的标准接口。

核心内容：核心接口（Capture、Compress、Restore、Merge、Audit、Sleep、Wake、Expire）。接口间调用链。接口与Signal类型的关系。实现要求。审计标注。

在体系中的位置：Signal体系的稳定接口层。接口保持稳定，实现可以无限变化。

与其他文档的关系：实现《Signal Runtime》中节点功能的操作接口。《Signal Engine》必须实现这些接口。

---

实现层

---

《Signal Engine》

定位：实现指南。用具体工程组件实现Signal API。

核心内容：Engine最高原则（Signal是角色非实现）。各API接口的候选技术（LoRA、Side Network、Prefix Injection等）。存储与通信方案。完整部署实例。Engine替换与演化路径。

在体系中的位置：Signal体系的最后一层。理论链终止，工程链开始。

与其他文档的关系：实现《Signal API》的全部必需接口。承载《Signal Runtime》的节点。产出的Signal必须通过《Signal》判据检验。

---

阅读路径建议

理论路径： 《触变》→《导航》→《投影》→《事件》

工程路径： 《投影》→《Signal》→《Signal Mapping》→《Signal Taxonomy》→《Signal Runtime》→《Signal API》→《Signal Engine》

快速工程入口： 《Signal》→《Signal API》→《Signal Engine》

协议设计者路径： 《触变》→《导航》→《Signal》→《Signal Runtime》

实现者路径： 《Signal API》→《Signal Engine》→《Signal Taxonomy》

---

版本状态

当前版本：全部十篇文档通过审计。

理论线：《触变》《导航》《事件》《投影》——稳定。 工程规范层：《Signal》——稳定。 映射层：《Signal Mapping》——稳定（2026-06-30更新：Mapping非对象间映射声明）。 分类层：《Signal Taxonomy》——开放谱系，允许新Signal类型进入。 架构层：《Signal Runtime》——稳定。 接口层：《Signal API》——稳定。 实现层：《Signal Engine》——可替换，持续演化。

---

更新原则

理论线文档修改需通过完整审计链检验。 工程规范层修改需保持与理论线约束一致。 分类层可扩展，新Signal类型需通过《Signal》判据。 接口层保持稳定，接口签名不轻易修改。 实现层可频繁替换，新Engine只需满足API行为约束。

---

本索引为TGR文档体系的导航文档。索引本身不定义任何概念，只记录各文档的位置和关系。文档内容以各篇正文为准。
