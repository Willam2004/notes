# 第4章 文本分类

## 核心概念

文本分类是 NLP 最常见的任务，为输入文本分配标签/类别。本章同时探讨**表示模型**（BERT）和**生成模型**（GPT）两种路线。

### 4.1 基于表示模型的分类（BERT）

利用 BERT 的 `[CLS]` token 输出作为文本语义表示，后接分类头（线性层 + Softmax）。

### 4.2 零样本分类（Zero-shot）

无需任何标注数据。核心思路：将分类问题转化为**自然语言推理（NLI）**任务。

```
待分类文本 = "前提"(Premise)
类别描述   = "假设"(Hypothesis)

判断哪个假设最可能被前提蕴含 → 即为分类结果
```

### 4.3 少样本分类 — SetFit

SetFit（Sentence Transformer Fine-tuning）：仅用**少量标注样本**（8条起）微调 sentence-transformer。

```
少量标注数据 → 对比学习生成正负样本对 → 微调嵌入模型 → 分类
```

### 4.4 基于嵌入的分类

```
文本 → 嵌入模型 → 向量 → 传统分类器（逻辑回归/SVM）
```

### 4.5 基于生成模型的分类

- **T5**：将分类统一为文本生成任务（输入文本 → 输出标签）
- **ChatGPT/大模型**：通过 Prompt 直接输出类别

## 方法对比

| 方法 | 标注数据 | 适用场景 | 核心优势 |
|------|---------|---------|---------|
| 特定任务模型 | 大量 | 情感分析、NER | 精度高、推理快 |
| 零样本分类 | 无 | 快速原型、标签探索 | 零标注成本 |
| 少样本 SetFit | 极少（8条） | 标注资源受限 | 少数据高精度 |
| 嵌入+分类器 | 中等 | 灵活场景 | 嵌入可复用 |
| 生成模型 | 无/少量 | 通用分类 | 灵活性最强 |

**核心启示**：数据稀缺时，零样本和少样本方法性价比远高于从头训练。

## 代码示例

```python
# 情感分析（特定任务模型）
from transformers import pipeline
classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")
result = classifier("This movie was absolutely fantastic!")
# → [{'label': 'POSITIVE', 'score': 0.9998}]

# 零样本分类
classifier = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")
result = classifier(
    "The new smartphone has an amazing camera.",
    ["technology", "sports", "politics", "entertainment"]
)
# → 按概率排序的类别列表

# SetFit 少样本分类
from setfit import SetFitModel, Trainer
from datasets import Dataset

train_data = Dataset.from_dict({
    "text": ["great product", "terrible experience", "loved it", "waste of money"],
    "label": [1, 0, 1, 0]
})
model = SetFitModel.from_pretrained("sentence-transformers/all-MiniLM-L6-v2")
trainer = Trainer(model=model, train_dataset=train_data)
trainer.train()
```

## 个人思考

1. 实际项目中如何选择分类方法？（先试零样本 → 不行用 SetFit → 量大再微调）
2. 零样本分类的 NLI 方法有什么局限？（类别描述的质量直接影响结果）
3. 生成模型做分类的稳定性如何保证？（temperature、多轮验证）
