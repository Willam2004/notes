# 第5章 文本聚类和主题建模

## 核心概念

文本聚类：将无标签文档自动分组。主题建模：从文档集合中自动提取主题。核心工具是 **BERTopic**（本书作者之一 Maarten Grootendorst 创建）。

### 5.1 文本聚类通用流程（四步管线）

```
文档 → ① 嵌入(sentence-transformers) → ② 降维(UMAP) → ③ 聚类(HDBSCAN) → ④ 检查簇
```

| 步骤 | 工具 | 作用 |
|------|------|------|
| 嵌入 | sentence-transformers | 文本 → 语义向量 |
| 降维 | UMAP | 保留结构的维度压缩 |
| 聚类 | HDBSCAN | 基于密度自动聚类，无需预设簇数 |
| 检查 | 可视化/统计 | 分析簇质量 |

### 5.2 BERTopic 框架详解

```
文档 → Embedding → UMAP 降维 → HDBSCAN 聚类 → c-TF-IDF 提取主题词
```

**c-TF-IDF**（class-based TF-IDF）：将同一簇内所有文档合并，衡量一个词对**整个簇（主题）**的重要性（而非单篇文档）。

### 5.3 BERTopic 的模块化设计

每一步都是可替换的"乐高积木块"：

```
可替换 Embedding 模型 → OpenAI / Cohere 嵌入
可替换降维算法     → t-SNE / PCA
可替换聚类算法     → K-Means
可加入文本生成模型 → 用 LLM 为主题生成可读描述
```

核心哲学：**模块化 > 端到端黑箱**（更可控、可调试、可解释）

## 代码示例

```python
# BERTopic 基本使用
from bertopic import BERTopic

docs = [...]  # 文档列表
topic_model = BERTopic(language="chinese", calculate_probabilities=True)
topics, probs = topic_model.fit_transform(docs)

# 查看主题
print(topic_model.get_topic_info().head())  # 主题概览
print(topic_model.get_topic(0))             # 主题0的关键词

# 可视化
topic_model.visualize_topics()
topic_model.visualize_barchart()

# 模块化自定义
from bertopic import BERTopic
from bertopic.representation import KeyBERTInspired
from sentence_transformers import SentenceTransformer
from umap import UMAP
from hdbscan import HDBSCAN

topic_model = BERTopic(
    embedding_model=SentenceTransformer("all-MiniLM-L6-v2"),
    umap_model=UMAP(n_neighbors=15, n_components=5, metric='cosine'),
    hdbscan_model=HDBSCAN(min_cluster_size=15, cluster_selection_method='eom'),
    representation_model=KeyBERTInspired(),
)
topics, probs = topic_model.fit_transform(docs)

# 动态主题建模（追踪主题随时间变化）
topics_over_time = topic_model.topics_over_time(docs, timestamps)
topic_model.visualize_topics_over_time(topics_over_time)
```

## 个人思考

1. BERTopic 的模块化设计思路可以借鉴到其他 NLP 管线中
2. HDBSCAN vs K-Means 的选择标准？（不需要预知簇数 vs 速度）
3. 如何评估聚类质量？（轮廓系数、簇间距离、人工抽检）
