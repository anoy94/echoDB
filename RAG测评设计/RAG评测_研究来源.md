# RAG 评测设计：一手来源核查补充

核查日期：2026-11-06。本文件为主评审的研究旁证，不修改原设计。以下“已核”均实际读取官方文档或官方仓库；功能描述不是产品性能保证。

## 1. LangSmith：评测平台的数据模型与工作流

- 来源：[Evaluation concepts](https://docs.langchain.com/langsmith/evaluation-concepts)。官方活文档，无固定论文发表日期，2026-11-06 核查。
- 已核：离线 evaluation 的目标是 dataset/examples；在线目标是 production runs/threads。Experiment 保存指定应用版本在数据集上的 outputs、evaluator scores、execution traces。
- 已核：例子可包含 inputs、可选 reference outputs、metadata；reference outputs 不传给受测应用。支持人工、代码、LLM、pairwise 等评测方式。
- 已核：dataset 内容变更自动产生 version，可以 tag，并在 CI 锁定指定版本；支持人工标注队列，将线上问题加入离线集。
- 已核：官方推荐从人工整理的少量高质量样例起步，合成数据以这些例子为模板扩充。
- 对本设计的意义：采用 TestCase/DatasetVersion、Run/Trace、Experiment、EvaluatorVersion、Score/Annotation 分离的数据模型；线上到离线回流；pairwise 对照而非只比较绝对 judge 分。
- 局限：说明的是平台支持的组织方式，不证明任一 judge 在中文、多模态或业务域中的准确度。

## 2. Phoenix：追踪、实验与可重放底座

- 来源：[arize-ai/phoenix 官方仓库](https://github.com/arize-ai/phoenix)，README，2026-11-06 核查。
- 已核：基于 OpenTelemetry tracing；OpenInference 仪表化；versioned datasets；experiments 追踪并评估 prompts、LLMs、retrieval 的变化；playground 可 replay traced LLM calls。
- 对本设计的意义：复用通用 span/trace 底座和实验存储，保持离线与线上一致的阶段数据契约。阶段 span 应保存实际输入输出及其版本，才能进行观察性诊断和受控重放。
- 局限：追踪本身不构成因果归因；真实干预实验仍需框架自己定义。同一 OTel 生态也不意味着检索语义字段天然统一。

## 3. DeepEval：轻量回归执行层

- 来源：[confident-ai/deepeval 官方仓库](https://github.com/confident-ai/deepeval)，README，2026-11-06 核查。
- 已核：支持端到端与 component-level 评测、pytest/assert_test 风格和 CI 集成；@observe、update_current_span 记录组件；trace 可用于完整 trajectory 评估；RAG metrics 可单独调用。
- 对本设计的意义：可复用测试执行、阈值断言和报告机制，避免从零实现一套运行器。业务 rubric、证据标准和 judge 校准仍应由自身框架管理。
- 局限：README 中“research-backed”“human-like accuracy”等产品描述不能当成业务域已验证准确率；阈值示例不是通用上线门槛。

## 4. Ragas Faithfulness：忠实度的准确分母

- 来源：[Faithfulness 官方文档](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/)，页面标注 December 9, 2025，2026-11-06 核查。
- 已核：分数为“被 retrieved context 支持的 response claims 数 / response claims 总数”；先拆解回答主张，再检查能否从上下文推出。
- 对本设计的意义：主张级证据核对可直接复用，但必须将 groundedness 与事实正确性、答案完备性、引用正确性分开。空答或少答的分母策略需单独定义，不能靠忠实度奖励保守空答。
- 局限：判定依赖 claim extraction 与 entailment judge 的误差；上下文本身错了时，高忠实度不能证明答案符合现实。

## 5. ARES：用人工校准自动评审，报告统计不确定性

- 来源：[stanford-futuredata/ARES 官方仓库](https://github.com/stanford-futuredata/ARES)；论文 [ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems](https://arxiv.org/abs/2311.09476)，预印本 2023。此处实读官方仓库的说明与示例，不将论文会议归属作为本次已核事实。
- 已核：synthetic data、fine-tuned classifiers 与 Prediction-Powered Inference (PPI) 组合，评估 context relevance、answer faithfulness、answer relevance；利用人工验证集修正自动分数，输出统计置信区间。
- 已核：仓库要求人工标注的 query-document-answer 验证集，至少 50 例、理想数百例；示例输出包括区间和使用的人工标注数。
- 对本设计的意义：多模型互审不能替代少量独立人工金标。先对 judge 作 meta-evaluation，再考虑自动化扩容；当需要总体质量估计与区间时，可试验 PPI 路线。
- 局限：仓库建议的样本数量不是对任何域、任何分布都充分的统计保证；PPI 的取样与统计假设需遵守。它主要改善总体质量估计，不自动修复每道题标签或定位每次失败。

## 6. 与主评审互证的近期来源

以下由主评审研究同步核对，本文不重复独立实读结论：

- [Sufficient Context: A New Lens on Retrieval Augmented Generation Systems](https://research.google/pubs/sufficient-context-a-new-lens-on-retrieval-augmented-generation-systems/)，Google 官方页面标 2025，arXiv:2411.06037。用于区分上下文相关与充分，设计选择性回答/拒答实验。
- [ViDoRe V3](https://arxiv.org/abs/2601.08620)，2026。用于原生视觉证据、页面/bbox 标注及整条检索 pipeline 的可比实验。
- [RAGChecker 官方仓库](https://github.com/amazon-science/RAGChecker)。用于主张级诊断指标、context utilization 与 noise sensitivity。
- [OmniDocBench 官方仓库](https://github.com/opendatalab/OmniDocBench)。用于解析层独立基准和文档元素匹配；版本变化需锁定。

## 可直接落地的组合建议

优先复用 Phoenix/OpenTelemetry 或现有 tracing 底座、LangSmith 风格的数据集与实验模型、Ragas/DeepEval 的可替换指标执行器。自己重点建设源文级证据标注、中文与业务域的 judge 校准、同预算对照、可控重放干预、数据版本与线上回流。平台选择取决于部署环境、数据管理与技术栈，并不需要同时引入四个平台。
