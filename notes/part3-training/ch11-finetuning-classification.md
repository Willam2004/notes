# 第11章 为分类任务微调表示模型

## 核心概念

> 从"用别人的模型"到"训练自己的模型"——微调让预训练模型适应你的特定任务。

### 11.1 监督分类

#### 微调预训练的 BERT 模型

核心思路：在 BERT 顶部添加分类头，用标注数据端到端训练。

```
输入文本 → BERT 编码器 → [CLS] 向量 → 线性分类层 → Softmax → 类别
              ↑                        ↑
         预训练参数（可微调）      新增参数（随机初始化）
```

**微调 vs 特征提取**：

| 方式 | 做法 | 效果 | 成本 |
|------|------|------|------|
| 特征提取 | 冻结 BERT，只训练分类头 | 一般 | 低 |
| 全量微调 | BERT + 分类头一起训练 | 最佳 | 高 |
| 部分微调 | 冻结底层，微调顶层 + 分类头 | 较好 | 中 |

#### 冻结层（Layer Freezing）

BERT 的不同层捕捉不同级别的特征：
- **底层**：通用语言特征（词形、语法）→ 冻结
- **顶层**：任务相关特征（语义、领域知识）→ 微调

```
BERT 层 1-6:  冻结 ❄️  (通用特征)
BERT 层 7-10: 微调 🔥  (领域特征)
分类头:       从头训练 🆕
```

**冻结策略的选择**：
- 数据量少 → 冻结更多层（防止过拟合）
- 数据量多 → 解冻更多层（充分学习）
- 领域差异大 → 解冻更多层

### 11.2 少样本分类

当标注数据极少（每类 8-64 条）时，SetFit 是最佳选择（第4章已介绍，这里深入原理）。

**SetFit 的训练流程**：

```
少量标注数据 (每类 8 条)
  → 第1步: 对比学习预训练
     同类样本 → 正样本对
     不同类样本 → 负样本对
     微调 sentence-transformer
  → 第2步: 训练分类头
     用微调后的模型生成嵌入
     在嵌入上训练逻辑回归/SVM
```

**SetFit 为什么有效？**
- 对比学习将同类样本聚拢、异类样本推远
- 即使只有 8 条/类，通过组合能生成大量训练对
- 嵌入空间更适合分类

### 11.3 基于掩码语言建模的继续预训练（Domain Adaptation）

当目标领域与预训练数据差异大时（如医疗、法律、金融），先做领域适应再做任务微调。

```
通用 BERT
  → 继续预训练（用领域无标注文本 + MLM 任务）
  → 领域 BERT
  → 监督微调（用领域标注数据 + 分类任务）
  → 最终模型
```

**MLM（Masked Language Modeling）**：随机遮蔽 15% 的词元，让模型预测被遮蔽的词。

**何时需要继续预训练？**
- 领域术语多、通用模型"看不懂"
- 有大量无标注领域文本
- 标注数据太少，直接微调不够

### 11.4 命名实体识别（NER）

为文本中每个词元分配实体标签（人名、地名、组织名等）。

```
输入:  "张 三 在 北 京 创 办 了 公 司"
标签:  B-PER I-PER O B-LOC I-LOC O  O  O  O  O
```

**NER 的 BERT 微调方式**：
- 在每个词元的输出上加分类层
- 使用 BIO 标注体系
- 序列标注任务（Token Classification）

**NER 评估指标**：
- 精确匹配（完整实体才算对）
- Precision / Recall / F1

## 代码示例

```python
# BERT 文本分类微调
from transformers import AutoTokenizer, AutoModelForSequenceClassification, Trainer, TrainingArguments
from datasets import Dataset

tokenizer = AutoTokenizer.from_pretrained("bert-base-chinese")
model = AutoModelForSequenceClassification.from_pretrained(
    "bert-base-chinese", num_labels=2
)

def tokenize(batch):
    return tokenizer(batch["text"], padding=True, truncation=True, max_length=128)

train_dataset = Dataset.from_dict({
    "text": ["这个产品很好", "质量太差了", "非常满意", "不推荐"],
    "label": [1, 0, 1, 0],
}).map(tokenize, batched=True)

training_args = TrainingArguments(
    output_dir="./results",
    num_train_epochs=3,
    per_device_train_batch_size=16,
    learning_rate=2e-5,
    weight_decay=0.01,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
)
trainer.train()

# SetFit 少样本分类
from setfit import SetFitModel, Trainer, TrainingArguments as SetFitArgs

model = SetFitModel.from_pretrained("sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2")
args = SetFitArgs(num_epochs=3, batch_size=16)
trainer = Trainer(model=model, args=args, train_dataset=train_dataset)
trainer.train()

# 冻结层示例
for name, param in model.named_parameters():
    if "encoder.layer" in name:
        layer_num = int(name.split("encoder.layer.")[1].split(".")[0])
        if layer_num < 6:  # 冻结前6层
            param.requires_grad = False

# NER 微调
from transformers import AutoModelForTokenClassification

ner_model = AutoModelForTokenClassification.from_pretrained(
    "bert-base-chinese", num_labels=9  # B-PER, I-PER, B-LOC, I-LOC, B-ORG, I-ORG, O, ...
)
```

## 个人思考

1. 微调时的学习率选择为什么这么重要？（太大→灾难性遗忘，太小→学不动）
2. 冻结多少层的经验法则？（数据量 < 1k→冻结 2/3；1k-10k→冻结 1/2；> 10k→全量）
3. NER 在中文场景的分词边界问题如何处理？
