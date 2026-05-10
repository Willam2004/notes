# 第2章 词元和嵌入

> 核心问题：计算机只认识数字，大模型如何"读懂"人类文字？

## 核心概念

### 2.1 文字 → 数字的基础

计算机存储的一切都是 0 和 1。文字到数字的转换经历了两层：

| 编码方案 | 年代 | 能力 | 例子 |
|---------|------|------|------|
| ASCII | 1963 | 128个字符（英文） | A → 65, a → 97 |
| Unicode | 现代 | 15万+字符（全球） | 你 → U+4F60, 中文字需3字节 |

### 2.2 Token（词元）

大模型不按"字"也不按"词"处理，而是按 **Token** 处理。

**Token 是介于字符和词之间的文本片段**，由 BPE（字节对编码）算法自动学习得到。

#### BPE 算法流程

```
1. 初始化：词元库 = 所有基本字符
2. 统计：找出语料中相邻出现频率最高的一对词元
3. 合并：将这对词元合并成新词元，加入词元库
4. 迭代：重复 2-3，直到词元库达到预设大小（3万～10万）
```

结果：**高频词独占一个 Token，低频词被拆成多个 Token**。

#### 中英文 Token 效率差异

| 语言 | 大约比例 | 说明 |
|------|---------|------|
| 英文 | ~0.75 词 = 1 Token | 更高效 |
| 中文 | ~1.5-2 字 = 1 Token | 消耗更多 Token |

同样信息量的中文，消耗的 Token 数通常比英文更多。国内模型（Qwen、DeepSeek）的中文 Tokenizer 效率高得多。

#### Token 的两个实用知识

1. **Token 是 API 计费单位** — 不是按字数收费
2. **上下文窗口** — 模型一次能"看到"的最大 Token 数（硬性上限）

> "中间丢失"效应（Lost in the Middle）：内容被塞在超长上下文中间时，模型关注度会下降。

### 2.3 词嵌入（Word Embedding）

为什么不直接用 Token ID？因为 ID 编号是**任意**的 —— ID 100 和 101 可能毫无关联，而语义相近的词 ID 可能相差几千。

**词嵌入的核心思想**：把每个 Token 映射为一个**高维向量**（768维或更高），让语义相近的词在向量空间中距离也近。

```
词元ID空间（无语义）：          嵌入向量空间（有语义）：
  Token#345 "猫" ──┐            🐱猫 ←近→ 🐶狗
  Token#892 "狗" ──┤ 数字距离     🚗汽车 ←近→ 🚂火车
  Token#156 "汽车" ─┘ 无意义      😊快乐 ←近→ 😃喜悦
```

**语义上的相似，变成了空间上的距离。**

### 2.4 余弦相似度

衡量两个向量方向的接近程度，结果 -1 到 1：

- 接近 1 = 非常相似
- 接近 0 = 无关
- 接近 -1 = 相反

### 2.5 词向量的神奇算术

```
向量("国王") - 向量("男人") + 向量("女人") ≈ 向量("女王")
```

词嵌入不仅记住了词，还编码了**词与词之间的关系结构**。

### 2.6 位置嵌入（Positional Embedding）

词嵌入有一个局限：**无法感知词序**。"人咬狗"和"狗咬人"的词嵌入完全一样。

解决：为每个位置创建专属向量，**加到**词元嵌入上。

```
最终输入 = 词元嵌入向量 + 位置编码向量
```

#### Transformer 原始位置编码（正弦编码）

```
偶数维度: PE(pos, 2i) = sin(pos / 10000^(2i/d))
奇数维度: PE(pos, 2i+1) = cos(pos / 10000^(2i/d))
```

直觉理解：
- 低维部分：高频波，编码精细位置差异（像二进制低位）
- 高维部分：低频波，编码大致位置范围（像二进制高位）
- 利用三角函数性质，蕴含了**相对位置**信息

### 2.7 旋转位置嵌入（RoPE）

当前主流方案（Llama 等模型使用）。

核心区别：**不再"加"入位置信息，而是"乘"入（旋转）**。

```
绝对位置编码: 输出 = 词嵌入 + 位置向量
旋转位置编码: 输出 = 旋转(词嵌入, 角度=位置×θ)
```

RoPE 的优势：
1. 天然的相对位置编码 — 绝对位置在点积中被消去
2. 良好的外推性 — 比训练时更长的文本也能处理

### 2.8 上下文感知嵌入

静态词嵌入（Word2Vec）的局限：同一个词不管语境如何，向量都一样。

- "我去 bank（银行）取钱"
- "我坐在 bank（河边）上"

Transformer 通过**自注意力（Self-Attention）**根据前后文动态调整词的向量 → 第3章详解。

## 关键图解

### 分词过程 = 拆解积木

```
"大语言模型真有趣！"  →  [大语言] [模型] [真] [有趣] [！]
  完整玩具车                标准积木块（Token）
```

### BPE 迭代合并

```
初始: [大] [语] [言] [模] [型]
第1轮: [大] [语] [言] [模型]    ← "模"+"型" 频率最高，合并
第2轮: [大] [语言] [模型]       ← "语"+"言" 频率最高，合并
第3轮: [大语言] [模型]          ← "大"+"语言" 频率最高，合并
```

### 从 Token ID 到嵌入向量

```
Token ID (整数) → 查表 → 嵌入向量 (高维浮点数数组)
     345        →          [0.12, -0.85, 0.33, ...768维]
```

### 位置编码方式对比

| 方案 | 方式 | 外推性 | 代表模型 |
|------|------|--------|---------|
| 正弦编码 | 加法 | 差 | 原始 Transformer |
| 可学习编码 | 加法 | 差 | BERT、GPT-2 |
| RoPE | 旋转（乘法） | 好 | LLaMA、Qwen、DeepSeek |

## 代码示例

### 实验 2-A：用 tiktoken 切分 Token

```python
import tiktoken

enc = tiktoken.encoding_for_model("gpt-4")

texts = [
    "Hello world",
    "Transformer",
    "人工智能",
    "大语言模型",
    "中文比英文消耗更多Token吗？",
]

for text in texts:
    tokens = enc.encode(text)
    decoded = [enc.decode([t]) for t in tokens]
    print(f"原文：{text!r}")
    print(f"  切分：{decoded}")
    print(f"  Token 数：{len(tokens)}\n")
```

输出：
```
原文：'人工智能'
  切分：['人工', '智能']
  Token 数：2

原文：'大语言模型'
  切分：['大', '语言', '模型']
  Token 数：3
```

### 实验 2-B：计算语义相似度

```python
import numpy as np
from openai import OpenAI

client = OpenAI(
    api_key="your_api_key_here",
    base_url="https://api.deepseek.com"
)

def get_embedding(text):
    response = client.embeddings.create(
        model="text-embedding-ada-002",
        input=text
    )
    return response.data[0].embedding

def cosine_similarity(a, b):
    a, b = np.array(a), np.array(b)
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

words = ["猫", "狗", "鱼", "汽车", "火车", "飞机", "快乐", "悲伤", "愤怒"]
embeddings = {w: get_embedding(w) for w in words}

query = "猫"
similarities = [
    (w, cosine_similarity(embeddings[query], embeddings[w]))
    for w in words if w != query
]
similarities.sort(key=lambda x: x[1], reverse=True)

for rank, (word, sim) in enumerate(similarities, 1):
    bar = "█" * int(sim * 20)
    print(f"  {rank}. {word:4s}  {sim:.3f}  {bar}")
```

输出：
```
  1. 狗     0.891  █████████████████
  2. 鱼     0.743  ██████████████
  3. 快乐   0.412  ████████
  4. 悲伤   0.388  ███████
  5. 汽车   0.301  ██████
```

## 个人思考

### 思考题

1. 128K Token 的上下文窗口约能容纳多少页中文文档？（提示：一页约 500 字）
2. 为什么不直接用 Unicode 编号训练模型，而要引入 Token + 词嵌入？
3. "快乐"和"悲伤"的余弦相似度出乎意料地高，为什么？（都属于情感词，共享大量语义维度）
4. RoPE 的"旋转"思路和直接加位置编码相比，本质优势是什么？

### 知识串联

```
文字 → Unicode编码 → BPE分词 → Token ID → 词嵌入(Embedding) → +位置编码 → 送入Transformer
                          ↑                                      ↑
                    本章重点 2.1-2.2                     本章重点 2.3-2.7
```
