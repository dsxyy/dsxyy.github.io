# DSpark 与 DeepSpec：从投机解码原理到 vLLM 实测

大模型 Decode 通常一次 Target Model 前向只生成一个 token。投机解码增加一条成本更低的 proposal 生成路径：先提出多个候选 token，Target 再一次性验证，只接受从第一个位置开始连续正确的前缀。

```
Draft：  A B C D E
Target： A B C X ...

接受：A B C
拒绝：D E
补充：Target 产生的 X
```

它的核心不是减少一次前向的计算量，而是用便宜的 Draft 计算减少昂贵的 Target 串行轮数。

## 1. 投机解码不只有独立 Draft Model

投机解码的共同结构是：先用低成本方式产生 proposal，再由 Target 验证。Proposal 不一定来自一个独立小模型。

```
投机解码
├── 独立小模型
│   └── 传统 Speculative Sampling
├── 特征级草稿模型
│   └── EAGLE / EAGLE3
├── 并行块草稿模型
│   └── DFlash / DSpark
├── Target 内置多 token 预测头
│   └── Medusa / MTP
├── 无模型草稿
│   └── Prompt Lookup / N-gram
└── Target 自身简化执行
    └── Early Exit / Layer Skipping
```

传统方法使用一个更小的语言模型自回归生成草稿；Medusa 和 MTP 在 Target 内增加多个预测头；Prompt Lookup 则直接从已有上下文中查找重复 token 序列。它们的实现不同，但目标相同：让一次昂贵的 Target 验证尽可能推进多个 token。

## 2. DSpark 的执行机制

本文使用的 DSpark checkpoint 每轮最多提出 7 个草稿 token：

```
Qwen3 hidden states
        ↓
DSpark Draft Model
        ↓
最多 7 个草稿 token
        ↓
Qwen3 批量验证
        ↓
最长正确前缀 + 1 个 Target token
```

在线流水线中，本轮 Target 验证上一轮草稿，随后 DSpark 使用本轮 hidden states 准备下一轮草稿。因此不是 Target 执行两遍，而是每轮执行一次 Target 和一次较小的 Draft。

## 3. EAGLE3 与 DSpark 的流程差异

EAGLE3 和 DSpark 都会读取 Target 的多层 hidden states，但生成草稿的方式不同。

### 3.1 EAGLE3：特征级自回归草稿

```
Target 验证并产生多层 hidden states
        ↓
融合为 EAGLE3 的输入特征
        ↓
EAGLE3 预测下一个草稿特征和 token
        ↓
把刚生成的 token 反馈给 Draft
        ↓
继续自回归生成后续草稿 token
        ↓
Target 批量验证整个草稿序列
```

EAGLE3 的重点是利用 Target 的中间特征降低草稿预测难度。草稿 token 仍按前后依赖逐步生成：后一个草稿依赖前一个草稿的结果。

### 3.2 DSpark：并行块草稿

```
Target 验证并产生多层 hidden states
        ↓
拼接、投影为 DSpark 输入
        ↓
DSpark Backbone 并行计算整个草稿块
        ↓
Markov Head 顺序注入块内 token 依赖
        ↓
Target 批量验证整个草稿块
```

DSpark 把主要的 Transformer 计算放到一次并行块计算中，只将较轻量的 Markov Head 保留为顺序步骤。

```
EAGLE3：Draft 主体偏自回归，逐步向后生成草稿
DSpark：Draft 主体并行计算整个块，再轻量串联 token 依赖
```

两者都属于 learned speculator，都需要额外 checkpoint，也都会占用额外显存。最终性能不只由算法类别决定，还取决于 Draft checkpoint 质量、草稿长度、接受率、硬件和业务内容。

## 4. DSpark 模型结构：三个容易混淆的数字

本次 Target 是 36 层 Qwen3-4B。DSpark 从 Target 的 5 个位置读取 hidden states：

```json
"target_layer_ids": [1, 9, 17, 25, 33]
```

每份 hidden state 的最后一维为 2560。vLLM 中先拼接，再通过训练好的线性层压回 2560：

```
5 × [T, 2560]
      ↓ concat
[T, 12800]
      ↓ Linear(12800 → 2560)
[T, 2560]
```

融合结果再送入 DSpark 自己的 Transformer：

```json
"num_hidden_layers": 5,
"block_size": 7
```

必须区分：

- Qwen3 Target：36 层。
- 从 Target 读取：5 份 hidden states。
- DSpark 自身：5 层 Transformer。
- 每轮草稿块：最多 7 个 token。

读取 Target 的层数与 Draft 自身层数是独立参数，本 checkpoint 中只是恰好都为 5。

## 5. DSpark 的并行骨干与 Markov Head

DSpark 的草稿生成可简化为：

```python
head_hidden = draft_backbone(...)
draft_tokens = sample_sequentially(head_hidden)
```

五层 Draft Transformer 先并行计算 7 个位置的基础特征，随后 LM Head 产生基础 logits，Markov Head 再引入块内依赖：

```python
for position in range(7):
    bias = markov_head(previous_token)
    token = argmax(base_logits[:, position] + bias)
    previous_token = token
```

昂贵的 Draft Transformer 主体已经并行计算；顺序部分只运行较轻量的 Markov Head。后一个草稿 token 依赖前一个 token，因此这一小段必须逐位置推进。

## 6. DeepSpec 与 vLLM 的职责边界

DeepSpec 是训练、评估和参考执行工具，适合验证：

- checkpoint 是否能正确加载；
- proposal 与 verification 是否符合算法预期；
- 输出是否与普通 Target 解码一致；
- 每轮提出、接受和推进多少 token；
- Confidence Head 是否能动态截断草稿。

vLLM 是生产服务运行时，负责 Paged KV Cache、Continuous Batching、CUDA Graph、高性能算子、在线调度和 OpenAI 兼容 API。

```
DeepSpec：证明模型与算法路径正确
vLLM：证明在线服务是否真正更快
```

这一区分非常重要。checkpoint 能加载，不代表执行语义已经对齐。hidden state 位置、token shift、position、RMSNorm 前后 residual 和 Draft KV Cache 更新约定只要有一处不一致，程序可能正常运行，但接受率和输出都会异常。

## 7. 为什么需要 oracle

在 EAGLE3 适配实验中，初始 vLLM 路径只有 7.28% 接受率，且输出不一致。DeepSpec oracle 证明 checkpoint 本身正常后，最终发现两项元数据差异：

1. DeepSpec 使用零基 decoder layer id，而 vLLM 使用已完成层数，`[1, 9, 17, 25, 33]` 需要转换为 `[2, 10, 18, 26, 34]`。
2. DeepSpec 保存 RMSNorm 之前的 residual，因此 vLLM 需要 `norm_before_residual=false`。

修复后 EAGLE3 吞吐从低于基线变为提升 33.2%。这说明 oracle 的价值不是“再跑一次模型”，而是隔离 checkpoint 问题与运行时适配问题。

## 8. 在 vLLM 中部署 DSpark

本次使用的运行时为：

```
vLLM 0.23.1rc1.dev771+gf329ce405
CUDA 12.9 nightly image
```

核心配置如下：

```bash
vllm serve /workspace/models/Qwen3-4B \
  --served-model-name Qwen3-4B \
  --dtype bfloat16 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.85 \
  --speculative-config '{
    "method":"dspark",
    "model":"/workspace/models/dspark_qwen3_4b_block7",
    "num_speculative_tokens":7,
    "attention_backend":"FLASH_ATTN",
    "draft_sample_method":"greedy"
  }'
```

性能对比必须固定 vLLM build、Target、dtype、提示词、采样参数、输出长度和并发量，只改变 speculative config。

## 9. 如何理解投机指标

```
raw acceptance
= accepted draft tokens / proposed draft tokens

accepted / verification
= 每轮平均接受的 Draft token 数

generated / verification
= accepted / verification + 1 个 Target token
```

Raw acceptance 不是最终优化目标。较短的 K 通常接受率更高，却可能每轮推进不足；较长的 K 接受率较低，却可能产生更高吞吐。最终应看 TPOT、端到端延迟和服务吞吐。

## 10. RTX 4090 单请求实测

固定英文提示词、并发 1、生成 128 token：

| 指标 | Qwen3-4B Baseline | DSpark K=7 |
|---|---:|---:|
| TTFT | 12.20 ms | 13.11 ms |
| TPOT | 9.53 ms | 4.33 ms |
| E2E | 1.22 s | 0.56 s |
| 输出吞吐 | 104.67 token/s | 227.16 token/s |

DSpark 获得约 2.17 倍吞吐。每轮平均推进约 3.10 个 token，但实际加速没有达到 3.10 倍，因为还存在 Draft Model、Markov Head、采样和调度开销。TTFT 略有增加也符合预期：首 token 阶段无法从后续块级验证获益。

## 11. 加速比为什么取决于生成内容

相同模型和 `K=7` 在不同提示词上的结果：

| 任务 | Baseline | DSpark | 加速 | Raw acceptance |
|---|---:|---:|---:|---:|
| 英文解释 | 104.76 | 257.56 | 2.46x | 37.70% |
| 中文问答 | 104.69 | 139.11 | 1.33x | 12.07% |
| 代码 | 104.65 | 435.48 | 4.16x | 72.79% |
| 数学 | 104.71 | 417.37 | 3.99x | 68.18% |
| 摘要 | 102.96 | 243.73 | 2.37x | 35.29% |

单位为 token/s。代码和数学结构更强，Draft 更容易连续预测正确；中文样本接受率较低，加速收益明显下降。

因此不能笼统地说“DSpark 能加速某模型多少倍”。结论必须绑定：

```
Target + Draft checkpoint + 硬件 + K + 采样方式 + 业务数据分布
```

## 12. Confidence Head 与动态草稿长度

固定 K 表示无论是否有信心，每轮都提出相同数量的草稿。DSpark checkpoint 还包含 Confidence Head，用于预测各位置被接受的可能性。系统使用人工选定的 threshold，截取首个低置信度位置之前的前缀：

```
confidence:  [0.92, 0.81, 0.73, 0.41, 0.30]
threshold:    0.50
valid length: 3
```

DeepSpec 中使用同一个阈值 0.5：中文平均只提交 1.16 个草稿，代码仍提交 6.92 个。调度器不需要识别语言或任务类型，只观察每轮 token 级置信度。

但 DeepSpec 参考路径会先计算完整 block，再截断低置信度后缀，所以该实验只验证动态 proposal 机制，不能直接证明服务性能提升。

当前测试的 vLLM nightly 支持固定 K DSpark，但没有接入 Confidence Head。源码会主动跳过：

```python
# confidence_head is not wired into inference yet
skip_substrs = ["mask_embedding", "confidence_head"]
```

因此当前 vLLM 支持的是 DSpark 核心路径，而不是 DeepSpec 中的全部能力。

## 13. 并发性能与显存代价

20 个同提示词请求、每请求 128 token：

| 并发 | Baseline | DSpark K=7 | 加速 |
|---:|---:|---:|---:|
| 1 | 104.78 token/s | 262.71 token/s | 2.51x |
| 4 | 393.93 token/s | 1010.91 token/s | 2.57x |
| 8 | 652.76 token/s | 1590.76 token/s | 2.44x |

Continuous Batching 提高多个活动请求的 GPU 利用率；DSpark 减少 Target 串行验证轮数。两者作用不同，可以叠加。不过该实验请求少、提示词完全相同且启用了 Prefix Caching，只用于说明机制，不能代表生产容量。

同一 RTX 4090、`gpu_memory_utilization=0.85`：

| 指标 | Baseline | EAGLE3 | DSpark |
|---|---:|---:|---:|
| 模型加载显存 | 7.56 GiB | 7.84 GiB | 8.76 GiB |
| GPU KV Cache | 84,416 token | 78,896 token | 65,936 token |
| 8192-token 并发估算 | 10.30x | 9.63x | 8.05x |

DSpark 相比 Baseline 增加约 1.20 GiB 模型显存，KV Cache token 容量下降约 21.9%。本实验中它比 EAGLE3 更快，但资源代价也更高。EAGLE3 使用 K=3、DSpark 使用 K=7，这属于可用 checkpoint 的实际效果比较，不是严格的算法同参数比较。

## 14. 一套可复用的评估流程

1. 使用官方参考实现建立 correctness 和 acceptance oracle。
2. 核对 hidden state、position、token shift、RMSNorm residual 和 Draft KV Cache 语义。
3. 使用同一运行时 build 建立普通解码基线。
4. 固定模型、采样参数、提示词、输出长度和并发，只改变 speculative config。
5. 同时观察 TTFT、TPOT、E2E、吞吐和显存容量。
6. 使用真实业务提示词分布，不以单个样本或 raw acceptance 下结论。
7. 区分正确性实验、微基准和生产容量测试。

## 15. 结论

DSpark 的价值来自“块级草稿 + Target 批量验证”。在本次 Qwen3-4B + RTX 4090 实验中：

- 固定长度 DSpark 已能通过 vLLM 在线部署；
- 单请求固定样本吞吐约提升 2.17 倍；
- 不同内容上的加速范围为 1.33 倍到 4.16 倍；
- 并发 8 的同质请求实验仍保留约 2.44 倍优势；
- KV Cache token 容量相比 Baseline 下降约 21.9%；
- 当前 vLLM 尚未接入 Confidence Head 动态长度。

工程上最重要的不是某个固定加速数字，而是先用 DeepSpec 证明算法和 checkpoint 正确，再用 vLLM 与真实业务负载验证服务收益，并把显存、并发容量和实现边界一起纳入判断。
