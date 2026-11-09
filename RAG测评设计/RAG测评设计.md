# 全链路 RAG 自动化测评框架与基准数据集构建技术设计方案

---

## 1. 架构定位与设计哲学

### 1.1 核心设计理念
本测评框架旨在解决传统 RAG 评测“只看生成结果、无法定位归因”的痛点，遵循以下三项核心原则：
1. **模块黑盒化与接口标准解耦**：将 RAG 各阶段（解析、切块、向量化、检索、重排、生成）视为黑盒，通过标准化的输入输出契约进行数据捕获，实现各组件的无侵入式评测与横向替换比对。
2. **双轨评测范式结合**：
   * **有参考评估（Reference-based）**：基于黄金基准集（Ground Truth），计算召回率、相关性、排序质量与答案准确度。
   * **无参考评估（Reference-free）**：基于单次运行的链路数据（Query-Context-Answer），计算忠实度（Faithfulness/自洽性）与抗幻觉度。
3. **评测分级自适应（Tiered Adaptability）**：框架流程不绑定特定丰富度的数据集，能根据当前提供的数据字段（仅有 Query、包含召回证据、或包含参考答案）动态适配执行路径。

---

## 2. 全链路阶段定义与全面测评指标体系

```text
[原始数据接入] 
      ↓ 1. 数据解析与清洗 (Parsing & Cleaning)
      ↓ 2. 文本切分与分块 (Chunking)
[知识组织与索引]
      ↓ 3. 向量化与索引构建 (Embedding & Indexing)
[查询理解与重写]
      ↓ 4. 检索前处理 (Query Rewrite / Expansion / Routing)
[检索与排序]
      ↓ 5. 混合检索与重排序 (Hybrid Retrieval & Reranking & Context Pruning)
[生成与合成]
      ↓ 6. 上下文组装与答案生成 (Context Assembly, Generation & Attribution)
[系统级全局保障]
      ↓ 7. 工程效能与安全性 (Engineering & Security)
```

### 阶段 1：数据解析与清洗层（Data Parsing & Cleaning）
* **核心任务**：处理 PDF、PPT、DOCX 等富文本，还原真实版面结构，过滤冗余噪声。
* **测评指标**：
  * **版面恢复准确率（Layout Accuracy）**：标题层级（H1-H4）、正文段落边界识别的准确率。
  * **表格树编辑距离相似度（TEDS, Tree-Edit-Distance-based Similarity）**：量化评估还原的 HTML/Markdown 表格与原始表格的结构+内容保真度。
  * **字符/词级错误率（CER/WER）**：扫描件和嵌入图片 OCR 识别的错误率。
  * **噪声剔除率（Noise Reduction Rate）**：页眉、页脚、版权声明、无效控制符的过滤召回率。

### 阶段 2：文本切分与分块层（Chunking）
* **核心任务**：平衡语义原子性与完整性，避免跨句子、跨实体的生硬截断。
* **测评指标**：
  * **语义自包含度（Semantic Coherence）**：单 Chunk 在脱离上下文时代词指代不明、语义残缺的比例。
  * **语法边界完整率（Boundary Quality）**：切分点落在自然句末/段落末的比例，对代码块、表格、公式的破坏截断率。
  * **信息密度与重叠冗余比（Information Density & Overlap Ratio）**：Chunk 内核心实体 Token 密度，以及相邻 Chunk 重叠区的信息利用效能。

### 阶段 3：向量化与表征层（Embedding & Indexing）
* **核心任务**：将 Chunk 映射到高维语义空间，提供低损耗的高并发近似检索。
* **测评指标**：
  * **语义相似度相关系数（STS Spearman Correlation）**：语义相近的段落在向量空间中的余弦距离相关性。
  * **聚类纯度（Clustering Purity）**：同主题/同章节知识块在向量空间聚类的紧凑程度。
  * **索引量化损耗率（Quantization Recall Loss）**：采用 HNSW、IVF-PQ 等近似近邻检索时，相比暴力全量搜索（Flat Exact Search）的 Recall 损失（通常要求 $< 2\%$）。

### 阶段 4：检索前处理层（Query Transformation & Routing）
* **核心任务**：解决模糊查询、多轮代词消歧、多跳复合问题拆解与意图分流。
* **测评指标**：
  * **意图路由准确率（Intent Classification Accuracy / F1）**：准确路由至 RAG、直答（LLM）、工具调用或网络搜索的分类指标。
  * **代词消歧召回率（Coreference Resolution Precision）**：多轮对话改写后补全主谓宾信息的准确率。
  * **子查询覆盖度（Sub-query Coverage）**：复合多跳问题分解出的子 Query 是否完全覆盖答题所需的充分条件。
  * **语义漂移度（Semantic Drift）**：改写后的 Query 与原始 Query 核心意图的语义偏离度（避免改写模型自嗨）。

### 阶段 5：检索与重排层（Retrieval & Reranking）
* **核心任务**：多路召回（BM25 + Dense + Sparse）及 Cross-Encoder 重排，截断并过滤噪声。
* **测评指标（传统信息检索 IR）**：
  * **Recall@K / Hit Rate@K**：Top-K 检索结果中包含黄金证据块（Golden Chunks）的覆盖率。
  * **MRR@K（Mean Reciprocal Rank）**：首个相关证据文档所处排名的倒数均值。
  * **NDCG@K（Normalized Discounted Cumulative Gain）**：考虑文档相关度级别与位置加权的排序增益。
* **测评指标（RAG 专用增强）**：
  * **上下文相关性（Context Relevance）**：召回片段中真正有效信息的 Token 占比（过滤无关噪声）。
  * **上下文充分度（Context Sufficiency）**：召回的 Context 集合理论上能否闭环支撑推导出问题答案。
  * **抗位置偏置敏感度（Lost-in-the-Middle Sensitivity）**：关键证据处于上下文顶部、中部、尾部时的留存率。
  * **上下文压缩率与信息丢失比（Context Pruning Efficiency）**：提示词压缩（如 LLMLingua）前后语义损耗比率。

### 阶段 6：生成与合成层（Generation & Attribution）
* **核心任务**：严格遵照检索事实进行推理并给出答案，准确标注事实出处。
* **测评指标**：
  * **忠实度/真实性（Faithfulness / Groundedness）**：**（最核心）** 将生成答案切分为原子主张（Atomic Claims），每个主张能被检索上下文直接推导（Entailed）的比例，衡量事实幻觉。
  * **回答相关性（Answer Relevance）**：答案针对 Query 意图的契合度（杜绝答非所问）。
  * **语义准确度（Answer Semantic Correctness）**：与 Ground Truth 的事实对齐度和语义重合度。
  * **引用精准度与召回率（Citation Precision & Recall）**：
    * *Precision*：答案中附带的引用标注 `[1]` 是否精准指向提供了对应支撑的段落。
    * *Recall*：答案中所有采纳的事实主张是否都打上了正确的引用标号。
  * **负向拒答准确率（Negative Rejection Rate）**：在无相关上下文或故意植入不相关诱导提问时，模型明确回复“依据现有知识库无法解答”的准确率。

### 阶段 7：系统工程与安全指标（System & Security）
* **首字延迟（TTFT, Time To First Token）** 与端到端总延迟（E2E Latency Breakdown）。
* **Token 消耗效率（Cost per Query）**：单次请求的 Prompt Token 与 Completion Token 损耗。
* **间接提示注入防御率（Indirect Prompt Injection Defense）**：防御知识库文档内部隐藏恶意指令攻击的有效性。
* **隐私脱敏率（PII Leakage Defense）**：电话、身份证、内网地址在生成时的过滤率。

---

## 3. 自动化基准数据集（Benchmark Dataset）构建方案

由于无人工标注数据，必须采用“离线、重算力、极度严苛”的逆向合成流水线。

```text
[原始多格式文档]
       ↓ (版面分析 + 结构化清洗)
[解析后文本 + 高保真图表]
       ↓
[锚点事实提取 (Fact Extraction)] 
       ↓ (逆向生成 QA 对)
[初步测试集 (Query, Golden Context, Ground Truth)]
       ↓
[双模型对抗与交叉验证 (Cross-Model Verification)]
       ↓
[分层质检与最终基准集 (Golden Benchmark)]
```

### 3.1 逆向锚定合成流程（Anchor-based QA Generation）
1. **切分上下文锚点**：提取单一 Chunk 或高关联相邻 Chunks 作为信息锚点（Anchor Context）。
2. **原子事实提取**：指令大模型抽取锚点中确定性的事实陈述（实体、关系、数据指标）。
3. **Evol-Instruct 演进提问**：
   * **Simple（简单事实）**：单一事实点直问。
   * **Reasoning（逻辑推理）**：引入“因果分析、推导运算”的逻辑链提问。
   * **Multi-Context（跨段多跳）**：根据向量相似度拉取两块不同章节的相关 Chunk，合成跨文档才能作答的复杂问题。
   * **Negative/Trap（陷阱与空题）**：基于该主题故意生成无法依据当前知识作答的问题，用于评测系统的拒答能力。
4. **绑定黄金标注**：将该锚点 Chunk 的唯一哈希 ID 存入 `golden_context_ids`，并由模型直接引用原文生成严格一致的 `ground_truth`。

### 3.2 多模态与图片插入场景处理规范
针对 PPT、PDF、DOCX 中的复杂图表、架构图与扫描插图：
1. **两级过滤机制**：
   * *规则与尺寸过滤*：剔除宽/高小于 100px、长宽比严重失衡的图标、装饰线、作者头像。
   * *图表属性分类*：仅筛选架构图（Architecture）、趋势折线图/柱状图（Chart）、截屏表格（Table）进入深度解析通道。
2. **上下文关联式图文提取（Contextual Image Captioning）**：
   * **禁止单图独立送审**。必须将 `[图表所在页标题 + 图表前后文段落 + 图表图片本身]` 组合后喂入多模态大模型（MLLM）。
   * 产出结构化 Markdown：包含原图 OSS 链接、核心数据极值、坐标轴信息、业务核心推断。
3. **专属多模态评测用例标签**：
   * 数据集中打上 `content_type: image_dependent`。
   * 评测指标专项增加：**图片引用匹配度（Image Citation Accuracy）**，即不仅文字答对，返回的前端证据必须关联到对应的 `golden_image_url`。

### 3.3 数据集分级产物定义（Schema）

```json
{
  "test_cases": [
    {
      "case_id": "TC_2026_001",
      "synthesizer_type": "multi_hop_reasoning",
      "content_type": "image_dependent",
      "query": "2024年三季度云业务线营收相比正文计划超额完成了多少？",
      "ground_truth": "超额完成了2000万元。正文计划为1亿元，第三季度财报图表中实际营收为1.2亿元。",
      "golden_context_ids": ["doc_chunk_884", "doc_chunk_885"],
      "golden_image_urls": ["https://oss.../financial_chart_q3.png"],
      "is_unanswerable": false
    }
  ]
]
```

---

## 4. 关键技术陷阱防范：共模偏差与循环自证

### 4.1 核心风险（Shared Blind Spots）
当**数据处理/标注阶段**与**评测框架阶段**使用**同一种多模态模型（例如均为 GPT-4o 或 Qwen2.5-VL）**时：
* 如果该模型在某个图表上有固有的认知幻觉（将 2023 看错为 2024），它在生成测试集时会写错参考答案。
* RAG 运行时模型同样犯错，评测裁判打分时认为“回答与参考答案高度重合”，最终判定为“满分”。
* **表现为离线测试指标虚高，线上人类用户体感极差。**

### 4.2 防御工程方案
1. **非对称独立验证（OCR & Rule Cross-check）**：
   * 数据集生成后，引入传统 OCR 引擎（如 PaddleOCR）单独提取纯文本/数字矩阵。
   * 采用硬规则检查：生成答案中的关键数字与专有名词，必须在 OCR 结果中找到物理匹配。若无匹配，判定为模型看图幻觉，自动废弃该题目。
2. **Prompt 角色对抗设计**：
   * **生成 Prompt（保守执行者）**：“仅提取图中 100% 确认无歧义的文字/数值，严禁任何推理和脑补”。
   * **评测 Prompt（极客审计员）**：“你的任务是挑刺。默认当前答案包含严重幻觉，除非其在证据块中找到绝对字符对齐的依据，否则一律扣分”。
3. **跨温度与采样多样化隔离**：
   * 标注生成阶段采用严格的低采样（`temperature = 0.0`），规避发散。
   * 评测阶段采用双步链式判决（先提取事实清单，再逐一比对），降低一次性打分的共谋概率。

---

## 5. 测评流程执行与自适应归因引擎设计

```text
               [读取输入用例 (TestCase)]
                          ↓
          [评测调度器 (Workflow Dispatcher)]
            /             |              \
 [仅包含 Query]   [含 Golden Chunks]   [含 Golden Chunk + GT]
        ↓                 ↓                       ↓
  Level 1 管道      Level 2 管道             Level 3 管道
(端到端自洽性评测)   (独立检索召回测试)     (全链路全指标评测)
        \                 |                       /
         ───────→ [归因下钻引擎 (Root-Cause Engine)]
                          ↓
                  [生成测试审计报告]
```

### 5.1 自适应流程调度（Workflow Dispatcher）
框架自动嗅探输入数据的元数据完备性，无缝降级运行：
* **模式 A（全量基准模式 - 拥有 Level 3 数据）**：启动双并发通道，分别测试检索模块（输出 Recall@K、NDCG、MRR）与生成模块（输出 Faithfulness、Answer Correctness、Citation Metrics）。
* **模式 B（检索调试模式 - 仅提供 Level 2 数据）**：绕过生成大模型，秒级批量评估切块算法、向量检索、重排策略的召回率，支持 CI/CD 自动化流水线。
* **模式 C（在线监控模式 - 仅捕获生产 Query 与链路输出）**：计算自洽性指标（Faithfulness、Relevance、Latency、Safety）。

### 5.2 根因归因决策树（Root-Cause Attribution）
评测框架在输出打分的同时，自动运行诊断决策树，输出问题排查指引：

```text
IF 端到端最终答案正确率 (Answer Correctness) 低:
    │
    ├── IF 黄金上下文检索召回率 (Context Recall@K) 低:
    │     └── 结论: 【检索层缺陷】知识库存在但未召回，排查切分颗粒度或向量检索召回度。
    │
    └── IF 检索召回率高 但 上下文相关性 (Context Relevance) 低:
          └── 结论: 【重排与去噪缺陷】召回了过多无关噪声，干扰了生成模型，应调优 Reranker。
          │
          └── IF 上下文充分度高 但 忠实度 (Faithfulness) 低:
                └── 结论: 【模型幻觉/Prompt缺陷】检索准确但生成模型脱缰胡编，应增强系统指令约束或降低生成温度。
```