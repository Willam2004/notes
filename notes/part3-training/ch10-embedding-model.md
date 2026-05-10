# 第10章 构建文本嵌入模型

## 核心概念

> 嵌入模型是语义搜索、聚类、RAG 等应用的"地基"——学会自己构建和微调嵌入模型。

### 10.1 嵌入模型

回顾（第2章已介绍）：嵌入模型将文本压缩为固定长度的稠密向量，捕捉语义信息。

**嵌入模型的核心能力**：
- 语义相似度高的文本 → 向量距离近
- 语义不同的文本 → 向量距离远

**通用嵌入模型 vs 领域嵌入模型**：

| 维度 | 通用模型 (all-MiniLM) | 领域模型 |
|------|----------------------|---------|
| 训练数据 | 通用文本 | 特定领域 |
| 领域表现 | 一般 | 优秀 |
| 开发成本 | 零（直接用） | 需标注数据 + 微调 |

### 10.2 对比学习（Contrastive Learning）

嵌入模型训练的核心方法。

**基本思想**：通过对比"相似对"和"不相似对"来学习好的表示。

```
正样本对：(相似文本, 相似文本) → 拉近距离
负样本对：(相似文本, 不相关文本) → 推远距离

损失函数：InfoNCE / 多重负样本排序损失
```

**关键概念**：
- **Anchor**：基准样本
- **Positive**：与 Anchor 语义相近的样本
- **Negative**：与 Anchor 语义不同的样本

**困难负样本**：选择"看起来相关但实际不同"的负样本，训练效果更好。

### 10.3 SBERT（Sentence-BERT）

原始 BERT 的句子表示质量差（直接取平均或 [CLS] 效果不好）。SBERT 通过**孪生网络 + 对比学习**解决这个问题。

```
句子 A → BERT → 向量 A ─┐
                          ├→ 余弦相似度 → 对比损失
句子 B → BERT → 向量 B ─┘

(两个 BERT 共享参数 = 孪生网络)
```

**SBERT 训练数据来源**：
- SNLI（自然语言推理）：蕴含/矛盾/中性
- NLI 数据集
- 问答对
- Paraphrase（释义对）

### 10.4 构建嵌入模型

从零构建嵌入模型的完整流程：

```
1. 选择基础模型（BERT / RoBERTa / DeBERTa）
2. 准备训练数据（正负样本对）
3. 选择损失函数
4. 训练
5. 评估（BEIR / MTEB benchmark）
```

**常用损失函数**：

| 损失函数 | 数据格式 | 适用场景 |
|----------|---------|---------|
| MultipleNegativesRankingLoss | (anchor, positive) | 大量正样本对 |
| ContrastiveLoss | (anchor, positive/negative) + 标签 | 有正负标注 |
| TripletLoss | (anchor, positive, negative) | 三元组数据 |
| CosineSimilarityLoss | (句子A, 句子B) + 相似度分数 | 连续相似度标注 |

### 10.5 微调嵌入模型

在已有嵌入模型基础上，用领域数据继续训练。

**微调策略**：
- **全量微调**：更新所有参数，效果好但成本高
- **LoRA 微调**：只训练低秩矩阵，成本低
- **冻结底层 + 微调顶层**：保留通用语义，学习领域特征

**评估嵌入质量**：
- 信息检索指标（NDCG、MRR）
- 语义相似度相关性（Spearman）
- MTEB benchmark 排行榜

### 10.6 无监督学习

无标注数据时，如何训练嵌入模型？

**方法**：
- **TF-IDF 检索 + 对比学习**：用 TF-IDF 检索相似文档作为正样本对
- **数据增强**：对同一文档做回译、同义词替换生成正样本
- **TPL（Triplet Loss from Pooling）**：从文档池中自动构建三元组

## 代码示例

```python
# 使用 sentence-transformers 微调嵌入模型
from sentence_transformers import SentenceTransformer, losses
from sentence_transformers.datasets import SentenceLabelDataset
from torch.utils.data import DataLoader

# 加载基础模型
model = SentenceTransformer("all-MiniLM-L6-v2")

# 准备训练数据（正样本对）
train_examples = [
    InputExample(texts=["如何学习Python", "Python入门教程"], label=1.0),
    InputExample(texts=["机器学习算法", "监督学习方法"], label=1.0),
    InputExample(texts=["如何学习Python", "今天天气不错"], label=0.0),
]

# 多重负样本排序损失（只需正样本对）
train_dataloader = DataLoader(
    [InputExample(texts=[e.texts[0], e.texts[1]]) for e in train_examples if e.label == 1.0],
    batch_size=16,
)
train_loss = losses.MultipleNegativesRankingLoss(model)

# 训练
model.fit(
    train_objectives=[(train_dataloader, train_loss)],
    epochs=3,
    warmup_steps=100,
)

# 评估
from sentence_transformers.evaluation import InformationRetrievalEvaluator

# evaluator = InformationRetrievalEvaluator(queries, corpus, relevant_docs)
# model.evaluate(evaluator)

# 域适应微调
# 用领域文档的 TF-IDF 检索结果自动生成训练对
from sentence_transformers.datasets import DenoisingAutoEncoderDataset

# train_dataset = DenoisingAutoEncoderDataset(domain_documents)
# 用 TSDAE 方法进行无监督训练
```

## 个人思考

1. 何时需要自己训练嵌入模型？（通用模型效果差时，有领域数据时）
2. 嵌入维度如何选择？维度越高越好吗？（高维=更多细节，但更难训练、更慢）
3. MTEB 排行榜上的模型选择——关注中文表现还是整体表现？
