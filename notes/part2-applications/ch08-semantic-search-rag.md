# 第8章 语义搜索与RAG

## 核心概念

> 让 LLM "翻书找答案"而非"凭记忆回答"——RAG 是目前大模型落地最成熟的技术模式。

### 8.1 语义搜索与 RAG 技术全景

**关键词搜索 vs 语义搜索**：

| 维度 | 关键词搜索 | 语义搜索 |
|------|-----------|---------|
| 匹配方式 | 字面精确/模糊匹配 | 语义向量相似度 |
| 理解能力 | 无（"苹果"分不清水果还是公司） | 有（根据上下文理解含义） |
| 查询方式 | 必须包含关键词 | 自然语言描述即可 |
| 典型工具 | Elasticsearch、BM25 | 向量数据库 + 嵌入模型 |

**语义搜索流程**：

```
离线：文档 → 切片 → 嵌入模型 → 向量 → 存入向量数据库
在线：查询 → 嵌入模型 → 向量 → 向量数据库检索 → 排序 → 结果
```

### 8.2 语言模型驱动的语义搜索实践

#### 向量数据库

存储和检索高维向量的专用数据库。

| 数据库 | 特点 | 适用场景 |
|--------|------|---------|
| FAISS | Meta 开源，纯库无服务 | 本地/原型验证 |
| Chroma | 轻量，Python 友好 | 快速原型 |
| Milvus | 分布式，高性能 | 生产环境 |
| Pinecone | 全托管云服务 | 免运维 |

#### 文档处理管线

```
原始文档
  → 加载（PDF/HTML/Markdown）
  → 切片（Chunking）
  → 嵌入（Embedding）
  → 存储（Vector Store）
```

**切片策略**（关键决策）：

| 策略 | 方式 | 优缺点 |
|------|------|--------|
| 固定长度 | 每 512 token 一段 | 简单但可能切断语义 |
| 按段落/章节 | 按文档结构切 | 保持语义完整 |
| 递归切分 | 按分隔符层级切 | 兼顾大小和语义 |
| 语义切分 | 嵌入后按相似度断开 | 最佳但成本高 |

#### 检索增强

- **重排（Re-ranking）**：初筛后用交叉编码器精排
- **混合检索**：关键词 + 语义双路召回，取并集再排序
- **查询改写**：用 LLM 将用户查询改写为更适合检索的形式

### 8.3 RAG（检索增强生成）

RAG = 检索 + 生成。先检索相关文档，再将文档作为上下文喂给 LLM 生成回答。

```
用户提问
  → 查询嵌入 → 向量检索 → 取出 Top-K 相关文档片段
  → 将 [文档片段 + 问题] 组装成 Prompt
  → LLM 基于文档生成回答（有据可依）
```

**RAG vs 微调 vs 纯 Prompt**：

| 维度 | 纯 Prompt | RAG | 微调 |
|------|-----------|-----|------|
| 知识更新 | 手动改 Prompt | 更新文档库即可 | 需重新训练 |
| 成本 | 低 | 中 | 高 |
| 领域知识 | 受限于训练数据 | 检索外部知识 | 写入模型参数 |
| 可解释性 | 低 | 高（可溯源） | 低 |
| 幻觉风险 | 高 | 较低 | 中 |

**RAG 的典型问题与优化**：

| 问题 | 原因 | 优化方案 |
|------|------|---------|
| 检索不到相关内容 | 切片太大/太小 | 调整切片大小 + 重叠 |
| 回答无关 | 检索质量差 | 加重排、混合检索 |
| 上下文太长 | 塞了太多文档 | 精选 Top-3~5 片段 |
| 回答不一致 | 模型忽略检索内容 | 强化 Prompt 指令 |

## 代码示例

```python
# 语义搜索基础流程
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("all-MiniLM-L6-v2")

# 文档嵌入
docs = ["机器学习是AI的子领域", "深度学习使用神经网络", "今天天气不错"]
doc_embeddings = model.encode(docs)

# 查询嵌入 + 检索
query = "什么是人工智能"
query_embedding = model.encode(query)

# 余弦相似度
similarities = np.dot(doc_embeddings, query_embedding) / (
    np.linalg.norm(doc_embeddings, axis=1) * np.linalg.norm(query_embedding)
)
top_idx = np.argmax(similarities)
print(f"最相关: {docs[top_idx]} (相似度: {similarities[top_idx]:.3f})")

# RAG 完整示例（伪代码）
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1. 文档加载与切片
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_text(document_text)

# 2. 嵌入与存储
embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")
vectorstore = Chroma.from_texts(chunks, embeddings, persist_directory="./db")

# 3. 检索
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})
relevant_docs = retriever.invoke("如何优化模型性能？")

# 4. 生成回答
context = "\n\n".join([doc.page_content for doc in relevant_docs])
prompt = f"""基于以下文档回答问题。如果文档中没有答案，请说"我不确定"。

文档：
{context}

问题：如何优化模型性能？
"""
# response = llm.invoke(prompt)
```

## 个人思考

1. RAG 的检索质量瓶颈——"垃圾进垃圾出"，文档预处理的重要性常被低估
2. 混合检索（关键词 + 语义）在中文场景下效果如何？
3. RAG 系统的评估指标：检索准确率、答案相关性、幻觉率
4. 什么时候该用 RAG，什么时候该微调？（动态知识用 RAG，能力提升用微调）
