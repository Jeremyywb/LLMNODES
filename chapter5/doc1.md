# 千问系列72B模型微调与双T4部署解析

让我以一段代码开始吧：

```python
llm = vllm.LLM(
        cfg.model.backbone_path,
        # quantization="awq",
        tensor_parallel_size=2,
        gpu_memory_utilization=0.99,
        trust_remote_code=True,
        dtype="half",
        enforce_eager=True,
        max_model_len=2048,
        disable_log_stats=True,
        cpu_offload_gb=8,
        swap_space=1,
        device="cuda",
        max_num_seqs=20,
        enable_prefix_caching=True
    )
```

让我分析一下这段代码是如何使能在Kaggle双卡T4上运行Qwen-72B模型的。

这段代码实际上是使用vLLM库来加载和运行大型语言模型，并应用于一个教育数据集(EEDI)上的任务。从配置来看，确实有几个关键的优化设置让这个巨大的模型能够在资源受限的T4 GPU上运行：

### 关键优化配置点

1. **张量并行化**：

   ```python
   tensor_parallel_size=2
   ```



   代码利用了张量并行技术，将模型分割到两个T4 GPU上同时运行，这是最基础的分布策略。
2. **GPU内存优化**：

   ```python
   gpu_memory_utilization=0.99
   ```


   将GPU内存使用率设为99%，充分利用可用GPU内存。
3. **CPU卸载**：

   ```python
   cpu_offload_gb=8
   ```


   将8GB的模型参数卸载到CPU内存中，需要时再加载到GPU。这对于T4这种VRAM较小(16GB)的GPU至关重要。
4. **交换空间**：

   ```python
   swap_space=1
   ```


   设置了1GB的磁盘交换空间，提供额外的虚拟内存。
5. **模型长度限制**：

   ```python
   max_model_len=2048
   ```


   将上下文窗口限制在2048 tokens，避免处理过长序列消耗过多资源。
6. **半精度计算**：

   ```python
   dtype="half"
   ```


   使用FP16半精度格式加载模型，相比FP32可减少一半内存占用。
7. **前缀缓存**：

   ```python
   enable_prefix_caching=True
   ```

   启用前缀缓存优化以加速批处理推理。
8. **强制即时模式**：

   ```python
   enforce_eager=True
   ```


   使用即时执行模式而非图形编译模式，可能有助于减少初始编译时的内存需求。
9. **序列数量限制**：

   ```python
   max_num_seqs=20
   ```

   限制了批处理中的最大序列数，防止过多的并行请求占用资源。

### 为什么能跑起来？

Qwen-72B能在双T4上运行的关键在于vLLM的优化策略结合上述配置：

1. **内存效率优化**：vLLM采用了连续批处理和KV缓存复用等技术，极大减少了推理过程中的内存需求。
2. **分布式计算**：通过张量并行将模型分布在两张GPU上，有效解决了单卡内存不足的问题。
3. **CPU-GPU协同工作**：通过CPU卸载功能，将不常用的参数存储在CPU内存中，需要时再加载到GPU，实现了计算资源的合理调度。
4. **仅执行推理**：代码仅用于推理（没有训练或微调操作），并使用低温度采样(temperature=0.0)和极短的生成长度(max\_tokens=1)，进一步减轻了计算负担。
5. **批量处理优化**：通过批处理和前缀缓存，减少了重复计算，提高了吞吐量。

值得注意的是，此配置可能是在"可以运行"和"有效运行"之间做了权衡。虽然模型能够加载并执行，但推理速度可能会较慢，特别是在处理大量数据时。代码中设计的任务是简单的分类任务(只需生成1个token进行选择)，这也降低了资源需求。

总之，这个配置巧妙地结合了多种内存优化和分布式计算技术，使得在计算资源有限的环境下也能运行如此庞大的模型。





根据运行日志(需要使用2张T4运行模型得到日志)，我来分析Qwen-72B在Kaggle双卡T4上的实际运行情况：

### 加载过程分析

1. **模型加载时间**：
   * 完整的模型加载耗时约5分42秒（从"Loading safetensors checkpoint shards"开始到结束）
   * 这表明模型采用了分片存储（9个分片），每个分片加载时间在30-45秒
2. **内存使用情况**：

   ```
   Loading model weights took 11.9423 GB
   ```


   * 日志显示加载模型权重占用了约12GB内存，这与预期相符，因为使用了AWQ量化技术
3. **GPU资源分配**：

```
   # GPU blocks: 325, # CPU blocks: 409
   Maximum concurrency for 2048 tokens per request: 2.54x
```

   * vLLM为模型分配了325个GPU内存块和409个CPU内存块
   * 每个2048 tokens请求的最大并发度为2.54倍

### 推理性能分析

1. **推理速度**：

```
   Processed prompts: 100%|█| 12/12 [00:40<00:00, 3.34s/it]
```

   * 处理12个提示总共花了40秒，平均每个提示处理时间约3.34秒
   * 估计输入吞吐量约165 tokens/秒（这是输入处理速度，不是生成速度）
2. **总运行时间**：

```
   Wall time: 7min 27s
```

   * 整个过程（包括模型加载和推理）总共耗时7分27秒
3. **使用了AWQ量化**：

```
   WARNING 12-08 18:57:34 config.py:321] awq quantization is not fully optimized yet. The speed can be slower than non-quantized models.
```

   * 日志显示使用了AWQ量化，并警告说该量化方法尚未完全优化，速度可能比非量化模型慢
4. **注意到使用Xformers后端**：

```
   INFO 12-08 18:57:35 selector.py:115] Using XFormers backend.
```

   * 由于T4 GPU是Turing架构，不支持FlashAttention-2，系统自动选择了Xformers作为注意力机制后端

### 部署优化分析

1. **量化技术**：
   代码中只提到了`quantization="awq"`的注释，但日志显示确实使用了AWQ量化，这是4位量化技术，可以大幅减少模型的内存占用
2. **P2P通信限制**：

```
   Custom allreduce is disabled because your platform lacks GPU P2P capability or P2P test failed
```

   * T4 GPU的P2P通信能力受限，导致自定义allreduce被禁用，这可能会影响多GPU协同工作效率
3. **VRAM使用情况**：
   虽然代码中设置了`gpu_memory_utilization=0.99`，但日志没有直接显示VRAM使用率，根据加载数据可推测：
   * T4 GPU每卡16GB VRAM
   * 12GB用于模型权重
   * 剩余约20GB（两卡合计）用于KV缓存、激活值和中间计算结果



### 模型占用分析

**1. 显存(VRAM)占用：**

* 每张T4 GPU有16GB VRAM，共32GB (双卡)
* 日志显示：`Loading model weights took 11.9423 GB`，这表示模型权重本身占用了约12GB
* vLLM分配了`# GPU blocks: 325`个GPU内存块，用于KV缓存和计算
* 推测总显存占用：每卡约14-15GB (接近设定的`gpu_memory_utilization=0.99`，约16GB×0.99=15.84GB)

**2. 内存(RAM)占用：**

* 日志显示分配了`# CPU blocks: 409`个CPU内存块
* 设置了`cpu_offload_gb=8`，表示约8GB模型参数被卸载到CPU内存
* 预估总RAM占用：约10-12GB (包括卸载的模型参数和工作内存)

**3. 硬盘占用：**

* 日志显示加载了9个模型分片：`Loading safetensors checkpoint shards: 100% Completed | 9/9`
* 设置了`swap_space=1`，表示额外1GB硬盘空间用作交换
* AWQ量化后的模型大小预估：约18-20GB (基于9个分片的加载时间和普通的AWQ量化比例)
* 实际硬盘占用：约20-22GB (包括模型参数和交换空间)

**4. 量化压缩比：**

* 原始72B参数模型理论大小：~144GB (FP16格式)
* AWQ量化后约18-20GB，压缩比约为7-8倍

### 推理时间分析

日志中的`Wall time: 7min 27s`是**整个执行过程**的总时间，包括：

1. **模型加载时间**：约5分42秒

```
   Loading safetensors checkpoint shards: 100% Completed | 9/9 [05:42<00:00, 38.00s/it]
```
2. **实际推理时间**：约40秒

```
   Processed prompts: 100%|█| 12/12 [00:40<00:00, 3.34s/it]
```
3. **其他开销**：约1分钟(初始化、资源分配、结果处理等)

所以，7分27秒是**完整执行时间**，其中大部分(76%)用于模型加载，实际推理只占了约9%的时间。

这表明如果需要多次使用该模型，保持模型常驻内存会大大提高效率。12个样本的小批量只需40秒完成推理，平均每个样本3.34秒，吞吐量约为18个样本/分钟。

如果是处理大规模数据集，模型加载的一次性成本会被分摊，整体效率会提高。例如，处理1000个样本可能需要约60分钟(模型加载6分钟+推理54分钟)，而不是7分钟×1000÷12=583分钟。



### 实际性能结论

1. **单样本处理时间**：约3.34秒/样本
   * 这比我之前预估的0.5-1秒慢，主要是因为:
     1. 模型使用了AWQ量化，日志中明确提示量化可能导致性能下降
     2. 实际推理上下文长度可能比预期长（样本包含数学题和思考过程）
2. **吞吐量**：约18个样本/分钟
   * 这对于资源受限的T4 GPU来说已经是不错的性能
3. **量化效果**：
   * AWQ量化成功将72B参数模型压缩到可以在双T4上运行
   * 但量化导致了一定的性能损失
4. **加载与推理时间对比**：
   * 模型加载时间(5分42秒)占总运行时间(7分27秒)的大部分
   * 实际推理只占约40秒

在这种配置下，Qwen-72B确实能在双T4上运行，但大部分时间花在模型加载上，实际推理速度约为3.34秒/样本。这对于批量处理任务或非实时应用来说是可接受的，但对于需要实时响应的应用仍然太慢。