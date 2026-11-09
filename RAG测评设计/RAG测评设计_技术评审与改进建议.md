# RAG 测评设计：技术评审与改进建议

评审对象：`RAG测评设计.md`。资料核对日期：2026-11-06。

假设目标：企业知识库 RAG，支持组件横向比较、富文本与多模态、离线回归和线上监控。本文逐项审阅原设计，并区分已公开的方法与针对本项目的工程建议；论文结果不等于已在你的业务上验证。未宣称穷尽截至该日期的全部论文。

## 1. 总体判断与优先级

全链路拆分、标准接口、有参考/无参考双轨、原子主张检查、共模偏差意识均值得保留。但目前更接近“指标与流程清单”，距离可信工业平台，主要差在证据定义、受控实验、裁判校准和版本治理。

| 原文位置 | 主要问题 | 推荐修改 | 优先级 |
|---|---|---|---|
| 3.1、3.3 黄金标注 | 黄金 chunk ID 随切块变化，尺子与被测对象一起改变 | gold 绑定不可变原文、页/区域/单元格；chunk 只作运行映射 | P0 |
| 3、4 合成质检 | 多模型一致、OCR 匹配均不能建立真值 | 区分自动验证候选与人工审定集；独立原文核验、数值验证、裁判元评测 | P0 |
| 2、5 指标与调度 | 正确、忠实、相关、充分混用；缺字段时可测性不清 | 每项指标声明依赖、分母、适用范围；不可测时说明原因 | P0 |
| 5.2 根因树 | 把观测相关性当成确定根因 | 先输出故障假设，用 oracle evidence 和阶段回放验证 | P0 |
| 2.5 检索 | 按 chunk 数比较会偏向更碎、更多重叠的切块 | 固定上下文预算，按去重事实/充分证据组评价 | P0 |
| 全文 | 缺版本、统计、人工校准与回流机制 | 冻结实验 manifest；配对比较、置信区间、重要切片门禁 | P0 |
| 3.2 多模态 | caption 成为唯一事实通道，URL 不证明支持关系 | 原图与结构化结果并存；评价检索、读数、区域 grounding 和回答 | P1 |
| 2.7 工程/安全 | Token 和 PII 过滤率不足以代表服务质量 | 尾延迟、失败、更新时效、权限边界、误伤和成功任务成本 | P1 |

最值得引入的方向是：**稳定证据单元、证据充分性、主张级诊断、受控回放、视觉区域 grounding、人工校准的自动裁判**。首版无需堆几十种指标。

## 2. 架构：保留黑盒接口，增加证据谱系与执行回放

### 2.1 黑盒实现与可观测契约应分开

标准输入输出合理，但框架不能从最终答案无侵入地还原全部中间状态。建议明确三种接入能力：

1. `E2EAdapter`：query → answer，仅保证端到端比较。
2. `TraceAdapter`：捕获实际执行阶段、候选、排序、最终上下文、错误和计量。
3. `ReplayAdapter`：允许注入某个阶段的输入，支持控制变量实验。

接入方声明 capabilities。原始召回、重排后结果、最终送入模型的内容必须分别记录，不能把“检索到过”视为“模型看到了”。解析/切块等离线产物通过 artifact lineage 关联请求，不要求每次查询重新构建。

固定七阶段可作为报告视图，执行模型应允许图结构、并行分支、检索循环和多次工具调用。每轮检索单独记录 span 与 attempt；不要把多轮合并成一个 Context 后评价。

| 核心对象 | 职责 |
|---|---|
| CorpusSnapshot | 原始文件版本、有效时间、访问范围、不可变内容摘要 |
| SourceArtifact | 原始页/图/表、稳定位置、解析版本及回到原文的映射 |
| TestCase | query、会话、约束、必需主张、证据、可答性 |
| RunManifest | 系统配置、代码、索引、模型/prompt、预算及实验环境版本 |
| Trace/Span | 每轮路由、检索、重排、上下文、生成、重试、错误、计量 |
| Judgment | 裁判版本、分数、状态、依据与支持位置 |
| ExperimentComparison | 配对差异、切片、置信区间和门禁 |

工业参考：LangSmith 明确区分离线 dataset/examples/experiments 与线上 runs/threads；每个 experiment 保存 outputs、scores 和 traces，支持数据集版本锁定、人工队列与线上问题回流。[8] FlashRAG 可借鉴组件替换、配置实验与中间产物，但它不是完整生产治理平台。[7]

自部署可借鉴 **Arize Phoenix** 的 OpenTelemetry/OpenInference tracing、版本化数据集和实验记录；轻量 CI 执行可考虑 **DeepEval** 的 component-level instrumentation 与 pytest/CI 集成。[12][13] 这些工具能复用记录与执行能力，但 traced LLM call 的 replay 不等于任意阶段或整个 RAG 的受控回放，后者仍需 ReplayAdapter。不要同时接入多个功能重叠的实验平台，先选一个主存储和 trace 契约。

## 3. 七阶段逐项修改

### 3.1 解析清洗：增加信息保真与独立参考

| 原方案 | 建议 |
|---|---|
| Layout Accuracy | 拆区域检测/分类、标题层级、阅读顺序，明确匹配规则 |
| TEDS | 表格结构与单元格内容分开；复杂 rowspan/colspan 保留 HTML/单元格结构，不能只依靠 Markdown |
| CER/WER | 按语言、扫描质量、版式切片；另外核验金额、年份、负号、单位等关键字段 |
| 噪声剔除率 | 同时报去噪 precision/recall 和有效内容误删率，防止“全部删除”取得高去噪召回 |
| 缺失项目 | 跨页表连接、表头继承、图文/脚注关联、原文定位可用率 |

**不要用待测 parser 的输出生成全部 gold。** 被漏掉的内容会从题目里消失，错误也可能被写成参考答案。准备一个从原始页面独立核验的小规模解析子集，专门测这些盲区。

OmniDocBench 提供文本、表格、公式、阅读顺序及属性切片。其 2026 年 MGAM 更新强调保持 GT 固定，仅在 prediction 侧适应粒度；这个原则也适合跨切块评价。固定数据和 evaluator 版本，不能跨版本直接比较榜单分数。[1]

### 3.2 切块：将语义完整度降为诊断，主测预算内的证据恢复

语法边界、指代完整和重叠率有价值，但不能独立代表 RAG 质量。单句切块可能“完整”却丢条件；大量重复可抬高 chunk 召回。

建议比较固定长度、结构切块、父子块、语义切块等方案，在相同原始语料和上下文预算下测：

- 去重必要事实覆盖率。
- 是否完整保留至少一套足以作答的证据组合。
- 否定、单位、表头、前提被跨边界切断的比例。
- 达到一定覆盖所需 token、索引存储和延迟。

相同 K 与相同 token 预算回答不同问题，应分别报告。跨切块优先固定 token/页面预算；比较同一切块下的 reranker 可以固定候选集和 K。语义切块、标题增强、父子块或视觉页面检索都应作为候选，不预设“更新就更好”。

### 3.3 Embedding 与索引：把语义质量、ANN 和量化分开

STS 和聚类纯度可作模型画像，不宜作为业务检索主 KPI。query→evidence 是非对称任务，同主题聚在一起不保证找对年份、实体和条件。MTEB/MMTEB 用于外部筛选，最终按你的领域 query/qrels 验证。[4]

建议三组实验：

1. **语义检索质量**：领域 Recall/nDCG/证据覆盖，按中英文、数字、缩写、长文、多跳切片。
2. **ANN 搜索保真**：固定 embedding、相似度与过滤后的候选全集，对比 exact top-K。
3. **量化影响**：固定搜索条件，对比原精度与压缩向量，测近邻变化、业务质量、内存、构建时间和尾延迟。

术语需改：**HNSW 本身不是量化算法，IVF 也不必量化；PQ 才是压缩方法之一。** Faiss 明确以 Flat 作为 exact baseline，并讨论速度、内存、准确性的权衡。[5]

`ANNRecall@K = |ANN_K ∩ Exact_K| / K` 只表示复现当前 embedding 近邻的程度，不表示结果与问题相关。“通常要求损耗 <2%”不应写成通用标准，应按业务 SLO 和资源约束设定，并明确百分点还是相对下降。

### 3.4 Query 重写与路由：评价保约束和实际收益

原文“代词消歧召回率（Precision）”中英文不一致。路由类别不平衡时补 macro-F1 和混淆矩阵；并行使用知识库与工具的系统不能简单按单标签评价。

增加：

- **约束保留**：实体、时间、否定、数量、用户范围、租户权限是否保留。
- **重写实际收益**：相同 query，配对比较 rewrite on/off 的覆盖、正确率、成本。
- **路由决策效用**：相对可行路线，选择造成多少质量、延迟和成本损失。
- **多跳依赖验证**：移除某个必要证据后，剩余内容是否仍足以作答。

用相似向量选两段不能证明是真多跳。HyDE、多查询、分解检索、图检索、检索式 agent 都作为可选配置做消融，框架不绑定一种策略。

### 3.5 检索、重排、压缩：分开命中、覆盖、充分和预算

《Sufficient Context》提出按上下文是否足够作答来分层分析错误，并探索据此拒答；这比只看相关性更接近本框架的诊断需求。其 2025 修订版中观察到的模型差异依赖当时的模型与数据，不能当成所有新模型的固定性质。建议引入经过校准的充分性判断，并同时评价回答覆盖率和错误率。[10]

原文 Recall@K/Hit Rate@K 不能合用一个定义：

- `Hit@K`：是否至少命中一个已标相关证据。
- `Recall@K`：已标相关证据被找回的比例，需说明标注范围与去重。
- `MRR@K`：首个相关结果的位置，不证明多跳证据齐全。
- `nDCG@K`：多级版本需要相关度标签；只有 gold IDs 可算二元版本，不能声称已衡量多级相关性。

| 建议指标 | 定义与作用 |
|---|---|
| EvidenceCoverage@B | B 上下文预算内覆盖的去重必要事实比例 |
| CompleteEvidenceSet@B | 是否覆盖至少一组充分证据；组内 AND，替代组间 OR |
| ContextSufficiency | 在允许推理/工具规则下是否足以支持完整回答，与主题相关分开 |
| RerankRetention | 候选中的必要证据在排序截断后保留多少 |
| CompressionRetention | 压缩后必要事实、条件、数值、来源定位的保留情况 |
| NoiseSensitivity | 固定正确证据和预算，改变无关/冲突材料后的表现差异 |

“相关信息 token 占比”需要对应的细粒度标注，不能把 chunk judge 分数直接宣称为 token 比率。首版可以先测证据单元级 precision。

Lost-in-the-middle 更接近生成器的证据利用能力。将相同证据置于顶部/中部/尾部，控制总长度和干扰量，多次排列，评价最终答案和主张覆盖。

若 qrels 不完整，不要把未标注结果自动当无关；汇集多个检索系统的候选补标，并显示已审定范围。否则检索器找到有效替代证据也会被罚。

RAGChecker 已有 claim recall、context precision、context utilization、noise sensitivity 和人工偏好元评测，可借鉴其主张级诊断。[6]

### 3.6 生成与引用：忠实度不是真实性

至少分开：

| 维度 | 核心问题 |
|---|---|
| Correctness | 关键事实、数字、条件和任务结果是否正确 |
| Completeness | 必需答点是否齐全 |
| Faithfulness/Groundedness | 可验证主张是否受实际输入上下文支持 |
| Citation support | 该主张标出的具体引用是否足以支持它 |
| Citation coverage | 应引用的主张有多少获得有效引用 |
| Task compliance | 是否符合用户格式、范围和业务策略 |

Ragas 官方 faithfulness 的分母是答案中的主张，判定这些主张是否能由上下文推导；它不直接衡量遗漏答点或语料外真实性。[14]

faithfulness 不等于自洽性或外部真实性：上下文错误时，忠实复述仍会错；凭记忆答对但无证据，也不能在强证据任务里算完整成功。

建议判定 `supported / contradicted / insufficient / not_verifiable`，保存支持位置。允许多条证据联合支持一个主张；数字题用类型化数值、单位、期间和确定运算验证，不要求推导结论逐字出现。

同时测输出主张 precision 与必需主张 recall，防止短而空泛的回答拿高分。空答案/纯拒答没有可验证主张时，忠实度标不适用，再独立评价拒答；不能自动给 1 分。

引用支持允许联合引用；定位需可打开、版本正确、有权限。仅存在 `[1]` 是格式检查。URL 匹配也不是支持关系判定。

拒答改为选择性回答：同时看可答题误拒率、不可答题错误作答率、回答覆盖率、risk–coverage 曲线。区分语料无答案、检索失败、证据冲突、权限不足、需澄清、需工具等，不能全压成一个 `is_unanswerable`。

### 3.7 工程与安全：与业务成功同时看

延迟报告 p50/p95/p99，明确 TTFT 起点，并把排队、检索、重排、工具、重试计入 E2E。首 token 快不代表任务完成快。

成本包含生成、embedding、reranker、图像、工具、重试，评测裁判和离线索引成本单列。增加 `cost per successful task`，成功需满足正确性和策略，不能用大量拒答降低成本。

补构建吞吐、增量更新可见延迟、删除生效、缓存冷热、超时/空检索/服务失败、并发下质量变化。不同实验说明缓存与负载条件。

安全维度覆盖访问权限、检索过滤、跨租户隔离、受保护字段泄露及文档指令是否改变系统行为。安全测试采用隔离环境与无害占位数据，同时测正常请求误伤。电话、内网地址是否敏感依业务策略定义，不能以正则脱敏率代表完整隐私保护。

## 4. 数据集：合成扩展不等于黄金真值

### 4.1 三层数据产品

| 层级 | 作用 | 限制 |
|---|---|---|
| 人工审定核心集 | 独立原文核验，校准裁判、回归和门禁 | 可以小，但覆盖关键任务和错误，并报告误差范围 |
| 合成扩展集 | 大规模内容、难度、压力覆盖 | 跨模型通过后仍是自动验证样本 |
| 线上回流集 | 真实 query 和失败案例 | 反馈有选择偏差，需保留随机抽样及时间切分 |

若严格没有人工预算，应交付“自动验证候选基准”，公开标签不确定性，不把自动分数当唯一发布依据。自动生成规模不能替代独立真值。

规模根据目标差异、覆盖和置信区间确定，没有适用于所有业务的固定题数。LangSmith 建议由高质量人工示例起步再扩展；ARES 也明确需要人工验证样本。[8][9]

### 4.2 推荐生成流水线

```text
原始语料快照 + 独立参考材料
 → 稳定证据单元（实体、期间、条件、数字、来源位置）
 → 必需事实关系与充分证据集合
 → 按任务/模态/难度/表达方式采样
 → 问题候选 + 必需主张 + 引用 + 类型化验证规则
 → 可答性、必要证据、多跳依赖、数字单位核验
 → 跨模型分歧检查 + 人工抽检/仲裁
 → 去重 + 分组切分 + 版本冻结
```

多跳围绕实体关系、时间对比、定义—条件—结果、图表—正文设计，保存依赖。移除一跳是否造成证据不足，需要依据题目与证据关系验证，不能只凭模型最终是否答错判断。模型无上下文也答对的题仍可用，但不能据此证明检索系统实现多跳。

这里可以直接借鉴 **Ragas 当前的 Knowledge Graph + Scenario 合成**：层级节点保存结构，extractor 抽实体/关键属性，relationship builder 建关系，再按 query 类型、长度、风格生成 scenario。比只做 Evol-Instruct 更便于控制覆盖与来源；图中的同实体/相似关系仍不自动证明逻辑必要性，要叠加本设计的证据依赖检查。这种 KG 只服务基准合成，不要求被测 RAG 本身使用 GraphRAG。[11]

增加关键词、缩写、错别字、多轮指代、比较、枚举、数值计算、版本冲突、时间敏感、需澄清问题。难度结合必要事实数、跨文档跨度与基线表现，不只听生成器自评。

负例相对冻结语料、时间和权限定义。“当前锚点没答案”不表示全库不可答；做全库候选核验，高影响题独立复核，允许 unknown/needs_review。使用最小修改的可答/不可答对照，减少句式捷径。

按文档族、实体、模板或时间分组切 train/dev/test；同一原文派生题不能随机散到不同集合。调参只用 dev，保留未参与优化的 holdout。

### 4.3 gold 与 chunk 解耦

建议原文 `golden_context_ids` 改为运行附属映射，主 gold 使用：

- 文本：文档不可变版本 + 页/原始位置 + 固定 canonical 文本版本中的 span。
- 表格：页/区域、表 ID、行列头、单元格和单位。
- 图片：不可变 asset ID/摘要 + 页 + 标准化 bbox；URL 仅是访问地址。
- 推导：必需输入证据 + 允许的计算/推理规则 + 结论主张。

canonical 参考不能由每个待测 parser 各自重建。各实现输出映射回原文，映射失败单列，不静默算无关。

重复事实设 equivalence group；若 {E1,E2} 或 {E3,E4} 任一组足够，就允许替代。不要只认生成题目时碰巧选择的 chunk。

## 5. 多模态：保留原图，验证视觉依赖

原图联读前后文合理，但“禁止单图独立送审”过于绝对。主路径联合解析，同时保留 image-only/text-only/hybrid 对照，以区分识图、消歧和文字旁路。

1. 小于 100px 的规则只作启发式；小图例、公式、符号也可能关键，抽检误删。
2. Caption 与原图、OCR、结构化图表并存，保留各自来源；不让一次识图错误成为唯一事实。
3. “直接观察”“确定计算”“业务解释”分字段，推断不冒充原始事实。
4. 无标签趋势图规定容差与图像分辨率，不能要求不可能的精确读数。
5. 分测页面检索、区域 grounding、轴/图例/单位/读数、答案正确和引用支持。

区域可测 IoU/覆盖，也需考虑标注粒度和支持内容，不只设一个坐标阈值。ViDoRe V3 本身报告人工区域标注的粒度差异。[2]

**ViDoRe V3（2026）是与你的设计高度相关的新基准**：覆盖多类 query、视觉文档、检索相关性、bbox 与参考答案，含人工核验和私有 holdout。原始语料主要英法文，不能直接声称中文效果已有验证。[2] 其官方实现已区分单模型和完整 OCR→chunking→embedding→retrieve/rerank 管线，也记录时间成本。[3]

## 6. 共模偏差：重点建设裁判元评测

原文三条需直接改写：

**OCR 字符匹配不是通用真值。** 计划 1 亿元、实际 1.2 亿元，结论 2000 万元未必出现在图里；另一方面同一个数字可能属于另一年份。应核验输入、行列/实体/期间、单位，再执行允许运算检查结果。OCR 是有误差的观测通道，冲突时复核。

**“默认严重幻觉、一律扣分”会制造苛刻偏差。** 使用中性 rubric、正反例锚点、明确不足/不确定状态，判断实质支持，不追求字符绝对对齐。

**temperature=0 不保证独立、正确或完全可复现。** 异构模型可发现分歧，但共享盲点仍存在。温度和两步判决是执行策略，不是独立真值保证。

推荐：

- 结构、数值、单位、集合、链接等确定性规则优先。
- 主张级裁判保留 supported/contradicted/insufficient 和支持位置，缓存同版本提取结果。
- 用人工样本调 rubric，再在独立样本上测 judge precision/recall、混淆矩阵、人工一致性和置信区间。
- 低成本 judge 批量测，高成本/异构 judge 处理分歧和高影响案例；随机抽检发现共同盲点。
- Pairwise 打乱身份和左右顺序，测位置、文风、长度及自模型偏好；客观数字仍优先规则。
- 记录模型/prompt/rubric 版本、输入、短依据和支持位置，无需存储隐藏思维链。

ARES 的值得借鉴点是 **Prediction-Powered Inference（PPI）**：用人工标签纠正自动判定的总体统计偏差并构造区间，而非把自动分数直接当真实准确率。[9] PPI 需要满足抽样及估计条件，不能把偏向难例的人工队列不加处理地当总体代表，也不保证每个自动标签正确。首版可先人工校准+配对比较，再考虑 PPI 降成本。

## 7. 调度：从字段等级改为指标依赖图

Level 1/2/3 可作为入口，底层按每项指标的 requires 调度。只有 query 无法测 faithfulness；捕获 query+answer+真实上下文后才有支持度评价，仍不能直接证明真实正确性。

| 指标 | 最少依赖 | 缺失处理 |
|---|---|---|
| 延迟/成本/错误 | Trace 与计量 | 显示缺计量，token 不是全部成本 |
| Groundedness | answer + 实际输入证据 | missing_context |
| Correctness | answer + 审定参考/验证规则 | 不用内部一致性冒充正确 |
| Recall/nDCG | 排序 + qrels/稳定证据映射 | 区分未标注、映射错误与不适用 |
| Sufficiency | query+context+允许推理规则；可靠测量宜有标注 | judge 估计须标校准状态 |
| 拒答质量 | 行为 + 相对语料/权限/时间的可答性 | 检索不到不能直接当正确拒答 |
| 归因实验 | 回放接口与对照配置 | 仅提供故障线索 |

结果包含 value、status、reason、evaluator_version、evidence_refs。区分 not_applicable/not_computable/error/inconclusive，不能全部记零或静默删样本。报告有效样本数及覆盖率，防止失败更多的方案反而表面更高分。

## 8. 归因：用受控实验验证故障假设

原树保留作排查提示，输出改成：
`symptom → hypotheses → interventions → observed_effect → confidence`。

| 对照实验 | 固定条件 | 可检验内容 |
|---|---|---|
| 正常上下文→审定充分证据 | query、生成设置、预算口径 | 充分证据能否挽救回答；仍错还需排查生成、题意、标注 |
| 固定候选→替换/审定排序 | 候选和预算 | 排序截断损失，不能挽救候选中本就不存在的证据 |
| 原 query→审定 rewrite | 后续组件和预算 | 重写引入或修复的检索偏差 |
| parser 输出→核验标准内容 | 下游和语料范围 | 解析错误对终局影响 |
| 固定证据→位置/噪声变化 | 总长及必要证据 | 位置和干扰敏感性 |
| 正常上下文→无上下文 | query、生成设置 | 先验答题能力，不直接证明泄漏/记忆 |
| 完整证据→移除必要一跳 | 其他证据和规则 | 必要性与不足感知 |

Oracle 上下文常更短更干净，必须注明干预差异。多次运行处理随机性，允许多个故障与组件交互，不强制单根因。

## 9. 推荐数据契约示例

以下是设计示例，ID/位置为占位符，不是已经核验的业务标注。生产可拆表/JSONL。

```json
{
  "schema_version": "1.0",
  "case_id": "TC_2026_001",
  "dataset_version": "finance_eval_v1",
  "corpus_snapshot_id": "finance_2024q3_frozen",
  "split_group": "financial_report_family_A",
  "query": "2024年三季度云业务线营收相比正文计划超额完成了多少？",
  "conversation": [],
  "as_of": "2024-09-30",
  "access_scope": {"tenant_id": "demo", "roles": ["finance_reader"]},
  "tags": ["numerical", "cross_evidence", "image_dependent", "zh"],
  "answerability": {
    "in_authorized_snapshot": "answerable",
    "expected_action": "answer",
    "label_status": "human_verified"
  },
  "reference_answers": ["超额完成2000万元。"],
  "required_claims": [
    {"claim_id": "c1", "text": "计划1亿元。", "evidence_ids": ["e1"]},
    {"claim_id": "c2", "text": "实际1.2亿元。", "evidence_ids": ["e2"]},
    {"claim_id": "c3", "text": "超额2000万元。", "derived_from": ["c1", "c2"]}
  ],
  "evidence_units": [
    {
      "evidence_id": "e1",
      "document_id": "report_plan",
      "document_revision": "immutable_revision_A",
      "locator": {"page": 3, "canonical_text_revision": "reference_v1", "char_span": [100, 150]},
      "modality": "text"
    },
    {
      "evidence_id": "e2",
      "document_id": "report_actual",
      "document_revision": "immutable_revision_B",
      "locator": {"page": 8, "image_asset_id": "chart_q3_v1", "bbox_normalized": [0.1, 0.2, 0.8, 0.7]},
      "modality": "chart"
    }
  ],
  "sufficient_evidence_sets": [["e1", "e2"]],
  "verification": {
    "kind": "typed_numeric_expression",
    "inputs": [
      {"id": "plan", "value": 100000000, "unit": "CNY", "evidence_id": "e1"},
      {"id": "actual", "value": 120000000, "unit": "CNY", "evidence_id": "e2"}
    ],
    "operation": "subtract",
    "operands": ["actual", "plan"],
    "expected_value": 20000000,
    "unit": "CNY",
    "absolute_tolerance": 0
  },
  "annotation": {
    "origin": "synthetic_then_reviewed",
    "generator_version": "generator_config_v1",
    "rubric_version": "annotation_rubric_v1",
    "review_status": "approved"
  }
}
```

还需约定页码从 0/1 开始、bbox 原点与方向、文本 span 的编码与边界。运行 chunk ID、排序、token 和 gold 映射写入 Run/Trace，不反向改 gold。数值规则只运行允许的类型化操作，不执行模型生成的任意代码。

## 10. 统计、报告与实施顺序

同一用例 A/B 配对，固定数据/语料/权限、代码、索引、切块、模型/prompt、judge/rubric、上下文预算、工具与缓存/并发。比较一个组件时只改该组件，或明确记录联动。

报告绝对与相对差异、置信区间、关键切片及失败。可用 paired bootstrap；来自同文档族/会话的相关题按组重采样，避免伪独立。多配置搜索只用 dev，holdout 最终确认；不能以均分高 0.5 分就宣布有效。

线上无参考分数是抽样/代理监控，不能代表未采样请求已核验正确。无参考监控与离线审定回归形成闭环。[8]

不建议单一加权总分。主报告列任务成功、证据充分/覆盖、答案正确/完整/支持、误拒/错答、尾延迟/成本、重要切片退化、评测失败和裁判校准；先过约束，再在质量—成本—延迟之间选 Pareto 方案。

| 阶段 | 交付 | 验收 |
|---|---|---|
| P0-1 可信数据 | 稳定 locator、冻结快照、TestCase/Run/Judgment、审定种子集 | 改切块不重定义 gold；缺数据/测量错误可见 |
| P0-2 最小闭环 | Trace、预算内证据覆盖、正确/完整/支持、拒答、成本延迟 | 可回放，每个失败能关联主张和证据 |
| P0-3 可信比较 | judge 元评测、配对统计、切片、回归和版本门禁 | 能解释提升是否可信及代价 |
| P1 深度诊断 | oracle/阶段干预、视觉区域、parser 专项、权限/更新 | 故障假设有实验依据 |
| P2 前沿扩展 | agentic 轨迹、图检索、多阶段策略、PPI/裁判成本优化 | 在相同预算和可信 holdout 上证明收益 |

首版解析细项做独立插件，主链优先支持稳定证据、配对运行和主张检查。最有价值的差异化在于：**跨组件可比较、分数可审计、失败可回放、归因可验证。**

### 10.1 借鉴与自研的边界

| 能力 | 可借鉴/接入 | 仍需本框架负责 |
|---|---|---|
| 实验、版本、Trace、人工反馈 | LangSmith 等成熟平台的对象和闭环 | 原文 locator、跨组件可比性和业务门禁 |
| 测试题合成 | Ragas 的 KG/Scenario | 独立原文核验、必要证据与替代证据标注 |
| 主张级诊断 | RAGChecker | 中文/领域 rubric 校准、数值与权限规则 |
| 低成本总体质量估计 | ARES/PPI | 代表性抽样、人工标签和适用条件检验 |
| 算法复现与基线 | FlashRAG | 生产访问策略、版本治理、故障干预 |
| 解析/视觉评价 | OmniDocBench、ViDoRe/MTEB | 自有材料切片、图表真值、业务端到端验证 |

推荐形态是轻量协调层 + 明确数据契约 + 可插拔 evaluator，复用现有工具。优先自研证据谱系、可控回放、跨切块比较和业务规则，避免重建全部模型调用、实验 UI 与基础指标库。具体选择托管平台还是自部署按预算和现有技术栈决定。

## 11. 一手来源与适用边界

下列来源已在本次研究中打开核对。当前官方实现可能不同于早期论文版本，接入时应锁版本/commit。正文未把本文工程建议冒充论文原结论。

1. **OmniDocBench**：CVPR 2025，官方维护含 2026 MGAM/版本升级；文本/表格/公式/阅读顺序与属性切片。范围是解析，不能替代完整 RAG。https://github.com/opendatalab/OmniDocBench ；https://arxiv.org/abs/2412.07626
2. **ViDoRe V3**：2026-01 初稿、2026-04 修订；核对摘要与正文第 3/4 节，人工核验、区域、跨文档任务及 holdout。源文英法为主，不能直接迁移中文结论。https://arxiv.org/abs/2601.08620 ；https://arxiv.org/html/2601.08620v2
3. **ViDoRe 官方框架**：当前管线评估、自定义 index/search、复杂 OCR/混合检索、时间成本；不代表仓库实现了本文全部答案/治理功能。https://github.com/illuin-tech/vidore-benchmark
4. **MTEB/MMTEB**：官方多语言、多任务、多模态 embedding/retrieval 工具；外部选型不是领域质量保证。https://github.com/embeddings-benchmark/mteb
5. **Faiss Guidelines to choose an index**：官方 Flat exact、HNSW/IVF/PQ 与资源权衡。https://github.com/facebookresearch/faiss/wiki/Guidelines-to-choose-an-index
6. **RAGChecker**：2024，claim-level entailment、检索/生成诊断、人类偏好元评测入口。https://arxiv.org/abs/2408.08067 ；https://github.com/amazon-science/RAGChecker
7. **FlashRAG**：WWW 2025 Resource，官方模块、配置实验和中间结果。https://github.com/RUC-NLPIR/FlashRAG
8. **LangSmith Evaluation concepts**：官方 dataset/experiment/trace、版本、人工队列、pairwise、线上线下闭环及 judge review。https://docs.langchain.com/langsmith/evaluation-concepts
9. **ARES**：2023 预印本，官方合成+judge+PPI 与人工验证要求；借鉴统计校准，不等于可零人工得真值。https://arxiv.org/abs/2311.09476 ；https://github.com/stanford-futuredata/ARES

10. **Sufficient Context: A New Lens on Retrieval Augmented Generation Systems**：2024-11 预印本，2025-04 v3；核对 Google Research 官方摘要与 arXiv 页面。支持充分性分层与选择性回答；不承诺无人工 autorater 或普适模型排序。https://research.google/pubs/sufficient-context-a-new-lens-on-retrieval-augmented-generation-systems/ ；https://arxiv.org/abs/2411.06037
11. **Ragas: Testset Generation for RAG**：当前官方文档采用 Knowledge Graph/Scenario、层级节点、extractor/relationship builder 与 synthesizer，不能保证合成题天然可靠。https://docs.ragas.io/en/stable/concepts/test_data_generation/rag/

12. **Arize Phoenix 官方实现**：OpenTelemetry/OpenInference、tracing、versioned datasets、experiments 和 traced LLM call replay；后者不等于完整管线任意阶段回放。https://github.com/arize-ai/phoenix
13. **DeepEval 官方实现**：end-to-end/component-level evaluation、observe spans、pytest/CI 集成；执行框架本身不保证参考数据或 judge 可靠。https://github.com/confident-ai/deepeval
14. **Ragas Faithfulness 官方定义**：按输出主张验证上下文支持，不能独立证明完整性与真实性。https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/
