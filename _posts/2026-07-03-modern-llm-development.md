---
title: "现代大语言模型的发展脉络与最新技术"
date: 2026-07-03 21:00:00 +0800
description: 梳理现代 LLM 从 Transformer、预训练、规模律、指令对齐到 RAG、工具调用、多模态与推理模型的技术演进脉络。
categories:
  - AI
home_category: essay-comments
tags:
  - LLM
  - Transformer
  - GPT
  - RAG
  - Agent
  - Multimodal
  - Reasoning
article_kicker: AI ESSAY
cover_image: /assets/images/modern_llm_development/image-20260703140329177.png
word_count: 约 8k 字
read_time: 约 28 分钟
---

## 摘要

现代大语言模型（Large Language Model, LLM）的发展，不是一条简单的“参数越大越强”的直线，而是由模型架构、训练数据、计算规模、对齐方法、工具使用、多模态能力和推理时计算共同推动的技术演进。本文按技术范式的形成顺序梳理现代 LLM 的核心脉络，并引用其中具有奠基意义的关键论文。

从早期统计语言模型到 Transformer，从 BERT 和 GPT 的预训练范式到 GPT-3 的少样本学习，从 ChatGPT 的指令对齐到 GPT-4 级别的通用能力平台，再到近年的 RAG、工具调用、智能体、多模态和 reasoning model，LLM 已经从“预测下一个 token 的文本模型”演化为“模型、工具、检索、记忆、推理和执行环境组合而成的 AI 系统”。

## 1. 前史：从统计语言模型到神经表征学习

早期自然语言处理（NLP）的核心问题不是“生成智能”，而是如何把语言变成机器可以处理的统计对象。n-gram 语言模型通过前若干个词预测下一个词，例如用 `P(w_t | w_{t-1}, w_{t-2})` 来近似语言概率。这类方法简单有效，但存在明显局限：无法处理长距离依赖，稀疏性严重，也缺乏稳定的语义表示。

2013 年前后，Word2Vec、GloVe 等词向量方法改变了 NLP 的基础表示方式。词不再只是离散符号，而是连续向量空间中的点。语义相近的词在向量空间中距离更近，甚至可以出现类似 `king - man + woman ≈ queen` 的线性结构。这一阶段的核心贡献在于：语言中的语义关系可以通过大规模语料中的统计共现被压缩进向量表示。

随后，RNN、LSTM、GRU 等循环神经网络被广泛用于建模序列数据。seq2seq 架构推动了机器翻译的发展：编码器把输入句子压缩成向量，解码器再生成目标语言文本。但这种结构在长句中容易出现信息瓶颈。注意力机制最初就是为了解决这个问题：模型在生成每个词时，可以动态关注输入序列的不同部分，而不是依赖单一固定向量。

这一系列进展为 Transformer 的出现奠定了基础。

## 2. Transformer：现代 LLM 的架构起点

2017 年，Vaswani 等人在论文《Attention Is All You Need》中提出 Transformer 架构。它抛弃了 RNN 的循环结构，完全依赖自注意力机制（self-attention）处理序列。Transformer 的意义不仅是效果更好，更重要的是它让训练可以高度并行化，从而适合在海量数据和大规模硬件上扩展。

自注意力机制的核心思想是：序列中的每个 token 都可以根据上下文中其他 token 的相关性动态聚合信息。相比 RNN 逐步扫描序列，Transformer 能更直接地建模长距离依赖。

Transformer 的关键组件包括：

- `Self-Attention`：计算序列内部 token 之间的依赖关系。
- `Multi-Head Attention`：让模型从多个子空间并行捕捉不同关系。
- `Position Encoding`：补充序列位置信息，因为自注意力本身不包含顺序。
- `Feed-Forward Network`：对每个位置的表示进行非线性变换。
- `Residual Connection` 和 `LayerNorm`：改善深层网络训练稳定性。

今天主流 LLM 几乎都属于 Transformer 架构族。无论是 GPT、LLaMA、Claude、Gemini、Qwen、DeepSeek，还是各类代码模型、多模态模型和推理模型，其基础结构都可以追溯到 Transformer。

> ### Transformer 架构：Encoder 与 Decoder
>
> ![Transformer 架构图](/assets/images/modern_llm_development/image-20260703140329177.png)
>
> #### 什么是 Encoder？什么是 Decoder？
>
> **Encoder 是“理解输入”的模块。**
>
> Encoder 接收一整段输入序列，例如一句话“我 喜欢 机器 学习”，它会让每个 token 通过 self-attention 观察整句话中的其他 token，得到一组带上下文含义的表示。
>
> 比如“苹果”这个词，Encoder 会根据上下文判断它更像水果还是公司：
>
> - `我买了一个苹果。` -> 水果
> - `苹果发布了新芯片。` -> 公司
>
> Encoder 的特点是：它通常可以同时看到输入序列的左右两边上下文。所以 Encoder 适合做理解型任务，例如文本分类、情感分析、语义匹配、信息抽取、文档检索和阅读理解。
>
> **Decoder 是“生成输出”的模块。**
>
> Decoder 负责一个 token 一个 token 地生成文本。例如模型要生成“我喜欢机器学习”，它通常按顺序生成：
>
> - `我`
> - `我 喜欢`
> - `我 喜欢 机器`
> - `我 喜欢 机器 学习`
>
> Decoder 的 self-attention 通常带有 causal mask，也就是当前位置只能看见自己之前的 token，不能偷看未来 token。例如生成第三个词“机器”时，模型只能看到“我 喜欢”，不能提前看到后面的“学习”。这种机制符合语言生成的本质：预测下一个 token。
>
> 所以 Decoder 适合做生成型任务，例如聊天、写作、代码生成、翻译生成、摘要生成和推理回答。GPT、LLaMA、Claude、Qwen、DeepSeek 这类现代大语言模型，大多是 decoder-only 模型。
>
> 在上面的 Transformer 架构图中，左边结构是 Encoder，右边结构是 Decoder。
>
> | 结构 | Encoder | Decoder |
> | --- | --- | --- |
> | 输入 | 源序列，例如中文句子 | 已生成的目标序列，例如英文前缀 |
> | Self-Attention | 双向，可看完整输入 | 单向 masked，只能看过去 |
> | Cross-Attention | 没有 | 有，读取 encoder 输出 |
> | FFN | 有 | 有 |
> | 输出 | 输入序列的上下文表示 | 下一个 token 的概率分布 |

## 3. 预训练范式：从特定任务模型到通用语言模型

Transformer 出现后，NLP 很快转向“预训练 + 下游适配”的范式。核心思想是：先在海量无标注文本上学习通用语言规律，再迁移到具体任务。

2018 年的 GPT-1 证明了“生成式预训练 + 任务微调”的可行性。它使用 decoder-only Transformer，通过自回归语言建模预测下一个 token。同期，BERT 则采用 encoder-only Transformer，通过遮蔽语言建模（Masked Language Modeling）学习双向上下文表示，并在阅读理解、文本分类、自然语言推理等任务上取得突破。

BERT 的关键意义不在于它能聊天，而在于它证明了通用语言表征可以通过大规模预训练获得，然后只需少量任务数据微调，就能在多个 NLP 任务上获得强性能。

此后，Transformer 语言模型大致形成三类架构路线：

- `Encoder-only`：以 BERT 为代表，擅长理解、分类、检索和抽取。
- `Decoder-only`：以 GPT 为代表，擅长自回归生成，后来成为主流 LLM 架构。
- `Encoder-decoder`：以 T5、BART 为代表，适合文本到文本任务，如翻译、摘要和问答。

现代聊天模型大多采用 decoder-only 架构，因为它天然适合“给定上下文，继续生成下一个 token”的交互形式。

> ### Decoder-only 架构
>
> ![GPT-1 Decoder-only 架构图](/assets/images/modern_llm_development/image-20260703141607733.png)
>
> 上图为 GPT-1 的模型架构图，可以看到相较于原始 Transformer，左边 Encoder 部分被删除，只保留了右侧的 Decoder 结构。

## 4. 规模律时代：GPT-3 与少样本学习

2020 年前后，研究者开始系统研究模型规模、数据规模和训练计算量之间的关系。Kaplan 等人在《Scaling Laws for Neural Language Models》中发现，语言模型损失会随参数量、训练数据量和计算量呈近似幂律下降。这一结论推动了行业进入大规模扩展阶段。

同年，OpenAI 发布 GPT-3。GPT-3 使用 1750 亿参数的 decoder-only Transformer，展示出一个关键现象：模型不再只依赖任务微调，而能通过 prompt 中的自然语言说明和少量示例完成新任务。这被称为 in-context learning 或 few-shot learning。

GPT-3 的意义在于，它把语言模型从“每个任务训练一个专用模型”推进到“一个大模型通过上下文临时适配任务”。用户可以在 prompt 中写入任务说明、样例、约束，模型就在上下文中模拟一个临时任务求解器。

这带来了对 LLM 的新理解：大模型不只是文本生成器，而是某种通用条件程序。输入上下文相当于临时程序和数据，模型根据上下文生成符合任务目标的输出。

## 5. 计算最优：Chinchilla 对“大模型路线”的修正

早期扩展路线偏向堆参数，但 DeepMind 的 Chinchilla 研究指出，许多大模型其实训练数据不足。在固定计算预算下，参数量和训练 token 数需要合理配比。Chinchilla 使用 700 亿参数和更多训练数据，在多项任务上超过参数量更大的模型。

这改变了行业对“规模”的理解。LLM 的能力不是由参数量单变量决定，而是由多个因素共同决定：

- 模型参数规模。
- 训练 token 数量。
- 数据质量和数据分布。
- 训练稳定性。
- 模型架构与优化器。
- 推理成本与部署效率。

后来的 LLaMA、Mistral、Qwen、DeepSeek 等模型进一步证明：较小但数据充分、训练精细的模型，可以接近甚至超过早期巨型闭源模型。

LLaMA 尤其重要，因为它推动了开放权重模型生态的爆发。研究者和开发者可以在本地微调、量化、部署模型，围绕 LoRA、QLoRA、RAG、agent、私有知识库等形成了丰富生态。

这一阶段可以用几个开放权重模型作为例子理解：

- `LLaMA / Llama 2 / Llama 3`：代表开放权重基础模型路线，让研究者可以在本地复现实验、微调和部署。
- `Mistral 7B / Mixtral 8x7B`：代表“小而强”和稀疏 MoE 路线。Mixtral 8x7B 总参数更多，但每个 token 只激活部分专家，从而控制推理成本。
- `Qwen / Qwen2.5 / Qwen3`：代表多语言、代码、长上下文和推理模式结合的开放模型路线。Qwen3 还引入 thinking / non-thinking 双模式，体现了“可控思考预算”的趋势。
- `DeepSeek-V2 / V3 / R1 / V4 Preview`：代表高效 MoE、强化学习推理和超长上下文方向。DeepSeek-R1 使开放 reasoning model 进入主流，DeepSeek-V4 Preview 则把开放权重、高活跃参数效率和 1M 级上下文作为重点。

## 6. ChatGPT 的关键：指令微调与人类偏好对齐

原始预训练模型的目标很简单：预测下一个 token。这个目标可以让模型学到知识、语言风格、代码模式和推理片段，但不会天然学会如何做一个“有用、诚实、安全”的助手。

ChatGPT 体验上的跃迁，关键来自指令微调和 RLHF（Reinforcement Learning from Human Feedback）。

InstructGPT 的训练流程通常包括三步：

1. 收集人工示范数据，对预训练模型进行监督微调。
2. 让人类比较多个模型回答，训练奖励模型。
3. 使用强化学习优化模型，使其更符合人类偏好。

InstructGPT 论文显示，一个经过人类偏好对齐的较小模型，在用户偏好上可以超过更大的原始 GPT-3 模型。这说明“模型是否好用”很大程度来自后训练，而不只是预训练规模。

之后，对齐技术继续演化：

- `RLHF`：用人类偏好训练奖励模型，再通过强化学习优化模型输出。
- `RLAIF`：用 AI 反馈替代部分人类反馈，降低标注成本。
- `Constitutional AI`：用显式原则指导模型自我批评和修正。
- `DPO`：Direct Preference Optimization 将偏好优化简化成类似监督学习的目标，避免复杂的 PPO 流程。
- `Self-Instruct`：用模型自动生成指令数据，再筛选和训练。

因此，现代 LLM 的训练通常不是一次预训练结束，而是多阶段流程：预训练获得通用能力，监督微调塑造指令跟随能力，偏好优化塑造交互体验和行为边界。

## 7. GPT-4 阶段：从聊天模型到通用能力平台

GPT-4 标志着 LLM 从“强聊天模型”进一步走向“通用能力平台”。OpenAI 的 GPT-4 技术报告称其是一个可接受图像和文本输入、输出文本的大型多模态模型，并在多种专业和学术 benchmark 上达到接近人类的表现。

这一阶段的变化包括：

- 模型能处理更复杂、更细致的用户指令。
- 长上下文能力增强，可以处理长文档、代码库和复杂报告。
- 多模态输入开始进入主流，图像、截图、表格和文档理解成为常见能力。
- 评测从传统 NLP benchmark 扩展到考试、代码、数学、医学、法律、安全红队等领域。
- 模型能力越来越依赖系统工程，而不只是单一模型架构。

GPT-4 之后，LLM 的产品形态发生明显变化。它不再只是一个聊天窗口，而逐渐成为搜索、办公、编程、数据分析、教育、客服和内容生产的底层能力。

这一阶段的代表性闭源模型包括：

- `GPT-4 / GPT-4 Turbo / GPT-4o`：GPT-4 强调复杂任务和多模态能力，GPT-4o 进一步把文本、图像和语音交互统一到更低延迟的实时多模态体验中。
- `OpenAI o1 / o3 系列`：代表从通用聊天模型转向专门 reasoning model 的路线，重点不是更快回答，而是允许模型在复杂任务上花更多计算进行推理。
- `Claude 3.5 / 3.7 Sonnet / Claude 4`：Anthropic 路线强调长文档、代码、agent 工作流和可控 extended thinking。Claude 3.7 Sonnet 是典型的“普通回答 + 扩展思考”混合模型，Claude 4 则进一步面向长程编码和 agent 任务。
- `Gemini 1.5 / 2.0 / 2.5`：Google 路线突出原生多模态、超长上下文和 thinking model。Gemini 1.5 让百万 token 级上下文进入主流讨论，Gemini 2.5 则明确把“先思考再回答”作为核心能力。
- `Grok 3`：xAI 路线强调大规模预训练知识与强化学习推理结合，并将 test-time compute 用于数学、代码和科学推理任务。

## 8. RAG：让模型连接外部知识

LLM 有一个根本限制：参数中的知识会过期，也可能产生幻觉。RAG（Retrieval-Augmented Generation，检索增强生成）通过把外部知识库检索结果放入上下文，让模型基于最新或私有资料回答问题。

典型 RAG 流程如下：

1. 将文档切分成片段。
2. 使用 embedding 模型把片段编码成向量。
3. 用户提问时，将问题编码成向量。
4. 从向量数据库中检索相关片段。
5. 将检索结果与用户问题一起放入 LLM 上下文。
6. 模型基于上下文生成回答。

RAG 的优势在于：

- 知识可以更新，不必重新训练模型。
- 可以接入企业私有数据。
- 回答可以附带来源，便于追踪。
- 能降低部分事实性幻觉。

但 RAG 不是万能解法。其效果高度依赖文档切分、索引质量、召回策略、重排序、上下文压缩和提示词设计。如果检索不到正确资料，模型仍可能编造。

## 9. 工具调用与智能体：模型开始“行动”

工具调用进一步改变了 LLM 的角色。模型不再只生成自然语言，而可以调用搜索引擎、计算器、数据库、代码解释器、浏览器、文件系统和外部 API。

Toolformer 证明，语言模型可以学习何时调用外部工具。ReAct 则提出将推理和行动交错进行：模型先分析问题，再执行动作，根据环境观察继续推理。

智能体（agent）通常由以下组件构成：

- `模型本体`：负责理解、规划、生成。
- `工具系统`：搜索、代码执行、数据库、浏览器、API 等。
- `记忆系统`：短期上下文、长期知识库、任务历史。
- `规划机制`：任务分解、步骤安排、优先级判断。
- `反思机制`：检查错误、重试、修正策略。
- `环境反馈`：测试结果、网页返回、文件变化、用户反馈。

这使 LLM 从“回答问题”进入“执行任务”的阶段。软件工程、数据分析、自动化办公、网页操作和机器人控制，都是 agent 化的重要应用方向。

不过 agent 也带来新的难题。模型在长程任务中容易偏航，错误会在多步执行中累积。工具权限、安全边界、状态管理和可观测性，成为实际部署中的核心工程问题。

## 10. 多模态：从语言模型到世界接口

现代 LLM 正在从纯文本模型演化为多模态基础模型。CLIP 通过图文对比学习证明，自然语言可以作为视觉监督信号，让模型获得零样本视觉分类能力。Flamingo 则把视觉模型和语言模型连接起来，处理图文交错输入。

GPT-4V、Gemini、Claude、GPT-4o 等模型进一步推动多模态进入主流应用。模型可以理解截图、照片、图表、PDF、音频和视频，甚至进行实时语音对话。

多模态的长期意义在于：模型不再只处理文字世界，而能连接人类真实工作环境中的多种信息形态。例如：

- 阅读屏幕截图并指导用户操作。
- 理解图表和实验结果。
- 分析视频片段。
- 进行语音实时交互。
- 结合机器人传感器执行物理任务。

因此，未来很多 AI 助手本质上会是多模态 agent，而不是单纯的聊天机器人。

## 11. 效率革命：LoRA、QLoRA、FlashAttention 与 MoE

LLM 的发展不只是前沿模型变强，也包括训练和部署成本不断下降。

LoRA 通过冻结原模型参数，只训练低秩适配矩阵，大幅降低微调成本。QLoRA 进一步把基础模型量化到 4-bit，并通过 LoRA 进行反向传播，使大模型微调可以在更低显存条件下完成。

FlashAttention 从 GPU 存储访问角度优化 attention 计算，减少显存读写开销，使长上下文训练和推理更高效。

MoE（Mixture of Experts，专家混合）则在架构层面重新流行。它的思路是：模型拥有大量专家子网络，但每个 token 只激活其中一部分专家。这样可以在不显著增加每次推理计算量的前提下，提升总参数容量。

Switch Transformer、Mixtral、DeepSeek-V3/R1 等都体现了类似思想。MoE 的核心价值是平衡容量和计算成本：模型可以很大，但每次只用一部分。

这说明未来模型竞争不只是“谁参数更多”，而是“谁在同等成本下更强、更快、更可部署”。

## 12. 现代 LLM 技术栈的七层结构

今天的 LLM 系统可以概括为七层：

1. `Tokenization`：把文本、图像 patch、音频片段等转成模型可处理的 token 或 embedding。
2. `Transformer / MoE Backbone`：负责大规模模式学习和上下文建模。
3. `Pretraining`：用海量数据学习语言、知识、代码、图文关联和世界规律。
4. `Post-training`：指令微调、RLHF、DPO、RLAIF、安全对齐和风格控制。
5. `Context Engineering`：长上下文、RAG、记忆、结构化提示和文档注入。
6. `Tool / Agent Layer`：搜索、代码执行、浏览器、数据库、外部 API 和环境交互。
7. `Reasoning-time Compute`：多次采样、验证器、过程奖励、树搜索、反思和预算控制。

越新的 AI 系统，越不像“一个模型”，而更像“模型、工具、记忆、检索、执行环境、安全策略和评测系统”的组合工程。

## 13. 结论

现代 LLM 的发展可以概括为五次范式跃迁。

第一次是 Transformer，让大规模并行序列建模成为可能。第二次是预训练，让通用语言表征成为基础。第三次是规模律和 GPT-3，让 in-context learning 出现。第四次是 ChatGPT 和 GPT-4，通过指令对齐和产品化把 LLM 变成通用助手。第五次是 2024 年之后的推理模型、多模态、RAG、工具调用和 agent 化，把 LLM 推向“可思考、可感知、可行动”的系统。

截至目前，最值得关注的前沿不是单一模型名称，而是三条技术合流：

- `推理时计算扩展`：模型在回答前动态消耗更多计算进行思考、搜索和验证。
- `多模态智能体`：模型连接文本、图像、语音、视频、工具和执行环境。
- `低成本开放生态`：开放权重、量化、LoRA、MoE 和高效推理降低使用门槛。

未来 LLM 的竞争，很可能不再只是预训练规模竞争，而是“谁能更可靠地把知识、推理、工具、行动和安全约束整合成可持续执行任务的系统”。

## 参考论文与资料

1. Vaswani et al., 2017. *Attention Is All You Need*. https://arxiv.org/abs/1706.03762
2. Devlin et al., 2018. *BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding*. https://arxiv.org/abs/1810.04805
3. Brown et al., 2020. *Language Models are Few-Shot Learners*. https://arxiv.org/abs/2005.14165
4. Kaplan et al., 2020. *Scaling Laws for Neural Language Models*. https://arxiv.org/abs/2001.08361
5. Lewis et al., 2020. *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*. https://arxiv.org/abs/2005.11401
6. Fedus et al., 2021. *Switch Transformers: Scaling to Trillion Parameter Models with Simple and Efficient Sparsity*. https://arxiv.org/abs/2101.03961
7. Radford et al., 2021. *Learning Transferable Visual Models From Natural Language Supervision*. https://arxiv.org/abs/2103.00020
8. Hu et al., 2021. *LoRA: Low-Rank Adaptation of Large Language Models*. https://arxiv.org/abs/2106.09685
9. Wei et al., 2022. *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. https://arxiv.org/abs/2201.11903
10. Ouyang et al., 2022. *Training language models to follow instructions with human feedback*. https://arxiv.org/abs/2203.02155
11. Hoffmann et al., 2022. *Training Compute-Optimal Large Language Models*. https://arxiv.org/abs/2203.15556
12. Alayrac et al., 2022. *Flamingo: a Visual Language Model for Few-Shot Learning*. https://arxiv.org/abs/2204.14198
13. Dao et al., 2022. *FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness*. https://arxiv.org/abs/2205.14135
14. Yao et al., 2022. *ReAct: Synergizing Reasoning and Acting in Language Models*. https://arxiv.org/abs/2210.03629
15. Bai et al., 2022. *Constitutional AI: Harmlessness from AI Feedback*. https://arxiv.org/abs/2212.08073
16. Wang et al., 2022. *Self-Instruct: Aligning Language Models with Self-Generated Instructions*. https://arxiv.org/abs/2212.10560
17. Schick et al., 2023. *Toolformer: Language Models Can Teach Themselves to Use Tools*. https://arxiv.org/abs/2302.04761
18. Touvron et al., 2023. *LLaMA: Open and Efficient Foundation Language Models*. https://arxiv.org/abs/2302.13971
19. OpenAI, 2023. *GPT-4 Technical Report*. https://arxiv.org/abs/2303.08774
20. Dettmers et al., 2023. *QLoRA: Efficient Finetuning of Quantized LLMs*. https://arxiv.org/abs/2305.14314
21. Rafailov et al., 2023. *Direct Preference Optimization: Your Language Model is Secretly a Reward Model*. https://arxiv.org/abs/2305.18290
22. Wang et al., 2023. *Voyager: An Open-Ended Embodied Agent with Large Language Models*. https://arxiv.org/abs/2305.16291
23. Gemini Team, 2024. *Gemini 1.5: Unlocking multimodal understanding across millions of tokens of context*. https://arxiv.org/abs/2403.05530
24. Yang et al., 2024. *SWE-agent: Agent-Computer Interfaces Enable Automated Software Engineering*. https://arxiv.org/abs/2405.15793
25. Brown et al., 2024. *Large Language Monkeys: Scaling Inference Compute with Repeated Sampling*. https://arxiv.org/abs/2407.21787
26. Snell et al., 2024. *Scaling LLM Test-Time Compute Optimally can be More Effective than Scaling Model Parameters*. https://arxiv.org/abs/2408.03314
27. OpenAI, 2024. *Learning to Reason with LLMs*. https://openai.com/index/learning-to-reason-with-llms/
28. OpenAI, 2024. *Hello GPT-4o*. https://openai.com/index/hello-gpt-4o/
29. DeepSeek-AI, 2025. *DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning*. https://arxiv.org/abs/2501.12948
30. Muennighoff et al., 2025. *s1: Simple test-time scaling*. https://arxiv.org/abs/2501.19393
31. Ye et al., 2025. *LIMO: Less is More for Reasoning*. https://arxiv.org/abs/2502.03387
32. Mistral AI, 2023. *Mixtral of Experts*. https://mistral.ai/news/mixtral-of-experts
33. Meta AI, 2025. *The Llama 4 herd: The beginning of a new era of natively multimodal AI innovation*. https://ai.meta.com/blog/llama-4-multimodal-intelligence/
34. Qwen Team, 2025. *Qwen3: Think Deeper, Act Faster*. https://qwenlm.github.io/blog/qwen3/
35. Anthropic, 2025. *Introducing Claude 4*. https://www.anthropic.com/news/claude-4
36. Google, 2025. *Gemini 2.5: Our most intelligent AI model*. https://blog.google/technology/google-deepmind/gemini-model-thinking-updates-march-2025/
37. xAI, 2025. *Grok 3 Beta - The Age of Reasoning Agents*. https://x.ai/news/grok-3
38. DeepSeek, 2026. *DeepSeek-V4 Preview Release*. https://api-docs.deepseek.com/news/news250120
