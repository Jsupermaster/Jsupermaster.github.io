---
title: "大语言模型推理与硬件加速综述"
date: 2026-07-03 14:30:00 +0800
description: 从推理阶段的计算流程、KV Cache、量化、PagedAttention、MoE 到 NPU/ASIC 设计约束，系统梳理 LLM 推理硬件加速的关键问题。
categories:
  - AI Accelerator
home_category: engineering-practice
tags:
  - LLM
  - Inference
  - Hardware
  - NPU
  - ASIC
  - KV Cache
  - Attention
article_kicker: AI HARDWARE NOTE
cover_image: /assets/images/llm_inference_hardware_overview/llm-inference-hardware-hero.png
word_count: 约 7k 字
read_time: 约 24 分钟
---

## 摘要

从硬件角度看，大语言模型（Large Language Model, LLM）并不是一个单纯的“矩阵乘法程序”。它确实以大规模矩阵乘法为主体，但真实推理系统还包含 attention、归一化、位置编码、激活函数、采样、KV Cache 管理、请求调度、显存管理、多卡通信和模型结构适配等一系列问题。

训练阶段关注的是如何用大规模数据把模型参数学出来，推理阶段关注的是如何在给定模型参数后，以可接受的成本、延迟和能耗持续服务用户请求。对 NPU、TPU、GPU、FPGA 或 ASIC 加速器而言，关键问题不是“峰值算力有多高”，而是：

- 模型的计算是否能持续喂满计算阵列。
- 权重、激活、KV Cache 是否能高效搬运。
- 低精度计算是否能保持精度和稳定性。
- 不同模型结构是否能映射到同一套硬件。
- serving 场景中的动态请求是否能被 runtime 和调度器处理。

因此，LLM 硬件加速的本质是模型、算子、存储层次、数据流、编译器和服务系统的协同设计。

## 1. 推理和训练的差异

训练通常以大 batch、大矩阵、长时间连续执行为主。一次训练 step 包含前向传播、反向传播和参数更新，计算密度高，矩阵乘法规模大，硬件利用率相对容易做高。

推理则分为两个阶段：

1. `Prefill`：处理用户输入的 prompt。假设输入长度为 `S`，模型可以一次性处理这 `S` 个 token，生成对应的 KV Cache。这个阶段的计算类似训练前向传播，矩阵规模较大，并行度较高。
2. `Decode`：逐 token 生成输出。每一步只新生成一个 token，但要读取此前所有 token 的 KV Cache。这个阶段有强自回归依赖，无法把未来 token 完全并行展开。

这导致推理的瓶颈具有阶段性：

- prefill 往往更偏计算密集，主要压力来自大矩阵乘法和 attention。
- decode 往往更偏访存密集，主要压力来自权重读取、KV Cache 读取和小 batch 下的计算阵列利用率。

对硬件而言，训练场景更像“持续的大矩阵吞吐任务”，推理场景更像“计算、访存、调度和动态控制混合任务”。这也是为什么同一颗芯片在训练和推理中的有效利用率可能差异很大。

## 2. 一个典型 decoder-only Transformer 的计算流程

现代聊天类 LLM 大多采用 decoder-only Transformer。下面用一个简化的单层结构说明其计算方式。设：

- batch size 为 `B`。
- 当前序列长度为 `S`。
- hidden size 为 `D`。
- attention head 数为 `H`。
- 每个 head 的维度为 `d = D / H`。
- FFN 中间维度为 `F`，通常约为 `2.5D` 到 `4D`，不同模型会有所变化。

输入 token id 先经过 embedding：

```text
X_0 = Embedding(token_ids)
X_0 shape: [B, S, D]
```

每一层 Transformer block 通常包含 attention 子层和 FFN 子层。以目前常见的 pre-norm 结构为例：

```text
Y = X + Attention(Norm(X))
Z = Y + FFN(Norm(Y))
```

其中 `Z` 是这一层输出，并作为下一层输入。

### 2.1 归一化：LayerNorm 与 RMSNorm

早期 Transformer 常用 LayerNorm，很多现代 LLM 使用 RMSNorm。LayerNorm 可以写成：

```text
mean = average(x)
var  = average((x - mean)^2)
LayerNorm(x) = gamma * (x - mean) / sqrt(var + eps) + beta
```

RMSNorm 去掉均值中心化，形式更简单：

```text
rms = sqrt(average(x^2) + eps)
RMSNorm(x) = gamma * x / rms
```

从硬件角度看，归一化不是主要 FLOPs 来源，但它包含 reduction、平方、倒数平方根、乘法等操作，且常常夹在大矩阵乘法之间。如果每个小算子都单独读写 HBM，会造成明显访存开销。因此实际加速中经常把 RMSNorm、残差连接、量化/反量化等操作做 fusion。

### 2.2 QKV 投影

attention 的第一步是把输入 `X` 投影为 query、key、value：

```text
Q = X W_Q
K = X W_K
V = X W_V
```

其中：

```text
X   shape: [B, S, D]
W_Q shape: [D, Hq * d]
W_K shape: [D, Hkv * d]
W_V shape: [D, Hkv * d]
```

在标准 multi-head attention 中，`Hq = Hkv = H`。在 Multi-Query Attention（MQA）或 Grouped-Query Attention（GQA）中，`Hkv` 小于 `Hq`。这会减少 K/V 的存储和读取成本，尤其有利于长上下文 decode。

很多实现会把三个矩阵合并成一个大矩阵：

```text
[Q, K, V] = X W_QKV
```

这样可以减少 kernel launch 和访存次数，也更适合矩阵计算阵列。

### 2.3 位置编码：RoPE 等机制

Transformer 本身不包含序列位置信息，因此需要位置编码。现代 decoder-only LLM 常用 RoPE（Rotary Position Embedding）。它不是简单地把位置向量加到输入上，而是在 Q、K 的部分维度上施加二维旋转：

```text
RoPE(q_pos) = rotate(q, position)
RoPE(k_pos) = rotate(k, position)
```

可以粗略理解为：

```text
[x_1', x_2'] = [x_1 cos(theta) - x_2 sin(theta),
                x_1 sin(theta) + x_2 cos(theta)]
```

RoPE 的硬件含义是：attention 前需要额外的 elementwise 旋转操作，并且不同模型可能有不同的 RoPE base、缩放方式和长上下文扩展方式。硬件如果只把位置编码写死，会很难适配后续模型。

### 2.4 Attention 计算

标准 scaled dot-product attention 为：

```text
Score = Q K^T / sqrt(d)
P = Softmax(Score + Mask)
O = P V
```

其中 causal mask 保证第 `t` 个 token 只能看见 `0..t` 的历史 token，不能看到未来 token。

在 prefill 阶段：

```text
Q shape: [B, Hq, S, d]
K shape: [B, Hkv, S, d]
V shape: [B, Hkv, S, d]
Score roughly shape: [B, Hq, S, S]
```

在 decode 阶段，每次只处理一个新 token：

```text
q_new shape: [B, Hq, 1, d]
K_cache shape: [B, Hkv, S_cache, d]
V_cache shape: [B, Hkv, S_cache, d]
```

此时 attention 需要把新 query 和历史所有 key 做点积，再用权重加权历史 value。随着上下文变长，每一步 decode 都要访问越来越大的 KV Cache。

### 2.5 输出投影

attention 输出会经过输出投影：

```text
AttnOut = O W_O
W_O shape: [Hq * d, D]
```

这也是一个 GEMM。对于硬件来说，QKV 投影、attention 的 `QK^T` 和 `PV`、输出投影共同构成 attention 子层的主要计算。

### 2.6 FFN：MLP、GELU 与 SwiGLU

Transformer block 的另一大计算主体是 FFN。早期结构通常为：

```text
FFN(x) = GELU(x W_1 + b_1) W_2 + b_2
```

许多现代 LLM 使用 SwiGLU 或类似门控结构：

```text
u = x W_up
g = x W_gate
FFN(x) = (SiLU(g) * u) W_down
```

其中：

```text
W_up   shape: [D, F]
W_gate shape: [D, F]
W_down shape: [F, D]
```

FFN 通常贡献了 Transformer block 中很大比例的 FLOPs。对加速器而言，FFN 是最容易用大矩阵阵列高效执行的部分；但 SwiGLU 引入的 elementwise 乘法和 SiLU 激活也需要向量单元或融合 kernel 支持。

### 2.7 LM Head 与采样

最后一层输出经过归一化后，与词表矩阵相乘得到 logits：

```text
logits = X_final W_vocab^T
logits shape: [B, vocab_size]
```

然后通过生成策略选择下一个 token：

```text
next_token = sample(logits)
```

采样可能包含 temperature、top-k、top-p、重复惩罚、约束解码等逻辑。它的 FLOPs 不大，但控制流复杂，且与 CPU/runtime 协同密切。对于低延迟推理，采样和调度开销也不能忽略。

## 3. LLM 推理的主要瓶颈

### 3.1 权重读取

每生成一个 token，都要经过所有层。对于参数量很大的模型，权重本身就是巨大的只读数据。即使计算阵列峰值很高，如果权重无法及时从 HBM 或片外 DRAM 搬入片上 buffer，阵列就会空转。

低 batch decode 场景下，权重复用不足，容易受内存带宽限制。提高 batch 可以提升权重复用，但会增加单个请求的排队延迟。因此 serving 系统需要在吞吐和延迟之间取舍。

### 3.2 KV Cache 读写

KV Cache 存储每一层历史 token 的 K 和 V。粗略估算，KV Cache 大小与以下因素成正比：

```text
KV size ~ layers * sequence_length * Hkv * d * 2 * bytes_per_element
```

这里的 `2` 表示 K 和 V 两份缓存。长上下文、多 batch、多并发请求会迅速消耗显存。decode 时每生成一个 token，还要读取历史 KV，并追加新 KV。

因此，对 LLM 推理而言，KV Cache 管理往往和算力同样重要，甚至在长上下文场景中更重要。

### 3.3 小 batch 与动态 shape

用户请求长度不同、输出长度不同、到达时间不同。推理系统面对的是动态负载，而不是固定形状的矩阵计算。硬件如果只擅长固定大矩阵，在真实服务场景中可能利用率不高。

这要求硬件和 runtime 支持：

- 变长序列。
- 动态 batch。
- prefix 复用。
- KV Cache 分页管理。
- prefill 和 decode 的不同调度策略。

### 3.4 非 GEMM 算子

LLM 中的主体计算是 GEMM，但非 GEMM 算子大量存在：

- RMSNorm / LayerNorm。
- RoPE。
- Softmax。
- SiLU / GELU。
- residual add。
- elementwise multiply。
- top-k / top-p sampling。
- MoE routing。
- gather / scatter。

如果这些算子都由通用 CPU 处理，会带来数据搬运和同步开销。如果全部写死在专用硬件里，又会牺牲模型适配能力。因此加速器通常需要矩阵单元、向量单元、特殊函数单元和可编程控制单元协同。

## 4. 推理优化技术及其具体做法

### 4.1 算子融合

算子融合的目标是减少中间结果写回片外内存。例如：

```text
RMSNorm + Quantize
MatMul + Bias + Activation
Residual Add + RMSNorm
Dequantize + MatMul
RoPE + Q/K reshape
```

如果不融合，每一步都可能产生一次 HBM 读写。融合后，中间结果可以保留在寄存器、SRAM 或片上 buffer 中。对硬件而言，这要求指令或数据流能够描述多个算子的组合，而不是只能执行孤立 GEMM。

### 4.2 FlashAttention 类优化

标准 attention 如果显式生成 `S x S` 的 attention matrix，会带来巨大显存访问。FlashAttention 的核心思路不是改变数学结果，而是通过 tiling 和 online softmax，把 attention 分块计算，并避免把完整 attention matrix 写入 HBM。

简化理解：

1. 把 Q、K、V 切成 block。
2. 每次把一小块载入片上 SRAM。
3. 在片上完成局部 `QK^T`、softmax 统计量更新和 `PV` 累积。
4. 只把最终输出写回 HBM。

这类方法对硬件的启示是：attention 加速不仅需要矩阵乘法能力，还需要足够的片上 SRAM、灵活的 block 调度、稳定的 reduction/softmax 支持和数值缩放机制。

### 4.3 PagedAttention 与 KV Cache 分页

serving 场景中，不同请求的上下文长度不同，传统连续分配 KV Cache 容易造成显存碎片。PagedAttention 借鉴虚拟内存思想，把 KV Cache 拆成固定大小的 block，通过 block table 管理逻辑序列到物理缓存页的映射。

这样可以：

- 减少显存碎片。
- 支持请求动态增长。
- 更容易复用 prefix。
- 更容易进行抢占、换入换出和 batch 合并。

硬件如果要高效支持这类机制，就不能假设 K/V 在物理内存中总是完全连续。它需要支持间接寻址、block table 访问、gather 读取和较高效的不连续内存访问。

### 4.4 Continuous Batching

传统 batching 会等一批请求凑齐后一起执行，但 LLM 请求的输出长度不同，短请求完成后会留下空位。Continuous batching 在每个 decode step 动态加入新请求，把已经结束的请求移出 batch。

这能显著提升吞吐，但也带来硬件和 runtime 问题：

- 每一步 batch 组成可能变化。
- 每个请求的 KV Cache 位置不同。
- attention 长度不同。
- 调度器需要平衡公平性、延迟和吞吐。

因此，LLM 加速器不能只看单请求 latency，也要考虑多请求 serving 下的动态调度效率。

### 4.5 Prefix Caching

很多应用有共享前缀，例如系统提示词、工具说明、RAG 模板或多轮对话中的历史上下文。Prefix caching 会缓存这些前缀的 KV Cache，新请求只需要从共享前缀之后继续 prefill 或 decode。

具体做法通常包括：

- 对 prompt 前缀做哈希。
- 查找是否已有对应 KV Cache。
- 命中后复用前缀缓存。
- 对新增 token 继续计算并追加 KV。

这对硬件没有引入新算子，但对内存管理和地址映射提出要求。缓存粒度、生命周期、引用计数和显存回收都需要 runtime 管理。

### 4.6 量化推理

量化是降低推理成本最重要的方法之一。常见路线包括：

- `FP16 / BF16`：当前通用高精度推理格式。
- `FP8`：在部分训练和推理场景中降低带宽与存储压力。
- `INT8`：较成熟的推理量化格式。
- `INT4`：主要用于 weight-only quantization，显著压缩权重。
- `KV Cache quantization`：压缩长上下文下的 K/V 缓存。

一个简化的线性量化形式为：

```text
x_int = round(x_fp / scale) + zero_point
x_fp  ~= scale * (x_int - zero_point)
```

对权重量化而言，常见粒度包括：

- per-tensor：整个张量一个 scale，硬件简单但精度较差。
- per-channel：每个输出通道一个 scale，精度更好。
- group-wise：每组若干通道共享 scale，常用于 INT4 权重量化。

硬件需要考虑：

- 是否支持 INT8/INT4 矩阵乘法。
- scale/zero point 如何读取和广播。
- 累加器使用 INT32、FP16 还是 FP32。
- dequant 是否与 matmul 融合。
- 激活是否也量化，还是只做 weight-only。
- 不同模型对量化误差的敏感性。

过度量化可能伤害数学、代码、长上下文和推理能力。因此硬件最好支持多精度混合，而不是只押注一种低精度格式。

### 4.7 Speculative Decoding

Speculative decoding 用一个较小或较快的 draft model 先生成多个候选 token，再由大模型一次性验证这些 token。若验证通过，就可以一次接受多个 token，从而减少大模型 decode 次数。

简化流程：

1. draft model 生成 `k` 个候选 token。
2. target model 对这 `k` 个 token 做并行验证。
3. 接受连续匹配的 token。
4. 如果某处不匹配，从该位置重新采样。

这种方法不改变目标模型的输出分布，但把部分串行 decode 转化为并行验证。硬件上需要注意：

- 同时部署 draft model 和 target model 的权重。
- 两个模型之间的调度和同步。
- 验证阶段的短序列 prefill-like 计算。
- 接受率不足时收益会下降。

### 4.8 GQA、MQA 与 KV Cache 压缩

标准 MHA 中每个 query head 都有自己的 K/V head。MQA 让所有 query head 共享一组 K/V，GQA 则让一组 query head 共享一组 K/V。它们的目的都是减少 KV Cache：

```text
MHA: Hkv = Hq
GQA: Hkv < Hq
MQA: Hkv = 1
```

这对 decode 特别重要，因为每一步都要读取历史 K/V。硬件上需要支持 `Hq` 和 `Hkv` 不相等的 attention 映射，不能假设 Q/K/V head 数完全一致。

### 4.9 稀疏化与 MoE 推理

MoE（Mixture of Experts）模型每个 token 只激活部分专家。例如 top-2 routing：

```text
router_logits = x W_router
selected_experts = topk(router_logits, k=2)
y = sum_i gate_i * Expert_i(x)
```

MoE 的优势是总参数容量大，但每个 token 的激活计算量相对可控。硬件难点在于：

- token 到 expert 的路由是动态的。
- 不同 expert 负载可能不均衡。
- 需要 gather/scatter。
- 多卡 expert parallel 会产生 all-to-all 通信。
- 小 expert batch 会降低矩阵阵列利用率。

因此，支持 MoE 的硬件不能只提供大 GEMM，还需要高效的路由、重排、局部 batch 聚合和片间通信能力。

## 5. 从算子到硬件模块

一个面向 LLM 的 NPU 或专用加速器通常需要以下模块。

### 5.1 矩阵计算阵列

矩阵阵列负责 QKV projection、output projection、FFN、LM head 等大部分计算。常见设计包括 systolic array、SIMT/SIMD tensor core、脉动阵列变体或 FPGA 上的可配置 MAC array。

设计关注点包括：

- 支持的矩阵 tile 大小。
- 对小 batch、小 M/N/K 的利用率。
- 是否支持 batched GEMM。
- 是否支持 INT4/INT8/FP8/FP16/BF16 混合精度。
- 累加精度和溢出处理。
- 权重预取和双缓冲。

LLM decode 中常见的矩阵形状可能较“瘦”，例如 `[B, D] x [D, F]`，其中 `B` 可能较小。硬件如果只对超大方阵优化，decode 利用率可能不理想。

### 5.2 向量与特殊函数单元

非 GEMM 算子需要向量单元支持：

- add、mul、sub、div。
- reduce sum / max。
- exp、tanh、sigmoid 或近似实现。
- reciprocal sqrt。
- compare、top-k。
- type conversion。

Softmax、RMSNorm、GELU、SiLU、RoPE、采样都依赖这些能力。实际设计中可以用查表、多项式近似或专用流水线实现特殊函数。

### 5.3 片上 SRAM 与 buffer

片上存储用于保存 tile、局部累加结果、scale、临时向量和部分 KV。由于片外 HBM/DRAM 访问能耗远高于片上访问，数据复用的关键在于能否把合适的数据留在片上。

需要考虑：

- 矩阵 tile buffer。
- double buffering。
- Q/K/V 临时 buffer。
- softmax 统计量 buffer。
- 量化 scale buffer。
- DMA 与计算流水重叠。

片上 SRAM 容量越大，attention 和 fusion 越容易做；但 SRAM 面积昂贵，容量不可能无限增加。

### 5.4 片外内存与带宽

LLM 推理高度依赖 HBM/DRAM 带宽。权重、KV Cache 和激活都需要通过片外内存传输。典型设计需要关注：

- HBM 通道数和有效带宽。
- 内存访问合并。
- bank conflict。
- 随机访问与连续访问的差异。
- KV Cache 的 block layout。
- 多租户请求下的带宽隔离。

对于长上下文模型，内存容量也同样关键。显存不够时，即使算力充足，也无法服务足够的并发请求。

### 5.5 DMA、调度器和控制核

LLM 推理不是固定流水线。不同层、不同请求、不同模型会产生不同执行图。因此需要控制逻辑：

- 搬运权重和激活。
- 发起矩阵计算。
- 调度向量算子。
- 管理 KV Cache 地址。
- 处理动态 batch。
- 与主机 CPU 或 runtime 通信。

这部分可以由嵌入式 RISC-V 核、微控制器、硬件调度器或编译器生成的命令流完成。越专用的硬件效率越高，但越需要可编程控制来适应模型变化。

### 5.6 片上互联与多芯片互联

单芯片内部需要 NoC 连接计算阵列、SRAM、DMA 和内存控制器。多芯片部署则需要高速互联：

- tensor parallel 中的 all-reduce / all-gather。
- pipeline parallel 中的激活传输。
- expert parallel 中的 all-to-all。
- KV Cache 分布式存储。

对 MoE 和超大模型而言，互联带宽与延迟会直接决定系统吞吐。硬件设计不能只看单芯片 TOPS，还要看集群级通信效率。

## 6. 不同模型结构对硬件的影响

LLM 架构仍在快速变化。硬件生命周期通常比模型迭代周期长，因此必须保留通用性和设计余量。

### 6.1 Norm 差异

不同模型可能使用：

- LayerNorm。
- RMSNorm。
- Pre-norm。
- Post-norm。
- Sandwich norm。

硬件应支持基本 reduction、scale、bias、rsqrt 和 residual fusion，而不是只固化某一种 norm 顺序。

### 6.2 Attention 变体

常见 attention 差异包括：

- MHA、MQA、GQA。
- Sliding window attention。
- Local attention。
- Global + local 混合 attention。
- Cross-attention。
- Long-context RoPE scaling。
- ALiBi 或其他位置偏置。

硬件需要至少保留以下余量：

- `Hq` 与 `Hkv` 可配置。
- head dimension 可配置。
- mask 类型可配置。
- attention block size 可配置。
- 支持不同位置编码或位置偏置。
- 支持不连续 KV Cache 地址。

### 6.3 FFN 与激活函数差异

模型可能使用：

- GELU FFN。
- SwiGLU / GeGLU。
- ReLU 或其他激活。
- 不同 FFN expansion ratio。
- shared expert + routed expert。

因此，硬件不要只支持固定的 `GEMM + GELU + GEMM`。更合理的是提供高性能 GEMM、可编程 elementwise、activation 近似和 fusion 描述能力。

### 6.4 MoE 与稀疏模型

MoE 模型对硬件的要求明显不同：

- router 需要 top-k。
- token 需要按 expert 重排。
- expert 计算需要小 batch 聚合。
- 多芯片部署需要 all-to-all。
- 负载均衡会影响利用率。

如果硬件没有 gather/scatter、动态调度和高速互联，MoE 的理论计算节省不一定能转化为实际速度。

### 6.5 Encoder-decoder 与多模态模型

虽然聊天模型多为 decoder-only，但硬件若面向更广泛 AI 工作负载，还要考虑：

- encoder-only 模型，如 BERT 类理解模型。
- encoder-decoder 模型，如 T5 类翻译/摘要模型。
- 视觉 encoder + LLM decoder 的多模态模型。
- 语音 encoder、音频 codec、视频 token 化模块。

这些模型会引入 cross-attention、卷积、patch embedding、resampler、投影层等模块。专用硬件如果完全围绕单一 decoder-only 路线设计，会在多模态和端侧应用中受限。

## 7. 硬件设计中应保留的通用性和余量

### 7.1 精度格式余量

建议支持多种精度组合：

- BF16/FP16 用于通用高精度推理。
- FP8 用于更激进的吞吐和带宽优化。
- INT8 用于成熟量化推理。
- INT4 用于 weight-only 大模型部署。
- FP32 或较高精度累加用于敏感 reduction。

同时要支持 per-channel 或 group-wise scale。否则即使矩阵单元支持 INT4，也可能因为 scale 处理效率低而无法有效运行真实量化模型。

### 7.2 Shape 与 tile 余量

模型的 `D`、`F`、`H`、`d`、层数、词表大小都不同。硬件应避免只对某一组固定维度优化。

需要考虑：

- 非 2 的幂维度。
- 不同 head dimension，如 64、80、96、128。
- 不同 FFN hidden size。
- 小 batch decode。
- 大 batch prefill。
- 长上下文 attention。
- LM head 大词表投影。

矩阵阵列可以有推荐 tile，但 runtime 和编译器应能处理边界 tile 和非整除 shape。

### 7.3 存储容量余量

KV Cache 会随上下文和并发增长。硬件设计要预留：

- 足够 HBM/DRAM 容量。
- 高带宽 K/V 读取路径。
- KV Cache 分页和压缩支持。
- prefix cache 管理空间。
- 长上下文场景下的地址位宽。

只按短上下文模型估算显存，很容易在长上下文应用中失效。

### 7.4 可编程算子余量

LLM 算子变化很快。完全固定函数硬件虽然效率高，但风险也高。更稳妥的设计是：

- 大算子用专用矩阵阵列。
- 小算子用可编程向量单元。
- 复杂流程用命令流或微码描述。
- fusion 由编译器或 runtime 配置。
- 新 activation、norm、position encoding 可通过软件升级支持。

这是一种“专用数据通路 + 可编程控制”的折中。

### 7.5 地址和数据布局余量

真实 serving 中，内存布局可能非常复杂：

- paged KV Cache。
- tensor parallel 分片权重。
- expert 分片。
- prefix cache 共享。
- 不同请求交错存储。
- 量化权重和 scale 混合布局。

硬件应支持足够灵活的 stride、offset、block table 和 gather/scatter 能力。否则 runtime 为了适配硬件需要频繁重排数据，抵消加速收益。

### 7.6 多芯片扩展余量

单卡或单芯片无法覆盖所有模型规模。设计时应考虑：

- 多芯片同步。
- collective communication。
- 权重分片。
- KV Cache 分片。
- expert parallel。
- pipeline stage 划分。

通信接口的选择会影响可扩展性。对大模型而言，集群级吞吐往往比单芯片 TOPS 更重要。

## 8. 面向 NPU/ASIC 的设计建议

如果设计一颗面向 LLM 推理的 NPU 或 ASIC，比较稳妥的方向是：

1. 以高利用率矩阵阵列为核心，但不要只优化大 batch GEMM。
2. 片上 SRAM 要服务 attention tiling、fusion 和临时 reduction。
3. 向量单元要足够强，能覆盖 norm、activation、RoPE、softmax 和采样前处理。
4. 内存系统要面向 KV Cache，而不是只面向权重流式读取。
5. 支持 BF16/FP16/FP8/INT8/INT4 多精度和灵活 scale。
6. runtime 要支持 continuous batching、paged KV、prefix cache 和 prefill/decode 分离。
7. 指令或命令流要能表达算子融合，避免每个小算子都往返 HBM。
8. 保留动态 shape、动态地址和不同 attention 结构的适配能力。
9. 多芯片互联要纳入早期设计，而不是后期补丁。
10. 编译器、kernel library 和 profiling 工具必须和硬件一起设计。

可以把 LLM 加速器理解成三层：

```text
模型层：Transformer、MoE、多模态、长上下文
系统层：serving、batching、KV Cache、调度、并发
硬件层：矩阵阵列、SRAM、HBM、NoC、互联、控制核
```

如果只优化硬件层而忽略系统层，峰值算力很可能无法兑现；如果只优化系统层而硬件缺少关键数据通路，也无法在能效上超过通用 GPU。

## 9. 一个简化的硬件执行视角

以一层 decoder block 的 decode 阶段为例，可以把硬件执行粗略分解为：

1. 从 HBM 读取当前 token hidden state。
2. 执行 RMSNorm，结果保留在片上 buffer。
3. 读取 QKV 权重，执行矩阵乘法得到 q、k、v。
4. 对 q、k 应用 RoPE。
5. 把新 k、v 写入 KV Cache。
6. 根据 block table 读取历史 K/V。
7. 分块计算 q 与 K 的点积。
8. 做 masked softmax。
9. 分块累积 `P V` 得到 attention 输出。
10. 执行 output projection。
11. residual add。
12. 执行第二个 RMSNorm。
13. 执行 FFN 的 up/gate projection。
14. 执行 SiLU 和 elementwise multiply。
15. 执行 down projection。
16. residual add，进入下一层。

这个流程说明：即使从公式看只有几行，真实硬件执行中也包含大量数据搬运、地址生成、同步、缓存管理和算子切换。LLM 加速器的难点恰恰在这些“矩阵乘法之外”的部分。

## 10. 评价指标

评价 LLM 推理硬件时，不应只看 TOPS。更有意义的指标包括：

- `TTFT`：Time To First Token，首 token 延迟，主要受 prefill 影响。
- `TPOT`：Time Per Output Token，逐 token 生成延迟，主要受 decode 影响。
- tokens/s：系统吞吐。
- tokens/s/W：能效。
- tokens/s/$：成本效率。
- 最大上下文长度。
- 最大并发请求数。
- KV Cache 容量利用率。
- 不同 batch 和序列长度下的利用率曲线。
- 模型精度损失，如 perplexity、benchmark 分数、长上下文稳定性。

硬件宣传中的峰值算力往往无法反映真实 LLM serving 表现。更可靠的方式是用典型模型、典型上下文长度、典型并发请求和真实生成长度进行端到端测量。

## 11. 总结

LLM 推理硬件加速的核心矛盾是：模型计算以矩阵乘法为主，但系统性能经常受内存、KV Cache、动态调度和非 GEMM 算子限制。优秀的 NPU 或专用加速器不能只堆 MAC 数量，还要同时解决数据搬运、片上缓存、低精度格式、attention 数据流、serving runtime 和模型结构适配。

从设计路线看，比较可持续的方向不是为某一个模型写死一条硬件流水线，而是构建一套面向 Transformer 家族的通用加速平台：

- 用矩阵阵列提供主体算力。
- 用向量和特殊函数单元覆盖非 GEMM 算子。
- 用片上 SRAM 和 HBM 数据流优化降低搬运成本。
- 用灵活地址机制支持 KV Cache 和动态 batching。
- 用多精度支持覆盖不同量化路线。
- 用可编程 runtime 和编译器适配不断变化的模型结构。

换句话说，LLM 加速不是单个算子的优化，而是一套从模型公式到硬件数据流、从单请求 latency 到多租户 serving 的系统工程。
