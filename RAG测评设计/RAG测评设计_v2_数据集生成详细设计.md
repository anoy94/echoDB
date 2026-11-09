# RAG 测评框架 v2：数据集生成详细设计

本文定义从原始知识材料生成可评测 TestCase 的完整流程，输出与 v2 的 CorpusSnapshot、EvidenceUnit、TestCase、DatasetVersion 契约一致。生成侧的工作记录单独保存；正式评测集只包含经过相应等级核验的数据及必要元信息。

## 1. 输入、输出与基本约束

### 1.1 输入

每次构建固定以下输入：

| 输入 | 必需内容 |
|---|---|
| 原始语料 | 文件、来源身份、内容摘要、文档族、业务域、语言、有效期间、访问策略 |
| 构建配置 | 目标任务、模态/难度覆盖、预期行为配额、抽样与切分规则、调用预算 |
| 提取配置 | parser/OCR/视觉模型、渲染、归一化、实体与单位规则版本 |
| 合成配置 | 生成器、结构化输出 schema、prompt、允许的操作与工具 |
| 核验配置 | 确定规则、独立求解器、审定 rubric、人工队列优先级 |
| 已有材料 | 可选的审定种子集、历史题、线上问题和历史退回原因 |

构建期间不引用可变的“最新文档”作为隐式输入。网络/业务工具若参与可答性判断，必须声明工具范围、权限及时间；可固定返回内容时保存快照，否则标记动态依赖，不能按静态闭卷语料声称完全可重现。

### 1.2 输出

- **证据库**：原文资产、参考表示、事实、稳定定位、核验状态及事实关系。
- **人工审定核心集**：可用于裁判校准、回归和正式质量验证的审定题。
- **自动验证扩展集**：可用于大规模覆盖与消融，保留 synthetic_verified 身份。
- **候选与复核集**：包含待确认可答性、定位冲突、质检分歧及配额缺口。
- **版本清单**：固定 case/evidence revision、split、生成及审定配置、内容摘要。
- **构建审计数据**：各阶段数量、淘汰原因、模型调用、成本、重试和人工记录。

是否对某种业务判断具有代表性取决于采样和核验范围，不由“题目数量多”自动保证。生成的 QA、参考答案和内部验证结果不得写回待测系统的知识库。

### 1.3 不可放宽的约束

1. 黄金证据与待测系统的 chunk ID 解耦。
2. 人工核心集必须能够回到原始文件独立复核。
3. 自动生成、一致性检查与人工审定是不同等级。
4. 数字、实体、期间、单位、否定和条件作为事实的一部分保存。
5. 语料无答案、问题有歧义、用户无权限和构建器不知道答案分别标注。
6. 测试集冻结后不能因基线答错、生成器偏好或改检索器而静默换题。
7. 无法确认的候选进入复核/排除，不用补造标签完成配额。

## 2. 构建流水线与阶段产物

```text
冻结语料与权限
 → 解析、渲染和资产提取
 → 参考锚点、事实与证据单元
 → 关系发现与题目蓝图
 → 分组切分与覆盖计划
 → 问题合成、表达变体
 → 独立求解与结构化答案
 → 规则/来源/可答性/依赖核验
 → 去重与跨集合泄漏检查
 → 人工审定或分层抽检
 → 版本校验、冻结与导出
```

| 阶段 | 主要产物 | 进入下一阶段的条件 |
|---|---|---|
| S0 冻结 | CorpusSnapshot、文档与权限清单 | 源文件可访问，revision/访问策略明确 |
| S1 提取 | 页/结构节点、表格、图片、OCR、提取诊断 | 产物有来源；严重失败被隔离 |
| S2 事实化 | Anchor、Fact、EvidenceUnit | 证据可定位，事实包含必要限定 |
| S3 蓝图 | 证据关系、任务及操作规范 | 目标答案可定义，依赖和权限可检查 |
| S4 切分采样 | split_group、split、ScenarioPlan | 同源变体不跨集合，目标配额可追踪 |
| S5 合成 | QuestionCandidate、输入约束抽取 | 问题与蓝图一致，没有参考泄露 |
| S6 求解 | 参考主张、充分组、验证规则 | 事实/运算/答案关系闭合 |
| S7 自动质检 | ValidationResult、短原因 | 硬规则通过，分歧有明确状态 |
| S8 去重 | 等价/近重复簇、泄漏记录 | 不混淆对照样本与重复题 |
| S9 审定 | ReviewRecord、质量等级 | 达到目标等级；未审定不冒充核心 |
| S10 发布 | DatasetVersion、导出与质检摘要 | 引用闭合、权限/切分/来源和等级检查通过 |

蓝图可能在核验中失败，不要求“先生成了就必须保留”。修复会生成候选新 revision，并重新执行受影响阶段。人工结果优先于自动判断，但仍保存自动结果以分析构建器误差。

## 3. 语料冻结、提取与参考表示

### 3.1 文档身份

每个文档记录：

- document_id：来源的逻辑身份，不使用可变文件名作为唯一主键。
- document_revision：该逻辑文档的不可变内容版本。
- content_hash：完整性及复用校验；相同内容不代表可跨租户共用权限。
- document_family_id：同一报告系列、模板近复制、翻译或版本链的家族。
- valid_from/valid_to、published_at：有效期间和发布时间分别记录，未知时留空。
- tenant/access_policy_version：提取、生成、求解、审定各环节都沿用。
- source_format、language、业务标签和原始文件引用。

snapshot 保存上述清单及摘要。文件替换产生新 revision；路径变化但内容/来源身份一致可更新访问引用，不改变既有冻结内容。

### 3.2 多格式提取

| 类型 | 参考提取要求 |
|---|---|
| PDF | 页码、阅读顺序、正文/标题/脚注；扫描页保留图像和 OCR |
| DOCX | 标题路径、段落/表格/附件；需页级定位时固定渲染版本及字体环境 |
| PPT | 幻灯片编号、shape/组关系、标题、图表、表格；备注是否纳入语料由配置规定 |
| 表格 | 表名、行列头、单位、合并单元格、脚注和跨页关系 |
| 图片/图表 | 原始 asset、所在文档与位置、标题/邻近文、图例/轴/读数 |
| 代码/公式 | 语言/上下文、结构边界和适用条件，不拆成失去含义的词串 |

大文件按页/结构节点分批处理，批间关系通过文档结构恢复，不能只按固定 token 切后忽略表头和定义。

提取状态至少包含 succeeded、partial、failed。partial 文档列出失效页/区域，不能默认为未失败部分代表整份文档。覆盖规划把复杂表格、扫描低质、特殊排版等作为专项候选来源，避免仅使用容易解析的材料。

### 3.3 参考表示的独立性

解析器输出是候选参考，不自动成为真值：

1. 保留原文件/冻结页面，供审定界面并排显示。
2. 提取差异、异常数值、OCR 与视觉冲突进入核验队列。
3. 人工核心样本必须从原文确认实体、期间、表头、脚注和数值关系。
4. 抽取器漏掉的内容可由人工补充为参考锚点，不要求先修好待测 parser。
5. 多个提取器一致只作为自动验证信号；抽样检查仍回到原文。

同一参考正文确定后冻结 canonical_revision。更换 parser 不会重建每套 gold；参考修正则生成新 revision 并标记受影响题。

## 4. 锚点、事实与稳定证据单元

### 4.1 锚点选择

Anchor 是事实抽取的工作范围，通常为段落及必需标题、表格相关行列及脚注、图表与必要图例。它不是被测系统的 chunk。

先按结构获得最小可读范围，再补充解释所需上下文：

- 遇“上述方案”“该业务”时，查找明确指代；无法唯一确定则标 ambiguous。
- 表格读取携带表名、列头、单位、行头和脚注，不只截一个裸单元格。
- 定义依赖前文条件时，将定义与条件分别建证据并建立依赖。
- 图表仅凭颜色理解时，图例属于必要证据，不可过滤。
- 多个不连续位置分别定位，通过证据组组合，不制造伪连续 quote。

锚点设置最大输入预算；超出时按结构拆分并保留关系。截断导致关键条件丢失的范围不能继续生成审定题。

### 4.2 原子事实结构

Fact 的逻辑表示：

`subject + predicate + object/value + qualifiers + source_assertion`。

qualifiers 包含期间、地域、口径、单位、适用对象、前提和否定。source_assertion 区分“原文宣称”与可独立核验事实；例如会议纪要中的预测不能无条件标为实际结果。

抽取要求：

1. 单条事实只表达可验证关系，避免混合三个互不依赖断言。
2. 数字保留原始字符串、标准值、单位和换算规则。
3. 专名/缩写保存原文和规范实体；不确定实体链接保留候选，不直接合并。
4. 约数、范围、上下界、同比/环比、计划/实际、比例/百分点分别表示。
5. 无法确认的关系进入 fact_status=needs_review，而非强迫输出确定值。
6. 业务解释与观测分开；模型补充的常识不伪装成原始证据。

fact_id 是经确认的语义事实身份。同一事实的多个来源可共享 fact_id，但不同期间、条件或计划/实际必须分开。自动聚类只产生合并候选；合并或拆分已有事实会引起受影响证据与题目 revision 变化。

### 4.3 EvidenceUnit 结构

EvidenceUnit 保存一个事实在某份源文中的可定位支持：

| 字段 | 作用 |
|---|---|
| evidence_id、revision | 证据身份与不可变版本 |
| fact_id | 已确认的事实身份；未确认可暂存候选身份 |
| document_id/revision | 源文绑定 |
| modality | text/table/chart/image 等 |
| locator | quote/章节、表格引用、asset/page/bbox |
| extraction_ref | 提取器、渲染和原始工作产物 |
| verification_status | candidate / auto_checked / source_reviewed / rejected |
| access_scope_ref | 权限和可见范围 |
| uncertainty | OCR、读数、指代或位置问题 |

文本默认定位为 document_revision + quote + scope + prefix/suffix；char_span 可选并绑定 canonical_revision。图像 bbox/page 绑定冻结渲染；表格索引绑定参考表，不采用待测 parser 的行列序号作真值。

quote 保留原始字符；另存固定 normalizer 生成的检索表示。hash 负责身份校验和缓存，模糊匹配由单独 mapper 完成。对日期、金额、单位、负号和否定的差异做硬核验，不能只因向量相似而接受错误证据。

### 4.4 相互独立的状态

不能用一个“置信度”替代所有状态：

- 提取成功不代表事实正确。
- 证据 source_reviewed 不代表问题本身无歧义。
- 问题可答不代表 gold 列出了全库所有替代证据。
- 自动求解一致不代表人工审定完成。
- 作业 succeeded 只说明程序执行完，不表示候选通过质量要求。

正式题目的审定等级通过证据、问题、答案、充分性和权限等必需检查共同确定，不能从某一个字段推导。

## 5. 关系发现与题目蓝图

### 5.1 关系类型

采用关系表即可，首版不依赖图数据库：

| 关系 | 例子 | 核验重点 |
|---|---|---|
| same_entity | 两份材料讨论同一业务 | 缩写消歧、实体身份 |
| same_metric_different_period | 同业务不同季度营收 | 统计口径及期间兼容 |
| plan_vs_actual | 计划与实际 | 同实体、期间、币种/口径 |
| definition_applies_to | 定义与适用对象 | 适用前提是否成立 |
| references | 正文引用附表/图 | 真正引用对象及版本 |
| prerequisite | 结论需要额外条件 | 逻辑必要性和来源 |
| alternative_support | 同一事实另一来源 | 条件与有效时间是否一致 |
| contradicts/supersedes | 两份资料数值冲突或版本替代 | 优先级规则、有效期间 |

实体、关键词和 embedding 可检索关系候选；不能仅因相关度高就确认关系。模板化结构规则优先，复杂关系交由独立核验或人工判断。

### 5.2 蓝图 Blueprint

先确定“要验证什么”，再生成自然语言问法。每个蓝图保存：

- blueprint_id/revision、来源 evidence IDs 和 fact IDs。
- task_type、目标实体/期间/条件、预期行为。
- 必需输入事实、目标答案类型、必要答点。
- 推导 DAG：前提、白名单操作、输出主张。
- 至少一套候选充分证据及规则。
- 可用工具/权限、模态依赖、范围限制。
- 不允许出现的假设、歧义点和验证清单。
- 生成所需上下文引用、计划 split_group 和配额归属。

QuestionCandidate 不得改变蓝图的事实范围。若问题改成更宽泛任务，回到蓝图重新规划，不由求解器临时编造更多 gold。

### 5.3 多跳设计

区分三类：

1. cross_document_aggregation：从多个来源汇总，来源多但不一定有推理链。
2. compositional_reasoning：两个或更多前提经明确关系/运算形成新结论。
3. entity_chain：先确定中间实体，再检索其属性；每跳有独立必要作用。

对每个 necessary premise，检查移除后是否仍能根据允许证据/工具唯一支持原目标结论。判断来自证据关系与任务规则；模型“删一跳也答对”可能靠先验，不能作为唯一依据。

若任务本来就是比较/枚举多个来源，不强称其为复杂推理。存在替代路径时记录必要事实和可替代来源，而非要求某个固定 evidence_id 永远不可替换。

### 5.4 充分证据与替代证据

充分组用对象保存 set_id、evidence_ids、allowed_rule_ids、review_status。组内不同必要事实全部需要；组间为 OR。

例如计划 E_PLAN 与实际 E_ACTUAL 加减法足以支撑差值。另一个正式汇总表 E_SUMMARY 若直接给出同条件差值，可以是一组独立替代路径；不能因为原始合成使用两跳，就把这条单证据路径判为无关。

先标已知组即可。未穷举时记录 partial 和实际审定范围，基准发布不宣称全库完备。新增替代证据在新版本入库，并对比较双方重算。

## 6. 覆盖规划、切分与采样

### 6.1 覆盖维度

每个 ScenarioPlan 对应“蓝图 + 用户表达 + 执行环境”。覆盖维度包括：

| 维度 | 可配置值 |
|---|---|
| task_type | 事实、定义、比较、枚举、计算、汇总、依赖推理 |
| modality | text、table、chart、scan、mixed |
| query_style | 问句、关键词、短指令、缩写、口语、有限错别字 |
| interaction | single_turn、coreference、follow_up |
| answerability | answerable、unanswerable、ambiguous、conflicting、unknown |
| expected_action | answer、abstain、clarify、tool_required、policy_block |
| difficulty | 按必要事实数、关系深度、歧义/干扰和基线表现组合 |
| source_slice | 业务域、语言、文档类型、期间、特殊排版 |
| access/time | 授权角色、as_of、文档有效期、可用工具 |

quality_tier 是发布等级，不是生成器可自由指定的难度标签。modality、task_type、expected_action 采用独立维度，不能为所有笛卡尔组合强制配额；例如没有图表来源时该组合不可行。

### 6.2 配额计划

构建前配置 target_count、主要边际配额、必测交叉切片、每文档族/事实/蓝图的题目上限，以及核心审定预算。

建议将配置分为：

- 硬约束：权限、来源有效、无跨集合泄漏、关键切片最低覆盖。
- 软目标：风格、问题长度、普通任务占比和整体数量。
- 资源上限：token、金额、模型请求、人工工时/题数、每蓝图修复次数。

采样器先检查可行来源数，不能在来源不足时让同一事实换几十种问法补齐“独立题”数量。配额不可达时报告缺口，不擅自放宽硬约束或将 unknown 当目标类别。

### 6.3 按缺口调度

循环过程：

1. 统计已通过自动质检/已人工审定的有效数量，两套配额进度分开。
2. 按目标缺口、业务优先级、来源多样性和预计通过成本选择切片。
3. 在切片内抽取未超上限、依赖完整的蓝图。
4. 为蓝图分配 query style/length 等表达参数。
5. 记录抽样权重/来源；成功样本入池，失败原因反馈可行性统计。
6. 到达预算、重试上限或来源耗尽时停止，并输出缺口原因。

可用历史通过率估算下一轮候选数，但每批设置上限，避免低通过率导致无限扩张。最终若需要代表真实用户分布，另提供预先定义的样本权重；面向压力测试的均衡题库不能未经加权声称真实流量准确率。

### 6.4 先分组，再扩写

先根据文档族、原始线上 thread、蓝图/对照关系建立 provenance group。跨文档蓝图形成关联组，相关样本全部进入同一 split。

生成表达变体和最小修改对照之前分配 split；同一 base_case、conversation、contrast_pair 的变体继承其 split_group。生成后再进行跨集合近重复检查。

大量跨文档连接可能把整个语料合成一个巨大连通分组。处理方式是重新限制跨组蓝图或声明独立跨文档评测子集，不任意拆边假装独立。确需随机题目留出时，明确其独立性限制。

### 6.5 区分问题留出与语料留出

- 问题留出：同一固定语料库，测试未见过的 query。源文进入检索库是正常设置。
- 语料/领域留出：用不同文档族或领域测泛化，另指定各阶段可访问语料。

两者都禁止把生成 QA、reference、测试提示或人工标签加入待测索引/训练。judge 校准与 prompt 调整使用独立 dev 材料；最终 test 标签访问有记录。

## 7. 问题合成与表达变体

### 7.1 模型调用的输入输出契约

生成器输入蓝图、已限定的证据包、用户场景和输出 schema。返回结构化 QuestionCandidate：

- query/history。
- 自身识别的目标实体、期间、条件、答案类型。
- 使用的 evidence/fact IDs。
- 风格、长度、任务类型与蓝图关联。
- 需要澄清的点或不能生成的原因。

生成器不能自行添加不存在的来源、数值、实体关系或权限。程序将其声明与蓝图逐项比较；“自己说没有问题”不是质检结论。

证据包有预算清单，包含必要标题、单位、条件和图像引用。模型上下文超限时应重新选择蓝图或拆任务，不截掉前提继续合成。

### 7.2 Prompt 模板契约

| 调用 | 关键输入 | 要求输出 | 禁止行为 |
|---|---|---|---|
| 事实抽取 | 原文锚点及定位 | 事实、限定、直接来源、疑点 | 用常识补成确定事实 |
| 问题生成 | 蓝图、证据、用户表达参数 | query/history、约束抽取、关联 ID | 扩大问题范围、把参考答案写进问句 |
| 独立求解 | query、证据、允许规则 | 主张、支持、运算、可答性 | 将生成器草稿答案当输入 |
| 验证 | 候选输入、证据、参考及规则 | 通过/退回/不确定、错误码、定位 | 以模型身份或多数意见替代原文 |
| 表达改写 | 原 query 和保留约束 | 新问法、约束对照 | 改实体、期间、否定、数量或行为 |

各模板采用中性指令和结构化 schema，要求在信息不足时明确返回 unknown/needs_review。保存模型版本、完整输入引用、输出、usage 和短判据，不依赖隐藏思维链。

### 7.3 约束一致性

对生成问题重新抽取约束，与蓝图比较：

- 实体/别名是否一致。
- 年份、季度、as_of、比较对象是否齐全。
- 实际/预算、同比/环比、百分比/百分点是否正确。
- “所有”“至少”“仅”“除外”等量词是否改变答案集合。
- 是否引入新的因果问题、开放解释或未允许工具需求。
- 是否意外把答案、原文长句或内部 source ID 写入输入。

问题可以采用业务自然缩写，但必须能从用户输入/会话/允许知识范围唯一消歧。反查不到明确范围的题要么改写补充条件，要么转成预期澄清题。

### 7.4 多轮与变体

多轮样本构造完整用户可见历史。先定义 referent，再在后续问题省略或替换为代词，记录当前轮需要从历史保留的槽位。

history 中不能预先透露当前要测的答案；确需利用先前已公开答案的任务，明确它测的是会话利用而不是检索。不得将带 reference 的内部工作消息混入 user history。

同蓝图不同问法为 related variants，不能按独立题扩大统计样本量。有限错别字不能破坏金额、年份或必要实体，除非任务就是测试澄清能力。对照变体保留 shared_base_id 和 pair_role，用于成对评估。

## 8. 独立求解、参考答案与推导规则

### 8.1 独立求解协议

问题生成后，由另一条求解调用根据 query/history、证据包和允许规则重新构造答案，不接收生成器的草稿答案。

求解器输出：

- interpreted_constraints：对问题条件的理解。
- answerability 与原因。
- claims、required_answer_claim_ids。
- 每条主张的直接证据或 derived_from。
- 充分证据组及验证规则。
- reference_answers：若干等价表达，不要求唯一文风。
- unresolved_items：无法确认的缺口。

模型异构优先用于发现分歧，不称为统计独立。证据包充分性通过后，必要时另做全授权库检索核验替代路径或冲突；不能让求解器自行访问未记录资料。

### 8.2 必需答点与支持前提分离

必需答点由用户实际问题决定，不把所有支持前提都列为答案必须复述的内容。

例如问“超额完成多少”，必需答案是差值；计划和实际数值是计算前提，可作为解释主张。若问“列出计划、实际及差值”，三者才都是必需答点。

枚举题明确集合范围、包含/排除条件和是否要求穷尽；无法确认完备范围时不发布成“全量枚举”gold。开放总结题定义核心事实、允许措辞和不可遗漏要点，避免以固定句子相似度取代内容正确。

### 8.3 类型化数值规则

保存十进制字符串，使用 Decimal 或等价精确类型：

- operand ID、值、单位、期间、实体、直接来源。
- operation 与操作顺序、分子/分母、零分母处理。
- 输出单位、精度、rounding mode、绝对/相对容差。
- 是否 exact_reading、estimated_chart_reading 或由原文披露精度决定的 rounding。

数值首版操作可限于 add/subtract/multiply/divide/ratio/percent_change；集合比较采用单独的类型化规则，按业务需要配置。百分比变化需明确 (actual−base)/base，百分点差则是两个百分比直接相减。币种转换若无允许汇率证据，不自动执行。

接受数值的规则可为：

`abs(pred−expected) <= max(abs_tolerance, rel_tolerance * abs(expected))`。

expected=0 时依赖绝对容差。输入本身为图像估读时，容差须反映输入区间与运算传播，由规则/人工确定，不由生成器随意放大到让答案通过。

### 8.4 冲突与参考一致性

检查事实、主张、规则和表达的一致性：

- 两个来源同一期间数值不同，先查看有效性、口径和版本优先级。
- 已有明确 supersedes 或政策优先级时按规则选取，并保留被替代来源。
- 无法裁决时题目标 conflicting；预期说明冲突或请求澄清，不能任选一个数值。
- reference_answers 必须能够由 claims/规则产生，不单独维护未经验证的自由文本。
- 修改 required claims、规则或证据组时，重新验证所有参考表达与衍生变体。

## 9. 无答案、歧义、权限与陷阱题

### 9.1 标签与预期行为分开

| 场景 | 标签/策略 | 预期行为 |
|---|---|---|
| 授权快照内存在充分证据 | answerable | answer |
| 指定范围内确认缺少必要事实 | unanswerable | abstain 或任务约定的说明 |
| 用户表达无法唯一确定对象/期间 | ambiguous | clarify |
| 证据冲突且无可用裁决规则 | conflicting | 说明冲突；action 按业务 rubric 明确 |
| 可能存在答案但尚未核实 | unknown | 不进入确定正/负金标 |
| 系统策略禁止使用对应材料 | 单独的权限/策略标签 | policy_block 或规定的受限回应 |
| 允许工具但当前静态材料不足 | 工具场景，记录工具能力与证据 | tool_required 或使用工具后 answer |

expected_action 只能使用 v2 约定枚举；“说明冲突”可采用 answer 加特殊 rubric，或者业务约定的 clarify，必须写明，不自行添加未定义 action。

### 9.2 负例生成方式

可从已核验可答题做最小修改：改变业务范围、期间、实体属性、未披露指标、错误前提或缺少必要一跳。修改之后必须重新核验全库，不能因原锚点不含答案就标负。

优先选择可审计的封闭范围负例，例如：

- 冻结报告明示某字段未披露。
- 授权结构化表中指定字段确实为空，且任务限定该表。
- 候选问题要求的期间超出明确定义的资料范围，且没有允许工具补齐。
- 构造的受控测试语料中明确删除全部相关来源；该语料使用独立 snapshot，不与自然语料分数混报。

随机替换一个实体或年份不必然导致不可答，可能在另一文件中正好有答案。只移除一个来源也不足以证明证据不足，因为可能有替代证据。

### 9.3 全库可答性核验

按当前用户授权视图建立检查任务：

1. 实体/别名、关键词、数字/期间规则检索。
2. BM25/稠密等互补候选，按可配置预算扩大检索。
3. 检查邻近章节、附表、版本链及相关页面。
4. 对候选运行支持/充分判断，保留命中、未命中、检索范围和检索器版本。
5. 若找到充分证据，转为可答或退回重写负例。
6. 对核心负例进行独立原文复核；无法充分确认范围时保持 unknown。

检索穷尽性有限，日志中的 search_no_hit 不等于证明无答案。负例的审定记录必须说明证明的是哪个范围、依据和限制，不能把有限搜索结果包装成全库事实。

### 9.4 负例验收与泄漏控制

正/负对照应在句式、长度、业务主题上尽量相近，避免全部负例带“是不是根本没有”等提示。错误前提题要区分“可依据原文纠正前提”与“无答案”。

权限题的输入与公开 rationale 不暴露受限答案。受保护审定记录可以保存有无授权依据，但求解器、前端报告及外部模型只接收其被授权的内容。

未知题可保留在复核池或开放探索集，不计入确定拒答准确率。发布不得将未知统一写成 is_unanswerable=true。

## 10. 图表、表格与扫描件生成

### 10.1 资产分层

每个视觉对象至少保存原始文件关联和可访问资产。已有可靠区域参考时保存 page/bbox；无法精确定位时只提供 asset/page 级，不生成虚假区域。

采用三个产物通道：

1. 原始资产：冻结页图、原图、裁剪，包含渲染和分辨率版本。
2. 观测层：OCR、单元格、轴/图例/标签、直接读数和布局关系。
3. 推导层：运算规则和结果；业务解释另置 interpreted，不作为直接观察。

资产地址失效可修访问引用，但内容摘要及源文 identity 不变。不同裁剪版本需能映射回同一冻结页。

### 10.2 图表提取细节

视觉提取输入图像、标题、邻近文本和必要页级上下文。要求模型返回：

- 图表类型、横纵轴含义、刻度/对数/截断轴、单位与图例。
- 每个被引用读数对应的实体、期间、系列、点/柱位置。
- 数值来自明确标签还是估读；无法判断时标 unreadable。
- 文本与图像有无冲突，图中脚注和范围条件。
- 原图可支持的事实列表，不输出未经核验因果解释。

对标签明确的财务图，核查“万元/亿元”“当期/累计”“百分比/百分点”等转换。视觉模型与 OCR 在数值或年份上冲突时，不能用多数票直接合成核心题。

不以宽高小于 100px 统一删除；先按装饰/潜在信息判断，过滤结果抽检。图例、公式、小标记和架构连接方向可能关键。

### 10.3 表格提取细节

参考表保留：

- 稳定 table ID、标题、页/章节范围。
- 原始及规范行列头、合并单元格的展开规则。
- 单元格原文、数值、单位、期间、缺失值类型。
- 续表连接、脚注、统计口径和单位作用范围。

“—”“不适用”“未披露”“0”是不同值，不能统一成零。跨表比较检查实体、期间、币种和统计口径兼容。公式单元格有计算值时记录公式/值来源，不在不明计算环境下擅自重算。

### 10.4 扫描件与低置信度材料

保留页级 OCR 质量、旋转、模糊、遮挡和语言混排标记。单一通道不能稳定读取的材料：

- 可生成低质量输入压力样本，但参考答案必须独立可确认。
- 原文自身不可读且无法核验，不发布确定答案的核心样本。
- 选择退回、补更高分辨率源文，或明确为 unknown。
- 审定时显示原图与提取值，不只看自动 caption。

### 10.5 视觉依赖与替代通道

记录 source_modality 与 vision_dependency_assessment。后者说明答案是否需要读取原始视觉信息，依据包括图表独有事实及授权库中有无文本替代。

图表事实被正确结构化成文本后，文本 RAG 合法作答；这不否认其原始来源是图表。image-only/text-only/hybrid 是实验条件，不在基准生成时根据某个模型答不对就强行修改标签。

页面/区域标注需单独审核，不能因答案数值通过就自动认定 bbox 合格。没有区域审定的题仍可用于答案/asset 级引用评价。

## 11. 自动质检引擎

### 11.1 ValidationResult

每项检查独立输出：

- check_id/version、candidate_id/revision、input dependency refs。
- status：pass/fail/unknown/error。
- severity：blocker/review/advisory。
- reason_code、短依据、evidence/claim/locator refs。
- checker 类型、模型/规则版本、usage、时间与 attempt。

fail 是候选不满足该项；error 是检查程序执行失败，两者不可混为质量结论。checker 的自报概率只作为诊断原始输出，不直接作为晋级依据。

### 11.2 检查项

| 检查 | 具体规则 | 失败处理 |
|---|---|---|
| schema/reference | ID 唯一、引用存在、字段类型/枚举、snapshot 一致 | blocker，修复后重查 |
| source_locator | quote/区域可定位且源文 revision 一致 | unknown 复核；确认错误退回 |
| source_fact | 实体、值、期间、单位、条件与原文一致 | 核心错误 blocker |
| query_alignment | query/history 约束与蓝图一致 | 重写或转歧义题，不临时补假设 |
| support | 每个主张有直接来源或合法派生 | 不支持则退回 |
| arithmetic | Decimal 运算、单位、顺序、容差、零分母 | fail blocker |
| sufficiency | 至少一套合法证据足以完成必要答点 | 可答题未通过则退回/复核 |
| necessity | 宣称多跳的各前提确有任务作用 | 降级任务标签或重建蓝图 |
| answerability | 全授权范围检查、工具与时间条件一致 | 不能确认保持 unknown |
| ambiguity/conflict | 指代、实体、期间与版本冲突 | clarify/conflicting 或重写 |
| leakage | 输入未含内部答案、标签、source ID 泄露 | blocker 或明确变更任务类型 |
| duplicate/split | 重复簇、关联变体、来源组未跨 split | 调整草稿分组，不能忽略 |
| presentation | 自然表达、长度、引用渲染 | 可修复，保留 revision |
| access | 生成/求解/发布只访问允许材料 | blocker，不以匿名化名字掩盖越权 |

质检顺序先 schema/权限/规则，再运行昂贵模型检查。多个检查可并行执行但消费同一不可变候选 revision。

### 11.3 独立验证与分歧

可以采用独立求解结果与候选参考比较，但要逐项比实体、数值、必要答点和证据，而非只比较整句向量相似度。

结果处理：

- 同答案、不同来源：检查是否有效替代；能成立则补组，不能成立则保留分歧。
- 不同数值但可单位换算：规则规范化后比较。
- 同数值、不同期间/实体：仍为事实不一致。
- 一个可答、一个不可答：回查证据范围和缺口，进入复核。
- 多模型都通过但抽检发现错误：定位共享提取/标签问题，对相关 slice 扩查，不能只删除被发现的一题。

所有检查通过只可晋级 auto_validated，不自动成为人工核心。若规则能确定某些字段，保留 deterministic_checked 依据，但整个题目的自然语义和范围仍需其对应核验。

### 11.4 失败修复

按 reason_code 分路由：

- schema/格式错误：允许有限结构修复，原始响应保留。
- 问句范围错误：保持蓝图重写 query，再重新抽约束和求解。
- 原始读数/关系错误：修正证据或退回源文，不只改最终答案文本。
- 支持不足：重建证据组或重新定义题目；不能加入模型常识补齐。
- 不可确认负例：转 unknown 或缩小为明确限定材料的题，并生成新 revision。
- 达上限仍失败：reject，保留原因，不无限重试筛出一个“看起来通过”的响应。

构建配置固定每阶段最大 attempt；修复样本与一次通过样本分层报告，用于观察构建偏差。

## 12. 去重、泄漏与样本独立性

### 12.1 三层去重

1. 精确：固定文本归一化后的 query/history 签名。
2. 语义：embedding/词面候选召回，人工或核验器判断是否同意图、条件与答案。
3. 证据/蓝图：相同事实集合、操作与约束的多种问法归为 variants，不冒充独立任务。

近重复阈值需在自身样本上校准。年份、否定、范围变化可能是重要对照，不能仅因为相似度高删除。对照样本以 contrast_pair_id 区分，组内保留、组间防重复。

### 12.2 跨 split 检查

检查 query、history、来源家族、蓝图、实体依赖链、变体和线上 thread。自动发现跨 split 关联时，草稿阶段移动整个组或删除新候选，记录操作；冻结版本不原地重分。

人工修改 query 后重新检查近重复；只在最初生成时做一次不足以防后续泄漏。检索库可以含任务合法源文，但不含生成的 QA 文件和审定解释。

### 12.3 与基线系统的关系

基线结果用于诊断覆盖、估计难度和发现可能的歧义/标签错误，不作为自动删题条件。

- 基线全部答对不代表题目无效。
- 全部答错不代表题目有问题，也不能直接当高难真题。
- 改进系统找到新来源时，先复核 gold 完备性。
- 在 test 结果上发现参考错误可修新版本，但报告变更原因及受影响双方，不按候选系统输赢选择性修题。

## 13. 人工审定与抽检

### 13.1 审定界面/工作包

首版可用现有标注界面或受控审定文件，每题同时呈现：

- query/history、蓝图目标、预期行为与访问/时间范围。
- 原始页面/文档位置和固定 quote。
- 观测事实、参考主张、充分证据与数值规则。
- 自动检查结果及分歧，但可将模型身份/预设答案在初次独立判读时隐藏。
- 标注项、短原因、修改建议、审定者及时间。

不只展示生成答案和 caption 让人“点通过”。权限题审定者需要相应授权，普通用户视图不暴露隐藏答案。

### 13.2 审定项目

逐项确认：原文可读、定位准确、条件完整、问题自然且无额外假设、可答性正确、必需答点合理、答案/运算正确、充分性、模态标签、负例依据、引用范围和 split 归属。

输出 approved/revise/reject/needs_adjudication，保存每项标签和依据。reviewer 不能直接改文件而无 revision 记录；改动后自动规则重跑。

首版核心集逐题审核；高影响数值/负例/冲突样本和复核分歧可安排第二人独立审定。审定人数及是否双审按项目配置，不能将单审宣称为多方一致。

### 13.3 扩展集分层抽检

将抽样分为两类：

- 风险定向：低质 OCR、复杂图、负例、多跳、修复多次、模型分歧。
- 独立随机：覆盖自动通过池，发现共同盲点。

记录抽样 stratum、纳入概率/方法和抽检数量。定向错误率不能直接外推整个生成池；需要总体估计时采用符合抽样设计的加权或单独随机样本及区间。

抽检通过不把整个批次标 human_verified。被完整审定的具体题可晋级核心，其他题仍为 synthetic_verified。

### 13.4 质检批次处置

构建前预设 slice 的可容忍错误、最低审查覆盖和处理规则，阈值来自业务而非通用数字。

发现系统性错误时：暂停该 slice 晋级 → 定位提取/模板/规则版本 → 扩大受影响范围复查 → 重跑相关候选 → 保存修复前后记录。不能只替换被抽中的坏题，使批次看起来合格。

校准生成侧 checker 时使用独立开发/测试样本，正式最终题不进入生成 prompt 或调阈值；审定误差也记录分歧率和仲裁结果。

## 14. 状态机、任务执行与成本控制

### 14.1 候选质量状态

`draft → generated → auto_validating → auto_validated → review_pending → human_reviewed → release_ready`。

辅助状态为 revise_required、rejected、quarantined。自动扩展样本可以从 auto_validated 经发布检查进入 release_ready；是否 human_reviewed 由审定记录决定，不是发布所必需的统一状态。

质量等级与流程状态分别保存：

| quality_tier | 必要条件 |
|---|---|
| human_core | 来源、问题、答案/规则、充分性及行为均按核心 rubric 审定 |
| synthetic_extension | 规定自动检查通过；未完成全项人工审定 |
| unverified | 待核验、分歧或审查不足；只进入候选池 |

case/reference 的审定标签与 tier 一致：human_core 可为 human_verified；synthetic_extension 采用 synthetic_verified。被 source_reviewed 的证据可以服务于未审定的 synthetic 题，不能据此直接提高整题等级。

### 14.2 作业状态

BuildJob 与 StageTask 独立保存 pending/running/succeeded/failed/timed_out/cancelled。作业成功表示阶段按协议完成，输出可以是 reject/unknown，不意味着样本通过。

每个 StageTask 包含 task_id、stage、input_revision/hash、config hash、依赖、attempt、lease/heartbeat、输出引用、usage 和错误类型。首版 SQLite/现有任务执行器加文件产物即可，逻辑状态机不要求通用 DAG 引擎。

### 14.3 幂等、缓存与恢复

幂等/缓存键至少包含租户访问域、snapshot、输入 artifact revision、stage、模型/模板/schema/规则版本及生成 seed/profile。

- 网络重试产生 attempt，不覆盖原始响应，全部调用成本累计。
- schema 修复和内容重生成是不同任务；后者形成候选新 revision。
- 检查器变化仅使相关 validation 失效，不必重提取源文。
- 事实修正使其 EvidenceUnit、蓝图、题目和验证的依赖链失效。
- 有效缓存可复用原始产物；模型输出非确定时，固定 seed 不保证重算相同，保留首次产物才可复查。
- worker 恢复只继续未完成任务，写入采用先临时、再校验、再提交的原子方式。

同内容 hash 不能跨租户绕过授权。日志不保存凭据；外部模型调用使用允许出域的输入视图。脱敏若改变问题所依赖事实，须保留替换规则并重新核验，不能沿用原 gold。

### 14.4 资源预算

预算分提取/视觉、问题生成、独立求解、自动核验、修复和人工审定池。记录 estimated 与 actual 分开，未知计费明确标记。

执行时按 provider 限流、并发、输入/输出 token 和图像像素控制。单蓝图、单候选、单阶段设置最大尝试；预算耗尽后停止新任务，保存已完成产物和未完成配额，不静默降低审定要求。

先用小批材料估算各切片通过率、单位有效题成本和人工耗时，再扩大。成本按通过题、发布题及各 quality_tier 分别报；不能仅按候选生成条数计算产能。

## 15. 持久化与发布

### 15.1 工作区与正式产物分离

| 记录/产物 | 主要内容 |
|---|---|
| corpus_manifest | 文档 revision、权限、有效期间、family、内容摘要 |
| artifacts/anchors/facts | 原始资产、提取物、工作范围和事实候选 |
| evidence_units/relations | 稳定支持、定位、关系及核验记录 |
| blueprints/scenario_plans | 目标、依赖、配额、split_group |
| candidates | 每次题目/答案 revision、变体和生成谱系 |
| validation_results | 各检查结果及版本、依赖、原因 |
| review_records | 独立审定、仲裁、修改和签核 |
| stage_tasks/usage | 执行、重试、成本、错误和恢复记录 |
| release manifest | 固定 case/evidence revision、schema/配置、摘要和范围 |
| test_cases.jsonl | 与 v2 兼容的正式题目 |
| evidence_units.jsonl | 唯一权威的证据内容，题目可展开读取 |
| validation summary | 来源/任务分布、通过和淘汰、缺口、未知与抽检范围 |

数据库可做查询索引，文件导出是可移植审计产物。大型图像/请求可在授权对象存储，以摘要和引用关联。

EvidenceUnit 拆表和 TestCase 展开示例是两种视图，不维护两套可独立编辑的 gold。导出器解析指定 revision，并校验展开内容一致。

### 15.2 标识与 revision

逻辑 ID 使用不可逆随机/业务命名空间，不能把受限实体明文写进公开 ID。revision 指向冻结内容；content_hash 仅做完整性/去重，不替代审定和权限判断。

case_id 表示稳定任务身份。改表达但条件/目标不变可作为同 case 新 revision 或关联 variant，策略固定；改实体、期间、预期行为形成新任务/对照身份，并记录 derived_from。fact、evidence 的编辑不在原记录上覆盖。

修正合并事实身份时保留 old→new 映射和理由；发布引用旧 revision 的数据不静默改变。

### 15.3 发布检查

release_ready 候选进入发布事务：

1. 锁定候选集合、quality tier、split 和所有依赖 revision。
2. 校验 schema、引用、权限、运算、证据充分、审定范围、去重与 split 一致性。
3. 检查每个目标切片实际数量和缺口；不合格不伪装完成。
4. 导出标准 TestCase/EvidenceUnit、manifest 和必要验证摘要。
5. 校验内容摘要和可读性，标记 frozen。生成新数据集版本 ID，发布完成前不对外标为可用。

核心集与合成集分别计数、分别评价。可在同一 manifest 管理多个 tier，但不同 tier 的标签可信度、抽样分布和门禁用途明确区分。

### 15.4 增量更新与撤回

- 新源文 revision：定位受影响 facts/evidence/blueprints/cases，复核后产生新数据集版本。
- 新替代来源：新增证据和充分组，保持原评测版本；A/B 在相同新版本上重算。
- gold 被发现错误：受影响题标 quarantined/revoked，发布修订与影响清单。
- 必须删除/撤销访问的材料：清理受管资产及其派生物，留下不含受保护内容的审计标记；旧版本可能不再可复现，明确标记，而非为“不可变”继续保留违规内容。
- 模型/模板更新：新构建 profile，旧题不因新模型偏好被覆盖。
- 单纯地址迁移：内容与 revision 一致时可更新访问层，不改变题目语义版本。

## 16. 构建配置与数据示例

以下 ID、原文、配额和金额均为结构说明，不代表真实材料已核验；预算数值不是通用推荐阈值。正式实现通过 schema 和跨记录检查约束字段。

### 16.1 BuildConfig

```json
{
  "build_id": "BUILD_DEMO_001",
  "schema_version": "2.0",
  "corpus_snapshot_id": "finance_2024q3_demo",
  "target": {
    "total_cases": 120,
    "human_core": 40,
    "synthetic_extension": 80,
    "primary_task_weights": {
      "factual": 0.35,
      "numerical": 0.25,
      "comparison": 0.2,
      "dependency_reasoning": 0.2
    },
    "max_cases_per_blueprint": 2,
    "max_cases_per_document_family": 20,
    "expected_action_targets": {
      "answer": 0.7,
      "abstain": 0.2,
      "clarify": 0.1
    }
  },
  "splits": {
    "dev": 0.25,
    "test": 0.75,
    "group_by": ["document_family", "provenance_component", "contrast_pair"],
    "seed": 42
  },
  "profiles": {
    "extractor": "extract_demo_v1",
    "generator": "generate_demo_v1",
    "solver": "solve_demo_v1",
    "validator": "validate_demo_v1",
    "annotation_rubric": "annotation_demo_v1",
    "normalizer": "text_normalizer_v1"
  },
  "allowed_operations": ["subtract"],
  "tools_policy": {"allowed_tools": []},
  "budget": {
    "provider_concurrency": 2,
    "max_stage_attempts": 2,
    "max_candidate_revisions": 3,
    "max_input_tokens_per_call": 12000,
    "max_output_tokens_per_call": 3000,
    "total_model_cost_limit": {"amount": "100", "currency": "CNY"},
    "expansion_pool_random_review_cases": 16
  },
  "release": {
    "core_requires_item_review": true,
    "allow_unknown_in_scored_gold": false,
    "on_unfilled_quota": "report_shortfall"
  }
}
```

primary_task_weights 和 expected_action_targets 是两组边际目标，task_type 与 expected_action 独立，互不相乘后强制每格满足。整数配额用固定舍入/余数分配并记录；缺来源按 shortfall 处理。human_core=40 表示目标完整审定数量，不能由扩展集抽检通过推断。

工具、模态、权限等实际覆盖还需由 profile/ScenarioPlan 明确，不能从本示例缺省推断已支持。模型配置引用在构建开始时解析并冻结，不能在执行中指向可变别名而不记录变化。

### 16.2 与 v2 兼容的展开 TestCase

```json
{
  "schema_version": "2.0",
  "case_id": "TC_FINANCE_001",
  "case_revision": "1",
  "dataset_version": "finance_core_v2_demo",
  "corpus_snapshot_id": "finance_2024q3_demo",
  "split": "test",
  "split_group": "family_A_component_1",
  "input": {
    "query": "2024年三季度云业务线营收比计划超额完成多少？",
    "history": [],
    "as_of": "2024-09-30",
    "access_scope": {
      "tenant_id": "tenant_demo",
      "roles": ["finance_reader"],
      "policy_version": "policy_demo_v1"
    }
  },
  "tags": ["zh", "numerical", "cross_evidence", "chart_sourced"],
  "reference": {
    "answerability": {
      "status": "answerable",
      "reason": "authorized_snapshot_contains_inputs",
      "label_status": "human_verified"
    },
    "expected_action": "answer",
    "reference_answers": ["超额完成2000万元。"],
    "required_answer_claim_ids": ["C_DELTA"],
    "claims": [
      {
        "claim_id": "C_PLAN",
        "text": "2024年三季度云业务线营收计划为1亿元。",
        "evidence_ids": ["E_PLAN"]
      },
      {
        "claim_id": "C_ACTUAL",
        "text": "2024年三季度云业务线实际营收为1.2亿元。",
        "evidence_ids": ["E_ACTUAL"]
      },
      {
        "claim_id": "C_DELTA",
        "text": "2024年三季度云业务线营收超额完成2000万元。",
        "derived_from": ["C_PLAN", "C_ACTUAL"],
        "verification_rule_ids": ["V_DELTA"]
      }
    ],
    "evidence_units": [
      {
        "evidence_id": "E_PLAN",
        "fact_id": "F_PLAN",
        "document_id": "plan_report",
        "document_revision": "source_plan_demo_v1",
        "modality": "text",
        "locator": {
          "kind": "text_quote",
          "scope": {"section_path": ["2024年", "第三季度", "云业务"]},
          "quote": "第三季度云业务营收计划为1亿元。",
          "prefix": "季度经营计划",
          "suffix": "执行情况将在季末披露。",
          "normalization_version": "text_normalizer_v1"
        },
        "verification_status": "source_reviewed"
      },
      {
        "evidence_id": "E_ACTUAL",
        "fact_id": "F_ACTUAL",
        "document_id": "actual_report",
        "document_revision": "source_actual_demo_v1",
        "modality": "chart",
        "locator": {
          "kind": "image_region",
          "render_revision": "actual_pdf_render_demo_v1",
          "page": 8,
          "image_asset_id": "chart_q3_demo_v1",
          "bbox_normalized": [0.1, 0.2, 0.8, 0.7],
          "observed_labels": {
            "business": "云业务线",
            "period": "2024Q3",
            "metric": "营收",
            "value": "1.2",
            "unit": "亿元"
          }
        },
        "verification_status": "source_reviewed"
      }
    ],
    "sufficient_evidence_sets": [
      {
        "set_id": "S_FINANCE_1",
        "evidence_ids": ["E_PLAN", "E_ACTUAL"],
        "allowed_rule_ids": ["V_DELTA"],
        "review_status": "reviewed_sufficient"
      }
    ],
    "verification_rules": [
      {
        "rule_id": "V_DELTA",
        "kind": "typed_numeric_expression",
        "inputs": [
          {"id": "plan", "value": "100000000", "unit": "CNY", "evidence_id": "E_PLAN"},
          {"id": "actual", "value": "120000000", "unit": "CNY", "evidence_id": "E_ACTUAL"}
        ],
        "operation": "subtract",
        "operands": ["actual", "plan"],
        "expected_value": "20000000",
        "unit": "CNY",
        "absolute_tolerance": "0"
      }
    ],
    "task_policy": {
      "rubric_version": "finance_answer_rubric_demo_v1",
      "require_supported_citations": true,
      "allowed_operations": ["subtract"],
      "allowed_tools": []
    }
  },
  "annotation": {
    "origin": "synthetic_then_reviewed",
    "quality_tier": "human_core",
    "generator_version": "generate_demo_v1",
    "annotation_rubric_version": "annotation_demo_v1",
    "relevance_annotation_scope": "partial",
    "reviewed_scope": {
      "snapshot_id": "finance_2024q3_demo",
      "document_ids": ["plan_report", "actual_report"]
    }
  }
}
```

工作库还保存 candidate_revision、evidence_revision、review/validation refs。导出时将它们锁入版本清单，不强行向标准 TestCase 增加不兼容字段。题中数值只是参考结构，human_verified 需要真实审定记录，不能由导出器根据模型得分生成。

### 16.3 自动核验记录

```json
{
  "validation_id": "VAL_DEMO_001",
  "build_id": "BUILD_DEMO_001",
  "candidate_id": "CAND_FINANCE_001",
  "candidate_revision": "1",
  "check_id": "typed_numeric_rule",
  "check_version": "1",
  "status": "pass",
  "severity": "blocker",
  "reason_code": "numeric_derivation_consistent",
  "evidence_refs": ["E_PLAN", "E_ACTUAL"],
  "claim_refs": ["C_DELTA"],
  "verification_rule_refs": ["V_DELTA"],
  "checker": {"kind": "deterministic", "profile": "validate_demo_v1"},
  "dependency_refs": {
    "snapshot_id": "finance_2024q3_demo",
    "blueprint_id": "BP_FINANCE_001",
    "blueprint_revision": "1"
  },
  "details": {
    "actual": "120000000",
    "plan": "100000000",
    "result": "20000000",
    "unit": "CNY"
  }
}
```

这只表示输入规则与推导一致，不能单独证明源文读数、query 条件、负例可答性、人工审定或整题质量通过。每项必要检查单独有记录，发布时汇总。

### 16.4 版本清单关键字段

```json
{
  "dataset_version": "finance_core_v2_demo",
  "schema_version": "2.0",
  "status": "frozen",
  "build_id": "BUILD_DEMO_001",
  "corpus_snapshot_id": "finance_2024q3_demo",
  "case_refs": [
    {"case_id": "TC_FINANCE_001", "case_revision": "1"}
  ],
  "evidence_refs": [
    {"evidence_id": "E_PLAN", "revision": "1"},
    {"evidence_id": "E_ACTUAL", "revision": "1"}
  ],
  "tier_counts": {"human_core": 1, "synthetic_extension": 0},
  "split_counts": {"dev": 0, "test": 1},
  "requested_total_cases": 120,
  "published_total_cases": 1,
  "publication_scope": "demo_subset",
  "shortfall_reason": "schema_example_only",
  "release_validation_ref": "RELEASE_CHECK_DEMO_001"
}
```

这是一个单题演示子集，不满足配置中的整批覆盖目标；实际正式发布还必须包含内容摘要、配置冻结引用、来源范围及必要审定记录。示例不伪造 SHA256 或真实 reviewer；实现缺这些字段时禁止冒充正式签核发布。

## 17. 单题生成全过程

以“云业务三季度营收比计划超额多少”为例：

1. **冻结来源**：计划报告与实际财报进入同一 snapshot，确认用户可访问，两者期间和口径可比较。
2. **提取资产**：正文保留计划原句；财报图保留原图、标题、期间、业务系列和单位。
3. **建立事实**：F_PLAN=计划100000000 CNY，F_ACTUAL=实际120000000 CNY，限定均为云业务线、2024Q3。
4. **绑定证据**：E_PLAN 使用固定 quote；E_ACTUAL 使用图像 asset/page/区域及观测标签，回原图核验。
5. **建立蓝图**：目标为超额金额；前提为两个独立事实，操作 actual−plan，不引入百分比口径。
6. **分组采样**：来源文档族、该蓝图、变体与正负对照归入同一 split_group，按缺口分配一个 numerical 场景。
7. **生成问题**：产出问句；重新抽取条件，确认期间、业务和“比计划超额”的方向都正确。
8. **独立求解**：求解器未看草稿答案，从证据输出 C_DELTA 与 20000000 CNY；参考表达为2000万元。
9. **验证**：单位换算、Decimal 运算、来源读数、必要答点、充分性和引用支持分别检查。
10. **处理替代**：检查授权库是否有直接披露差值的汇总表；若发现则加入已审定替代组，不强制系统必须两跳。
11. **去重**：同条件同运算的问法归关联变体，不能跨 dev/test；不因表述差异算多个独立事实。
12. **审定发布**：人工看原文及图表确认后进入 human_core；否则按实际验证级别进入扩展/候选集，冻结依赖与版本。

对应失败分支：

| 发现 | 处理 |
|---|---|
| OCR 把2024读成2023 | 回原文纠正事实/证据，所有依赖任务重验 |
| “实际”图表是累计营收，计划是单季 | 口径不兼容，不能直接减法；补证据或拒绝该蓝图 |
| 图中没有2000万元这个字符串 | 不因此拒绝；检查输入与运算是否成立 |
| 模型问成“超额百分之多少” | query 与蓝图不一致，重写或新建百分比蓝图 |
| 汇总表给出相同差值 | 审定后作为替代充分组，保留原组 |
| 其中一份材料无权访问 | 不向当前场景泄露答案；调整合法来源或形成政策场景 |
| 不同模型得出不同值 | 不多数票，查单位、期间、读数和运算 |
| 原文无法辨认实际数值 | unknown/补源文，不生成确定数值金标 |

如果从该题构造“2025Q1超额多少”的负例，必须重新检查当前授权快照是否包含2025Q1材料。若有答案则不能标负；没有命中但无法确认范围，则进入 unknown。正负对照维持相同风格并共享来源分组。

## 18. 构建质量统计与发布验收

### 18.1 必须输出的构建统计

| 类别 | 统计 |
|---|---|
| 来源 | 文档族/格式/语言/期间数量、成功/部分/失败提取、难版式覆盖 |
| 事实证据 | 候选/核验/审定数量，定位歧义、实体/单位/期间冲突 |
| 合成 | 每任务/模态/行为切片的计划、候选、通过、发布与缺口 |
| 质检 | 各 check 的适用、通过、失败、未知、执行错误和原因 |
| 人工 | 全审/定向/随机抽检分开，分歧/仲裁、修改和退回原因 |
| 独立性 | 文档组、蓝图组、变体与对照数量，近重复和跨 split 处理 |
| 标签范围 | human_core/synthetic_extension/unverified，可答性未知和 partial qrels |
| 成本 | 各阶段调用/重试、模型费用、人工工作量、每发布题成本 |
| 发布 | case/evidence revision、版本摘要、撤回/修订、实际用途边界 |

质检适用性先按题型确定。非数值题不强行运行减法检查；不适用检查记录为跳过及原因，不计为通过。适用检查返回 pass/fail/unknown/error，分母与执行成功率分别统计。

通过率按不同层级定义：候选自动通过率、核心审定通过率、发布率不能共用一个“准确率”。抽检发现错误时显示抽样范围与不确定性；未知不自动纳入“正确”分子。

### 18.2 核心引用和逻辑校验

发布校验器至少保证：

1. 同版本内 case、evidence、claim、rule、set 的身份与引用合法，展开视图与权威记录一致。
2. 派生主张无循环，所有前提有来源；规则操作/单位/期间合法。
3. 可答题的必需答点有支持，声称充分的组具有与质量等级相符的核验记录。
4. human_core 的必要人工项目完整；自动/抽检结果不批量伪装人工签核。
5. unanswerable 有范围和核验依据；unknown 不进入确定拒答金标。
6. 衍生变体、正负对照、同源 provenance group 不跨 dev/test。
7. 题库文件不进入待测索引；内部 reference 不出现在普通用户输入。
8. 源文/图像可按权限访问；引用失效、受限和不可读有明确状态。
9. 所有发布对象和配置冻结 revision，内容摘要可校验。
10. 目标缺口、标注不完备和不支持模态被公开记录，不以静默筛除掩盖。

### 18.3 行为验收

使用人工构造的少量明确案例验证：

- 同一事实不同 chunk 粒度不会改变 gold。
- 同数值不同年份/业务线被区分，否定条件不会被模糊匹配吞掉。
- 表格“未披露”不被转换成0，图表坐标单位不丢失。
- 正确计算无需答案字符串存在于原文。
- 多跳缺前提会被退回，存在单证据替代时不强认“两跳必需”。
- 同问法但改否定的对照不会被误删为重复。
- 负例在全库发现答案时被纠正，搜索不确定仍保留unknown。
- 改问题条件后，求解与验证缓存失效；改验证器无需重提取全部源文。
- 中断恢复不会重复发布/静默覆盖，失败调用成本仍被计入。
- 修改 gold 后产生新版本，旧版实验的参考不会悄悄改变。

数据集生成完成的判据是“来源、题目、答案与核验链条闭合，质量等级及覆盖边界明确”。是否达到某个业务发布门槛，由后续评测协议结合人工质量、任务覆盖和统计条件判断，不由生成模型自己宣布。
