---
title: "文档处理与分块"
description: "解析方式与分块边界如何决定你的检索上限，为什么重叠和块大小都是评测问题，以及哪些默认值能扛住真实语料的检验。"
slug: 01-document-processing-and-chunking
part: 6
chapter: 1
difficulty: intermediate
prerequisites:
  - part-4-eval-fundamentals/00-why-you-cant-ship-without-evals
  - part-5-context-engineering/00-context-is-more-than-the-prompt
  - 00-when-you-actually-need-rag
tags: [llm, rag]
date_created: 2026-06-12
date_updated: 2026-10-16
---

# 文档处理与分块

一个团队在上季度的财报 PDF 之上搭了个财务助手。检索器没问题。嵌入模型没问题。重排器也没问题。可每次有人问“Google Cloud 第二季度营收是多少？”，答案都是错的。正确的那张表就在语料库里。检索器甚至把正确的页面取了回来。答案里的数字依然是编的。

打开被索引的那个块看看。PDF 里读起来清爽的三列营收表，现在变成了一串扁平的字符串：`Revenue 8417 9192 10347 Operating income 1187 1453 1737`。数字没有表头。季度没有标签。嵌入表示的就是这锅糊，交给 LLM 的也是这锅糊，再怎么调提示词也救不回解析器已经毁掉的东西。

检索上限在建立索引的那一刻就定死了，那时查询还不存在。块是系统能返回的最小单位。一个事实被拆到两个块里，或者一张表被压成不可解析的文本，任何嵌入模型、任何重排器都补不回来。本章讲的就是钉住这个上限的两个决策：怎么解析，怎么分块（chunking）。

## 解析：PDF 是显示格式，不是数据格式

表格崩掉的原因是结构性的。PDF 把文本存成带坐标的字形（glyph），而不是有逻辑的文本流。朴素解析器按字形在页面上出现的先后顺序读，这对单栏正文没问题，对其它版面就是灾难。双栏的学术论文会把两栏交叠着读。页眉会混进正文。表格一旦被从左到右、从上到下地读，行列表属关系当场就没了。

![左边是一张财务表格被解析成一串没有表头的数字，打了红叉；右边是同一张表被解析成结构化 HTML，行列表头完整保留，打了绿勾。](../../../ai-eng/part-6-retrieval-augmented-generation/images/01-table-parsing-failure.png)
*同一张表，两种解析方式：压平之后，关系在检索开始之前就已经丢失。*

生产工具的地盘沿两个维度分开：一类是基于规则的解析器，读取 PDF 对象模型再套启发式规则；另一类是视觉语言解析器，先把页面渲染出来再做版面检测。截至 2026 年中，四款工具覆盖了几乎所有真实流水线：

- **PyMuPDF / PyMuPDF4LLM** 是速度上的默认选择：纯文本抽取在单个 CPU 核心上约每页 0.006 秒（远超每秒 100 页），而 PyMuPDF4LLM 的版面感知 Markdown 输出慢得多，约每页 0.5 秒[^1]。2026 年 3 月发布的 PyMuPDF-Layout 在 DocLayNet 基准上 F1 达到 0.864[^2]。正文密集的程序生成 PDF、高吞吐批量索引，用它。它在合并单元格、跨页表格和任何扫描件上依然吃力。
- **Docling**（IBM Research，MIT 许可）用 DocLayNet 做版面分析，用 TableFormer transformer 还原表格结构。在 Apple M3 Max 配四个 CPU 线程的环境下，它每秒处理 1.27 到 1.34 页，占用约 6.2 GB 内存[^3]。当表格承载关键信息、文档有多栏版面，或者数据不允许离开你的网络时，选它。
- **Unstructured**（开源版加商业 API）的 `hi_res` 策略在 `infer_table_structure=True` 时，会把表格抽成 HTML，行列结构完整保留，并把元素类型（Title、NarrativeText、Table）作为元数据附上[^4]。表格的推荐做法：把 LLM 生成的自然语言摘要送去嵌入，原始 HTML 留着展示。
- **LlamaParse** 是商业化的升级选项：Agentic 档每页 10 个 credits（按标价约 $0.0125），在 LlamaIndex 自己发布的 ParseBench 上总分 84.9%，数据截至 2026 年 4 月[^5]。当准确性比数据驻留更重要时用它。

扫描版 PDF 是另一类。这种文档没有文本层，解析器只能调 OCR。Docling 默认的 EasyOCR 在 CPU 上每页要 30 秒往上[^3]，一份 500 页的扫描件在单核上就是四小时的活。这类文档要么转给 Azure Document Intelligence 或 AWS Textract，要么自己扛下跑 OCR 的 GPU 成本。

HTML 用 BeautifulSoup 或 lxml 解析，把标题层级作为元数据挂上去。LangChain 的 `HTMLHeaderTextSplitter` 会走一遍文档，给每个块打上从 `h1` 到 `h6` 的父级路径标签，这些标签在下游就成了可过滤的检索元数据。

决策比这份工具清单看起来简单：**默认用 PyMuPDF4LLM；一旦表格变得关键就升级到 Docling；只有当内部基准证明确有必要时才升级到 LlamaParse**。别靠读厂商的准确率宣传来挑解析器。挑法是：拿 50 份有代表性的文档，用两个候选方案各跑一遍，亲眼看输出。

## 分块：一串阶梯式策略，而不是一个标准答案

文本干净了，接下来必须切开。每种分块策略都落在一条成本对质量的曲线上，而行业里的“常识”一直在被推翻。过去一年有两篇论文重新打了地基：

- Bennani 和 Moslonka（arXiv:2601.14123，2026 年 1 月）在 Natural Questions 上做了受控实验，用句子感知分块、SPLADE 检索和 Mistral-8B。加 10% 到 20% 的重叠没带来可测量的提升（BERTScore 变动在 0.004 以内，Exact Match 在 0.001 以内），却把索引体积放大了 1/(1-r) 倍。等于白付 25% 的存储和入库成本[^6]。
- Qu 等人（NAACL 2025 Findings，Vectara）发现，在真实文档上语义分块并不能稳定胜过固定长度分块，未必值得付出约 14 倍的索引耗时[^7]。它的优势出现在人工构造的多样化语料上；到了生产文本里，这个优势往往就消失了。

结论不是“语义分块不行”，也不是“重叠已死”。真正的结论是：2023 到 2024 年间从教程里扩散开的那套分块默认值，在大多数语料上根本没被验证过。正确的姿态是把它当一架梯子，只有评测逼你往上时，才爬一级。

![一架竖直的梯子，列出五种分块策略：最便宜的基线在最底下，最贵的上下文检索在最上面，每一级之间标着升级条件。](../../../ai-eng/part-6-retrieval-augmented-generation/images/01-chunking-strategy-ladder.png)
*只有当评测数据说明下一级不够用时，才往上爬。*

**按 token 计的递归固定长度切分。** 这是地板。LangChain 的 `RecursiveCharacterTextSplitter` 按顺序尝试分隔符：`["\n\n", "\n", " ", ""]`，从段落退到行、退到词、退到字符。有一个坑几乎每个人第一次都会踩：

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 错：这里的 chunk_size=512 指的是 512 个字符，约 128 个 token
bad = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=0)

# 对：用 tiktoken 按 BPE token 计数
splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
    model_name="cl100k_base",   # GPT-4 / text-embedding-3 的分词器
    chunk_size=512,
    chunk_overlap=0,
    separators=["\n\n", "\n", " ", ""],
)
chunks = splitter.split_text(document_text)
```

默认的 `chunk_size` 参数数的是字符，不是 token。用默认构造函数设 `chunk_size=512`，每个块大约只有 128 个 token，只有预期大小的四分之一[^8]。表现出来就是：块数是你预期的四倍，检索精度好得不正常，而模型因为每块都太短根本推不出答案。只要你在意块大小，就始终用 `from_tiktoken_encoder`（或你的嵌入模型实际使用的分词器）。

**按句子切分** 以句子边界为界，不用固定计数。Bennani 和 Moslonka 发现，在上下文约 5,000 token 以内时，它与语义分块在统计上打平，算力却只花一个零头[^6]。他们的实用建议：默认用句子分块；只有当上下文预算非常大、或文档确实东拉西扯时才升级到语义分块。

**语义分块** 计算相邻句子的嵌入余弦相似度，在相似度跌破阈值处断一刀。直觉是对的：在话题切换处切开。实证结果好坏参半。索引慢约 14 倍[^9]，而在真实语料上质量收益常常被噪声淹没[^7]。适用场景是：文档有强烈的话题跳转（新闻聚合、多主题 wiki），*并且*你的评测显示句子分块确实不够。

**结构感知切分** 用文档自身的逻辑单元做边界：HTML 的 `h1`-`h6`、代码的 AST 函数边界、法律合同的条款编号。2025 年 11 月一项临床领域的研究发现，对齐逻辑边界的自适应分块在临床决策支持查询上准确率达到 87%，而固定长度只有 13%[^9]。这是领域特定的结果，但方向是可靠的。当文档自带的结构恰好对应用户的提问方式时，用它。

**上下文检索（Contextual Retrieval）** 是 Anthropic 2024 年 9 月的技术。嵌入之前，先调一个小模型（Claude 3 Haiku）为每个块生成 50 到 100 token 的描述，说明它落在文档的什么位置，再把这段描述拼到块的前面。数字很惊人：单靠上下文嵌入，top-20 检索失败率从 5.7% 降到 3.7%；再叠上 BM25 降到 2.9%；最上面再加一层重排降到 1.9%，降幅 67%[^10]。配合提示词缓存，一次性预处理成本约为每百万文档 token $1.02。这是实打实的升级，不是白捡的便宜，但如果你诊断出的瓶颈正是块边界处的上下文丢失，这笔钱花得值。

## 重叠是可调参数，不是默认值

10% 到 20% 重叠这条规则到处都是。Pinecone 的指南至今仍把“512 token 的块配 50 到 100 token 重叠”写成起步基线。Bennani 和 Moslonka 的研究把它戳破了：在 Natural Questions 上，用句子感知分块加 SPLADE 检索，加 10% 到 20% 的重叠让 BERTScore 变动不到 0.004、Exact Match 变动不到 0.001，而 20% 的重叠把索引撑到 1.25 倍[^6]。机制看明白之后很直观：用句子感知分块时，边界溢出很少改变 top-N 返回哪些块，所以重叠买到的多半只是近似重复的块——占掉检索预算，却不增加信息。

但限定条件很重要。那项研究只用了单一检索器、单一通用领域问答语料。作者特别点出边界敏感的领域——法律条款、编号的规格步骤、格式紧凑的参考资料——这类地方一个事实经常横跨两个块，重叠在那里可能仍然划算。正确的姿态：**从 overlap=0 起步，只有当评测集上的 A/B 对比显示它能提升检索指标、且提升足以抵消索引膨胀时，才加重叠**。

## 元数据是块的一部分

向量索引里的一个块不只是文本。它是一个带两组字段的数据对象：嵌入文本（模型编码的内容）和元数据（可过滤的来源信息，永远不会撑大嵌入向量）。省掉元数据是那种上线当天毫无代价、第 30 天代价全来的决策——那天你需要引用、需要多租户隔离、需要增量重建索引。

每个块最少要有这几个字段：

- **`source`**：文档标识或 URL，用于引用
- **`section`**：最近的标题，既能做过滤检索，也给重排器提供上下文
- **`page_number`**：供用户核对
- **`chunk_index`**：在父文档中的位置
- **`tenant_id`** 或 **`permission_level`**：任何多租户部署都要，在查询时强制执行
- **`content_hash`** 和 **`last_modified`**：文档更新时做删除并重建索引，不用重建整个语料库

一个把元数据变成硬性要求、而不是可选增强的 schema：

```python
from dataclasses import dataclass, field

@dataclass
class Chunk:
    text: str
    context_prefix: str = ""           # 用于上下文检索
    metadata: dict = field(default_factory=dict)

    @property
    def embed_text(self) -> str:
        if self.context_prefix:
            return f"{self.context_prefix}\n\n{self.text}"
        return self.text

def attach_metadata(texts: list[str], source: str, section: str, page: int) -> list[Chunk]:
    return [
        Chunk(text=t, metadata={
            "source": source, "section": section,
            "page_number": page, "chunk_index": i,
        })
        for i, t in enumerate(texts)
    ]
```

表格沿用 Unstructured 的做法：嵌入 LLM 生成的自然语言表格摘要；把原始 HTML 存进 `display_html` 元数据字段；查询时把 HTML 返回给应用去渲染[^4]。向量索引检索的是语义，用户看到的是结构。

## 块大小是个评测问题

没有哪个块大小能处处取胜，也没有办法靠直觉找到属于你的那个。这个变量与三件没有数据就无法预判的事相互作用：你的查询分布、嵌入模型对块长度的敏感度、以及你的文档结构。

Bhat 等人（arXiv:2505.21700，2025 年 5 月）在多个嵌入模型和数据集上测了固定长度块。最优值按查询类型分得很干净：答案简短的事实型查询用 64-128 个 token，需要更宽上下文的查询用 512-1024 个 token。嵌入模型也各有性格。Stella 在长距离检索上偏好更大的块，Snowflake 则在细粒度实体匹配上小块表现更好[^11]。LlamaIndex 针对 Uber 2021 年 10-K 文件的研究测了 128 到 2,048 token 的各档，发现 1,024 取得最高忠实度，延迟只小幅上升[^9]。Weaviate 2025 年 9 月的指南报告，同一语料、同一检索器上，最好与最差的分块策略在召回率上能差到 9%[^9]。

Bennani 和 Moslonka 还揭示了一个约束：在他们的设置下，检索到的上下文总量到约 2,500 token 时出现一道上下文悬崖。在此之前 BERTScore 稳定；越过去，到 10,000 token 时准确率相对下降 4-5%[^6]。机制就是贯穿[压缩与上下文预算](../part-5-context-engineering/02-compression-and-context-budgets.md)的“中间遗忘”效应：上下文更多不等于能力更强。悬崖的确切位置取决于模型，但形状是一致的。

出路是一套测量循环，不是猜。[RAGAS](https://docs.ragas.io/) 框架提供四个由 LLM 当裁判的指标，起步阶段不需要标注好的标准答案：`context_precision`（检索回来的内容里有多少真被用上）、`context_recall`（该检索到的有没有都检索到）、`faithfulness`（回答是否忠于材料）、`answer_relevancy`（回答是否切题）。固定除块大小之外的所有变量，扫过 256、512、1024、2048 个 token，看着精确率和召回率朝相反方向移动，选出它们在你*自己*的查询上交叉的那个大小。换了嵌入模型就重跑同一套扫描。完整的 RAG 评测框架在[评测检索](../../../ai-eng/part-6-retrieval-augmented-generation/07-evaluating-retrieval.md)，而坚持跑评测这件事本身在第四部分的[为什么没有评测就不能上线](../part-4-eval-fundamentals/00-why-you-cant-ship-without-evals.md)。

在建评测集期间的合理起点：512 个 token、不重叠、句子切分配 `from_tiktoken_encoder`、入库时挂好元数据。这是假设，不是配置。每个把 RAG 系统真正上线的团队，提交历史里都躺着那次证明自家语料块大小的实验；六个月后语料变了，又躺着另一次。

放到架构层面，HLD 第九部分的[企业级 RAG 系统设计](../../hld/part-9-ai-ml-system-design/01-rag-pipelines.md)用白板视角讲了索引流水线、存储拓扑和多租户隔离。

## 参考文献

[^1]: Artifex, "PyMuPDF benchmark" (ParseBench, AWS c7i.2xlarge, single CPU core, PyMuPDF 1.28.2), September 2026, https://pymupdf.io/benchmark
[^2]: Artifex, "PyMuPDF-Layout Performance on DocLayNet: A Comparative Evaluation," March 2026, https://artifex.com/blog/pymupdf-layout-performance-on-doclaynet-a-comparative-evaluation
[^3]: Christoph Auer et al. (IBM Research), "Docling Technical Report," arXiv:2408.09869, August 2024, https://arxiv.org/abs/2408.09869
[^4]: Unstructured, "Preserving Table Structure for Better Retrieval," August 2025, https://unstructured.io/blog/preserving-table-structure-for-better-retrieval
[^5]: LlamaIndex, "Introducing LlamaParse v2", December 18, 2025, https://www.llamaindex.ai/blog/introducing-llamaparse-v2-simpler-better-cheaper; LlamaIndex, "Pricing" (1,000 credits = $1.25), https://www.llamaindex.ai/pricing; LlamaIndex, "ParseBench", April 13, 2026, https://www.llamaindex.ai/blog/parsebench
[^6]: Hicham Bennani and Maxime Moslonka, "A Systematic Analysis of Chunking Strategies for Reliable Question Answering," arXiv:2601.14123, January 2026, https://arxiv.org/abs/2601.14123
[^7]: Renyi Qu, Ruixuan Bao, and Lyumanshan Tu (Vectara), "Is Semantic Chunking Worth the Computational Cost?", NAACL 2025 Findings, https://aclanthology.org/2025.findings-naacl.114/
[^8]: LangChain documentation, "How to split text by tokens," https://python.langchain.com/docs/how_to/split_by_token/
[^9]: Digital Applied, "RAG Chunking Strategies: A 2026 Retrieval Playbook," May 2026, https://www.digitalapplied.com/blog/rag-chunking-strategies-2026-retrieval-quality-playbook (synthesizing Chonkie benchmarks, LlamaIndex 10-K study, Weaviate September 2025 guide, and MDPI Bioengineering November 2025 clinical study)
[^10]: Anthropic, "Introducing Contextual Retrieval," September 19, 2024, https://www.anthropic.com/engineering/contextual-retrieval
[^11]: Ishaan Bhat et al., "Rethinking Chunk Size For Long-Document Retrieval," arXiv:2505.21700, May 2025, https://arxiv.org/abs/2505.21700
