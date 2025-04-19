# 目标：拉近 query 和 answer 之间 embedding 距离

难点：
	- 一般问题query到answer 的分布式上下文语义可能好学，但是需要经过推理再到answer的可能难度比较大
	- 考虑利用 cot 过程作为锚点 间接达到目的

### 继续预训练？
是否需要领域侧 继续 预训练
	- MLM ？
		- [MASK] token 随机遮掩 15%（不连续），好像是一次性的，且更像是一种“知识注入”？
		- 可以试试，但是可能不够。需要其他任务配合。
	- 动态MLM？
		- 更像大模型类的预训练，知识学习的可能更充分，升级版的MLM
		- 也是不够的，可能 answer或者cot部分语义与query更近了（如果作为预训练材料的话）

	-SpanBERT  SBO？
		- 与MLM相比，他是连续的块

### 任务微调

- 对比学习（constrastive learning - CTL) 本质是 正样本与query的距离
- NSP？
	- 看起来与目标相近：拼接两个句子，预测后者是否前者的后一个句子
	- 任务是分类任务，且负样本为其他无关文档：1. 任务难度低，2. 引入噪声


借鉴下面论文的思路，分2~3步增强当前任务效果---[How to Fine-Tune BERT for Text Classification?]（https://arxiv.org/pdf/1905.05583）

- step1：任务数据或领域内容继续预训练--- 数学推理数据 query+cot 进行MLM或者span预训练--->测试训练后效果是否有增强，例如线下召回或线上分与基线对比
- step2：多相似领域内容，多任务预训练--->略
- step3：任务相关微调---> 对比学习

******************************
- **背景**：数学推理背景题，面试被问到query到answer相似度可能只有50%，可能只是随机，bge等检索怎么做到有效召回

- **疑点**：
    - 推理背景下的query到answer真的只是如此弱的关联吗（相似度衡量）？
    - 用相似度考察retriever的能力是否有意义？
  
- **目的**：以下实验终极目的是为了增强query到answer的embedding逻辑推理语义关联。
- **难点**：本实验的query到answer即使在LLM中需要经过COT才能显著增强，故而query到answer可能注定是弱上下文关系

1. 继续预训练的任务为了知识注入（这条LLM思想是否适用于PTM也即所有预训练模型）
2. 第一条成立的情况下，如何证明知识注入是有效的
3. 抛弃第一条论述（第二条关联第一条此时也无需理会），实验目的在于模型如何学会COT
4. 注意消融实验
5. ...

******************************

文章对应观点片段
```We propose a general solution to fine-tune
the pre-trained BERT model, which includes
three steps: (1) further pre-train BERT on
within-task training data or in-domain data;
(2) optional fine-tuning BERT with multitask learning if several related tasks are available; (3) fine-tune BERT for the target task.
```

来源:

如何微调bert适应下游任务
[How to Fine-Tune BERT for Text Classification?]（https://arxiv.org/pdf/1905.05583）

预训练的任务分类
[Pre-trained models for natural language processing: A survey]（https://xuanjing-huang.github.io/files/PTM.pdf）
