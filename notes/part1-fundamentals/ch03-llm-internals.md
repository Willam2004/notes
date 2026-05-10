# 第3章 LLM的内部机制

> 核心问题：Transformer 这个"工厂"如何加工向量"原材料"，提炼出语言理解能力？

## 核心概念

### 3.1 整体架构：Transformer 块堆叠

大语言模型 = **多个结构相同的 Transformer 块** 堆叠而成（几十到上百层）。输入向量自下而上流经堆栈，每层提炼一次信息。

每个 Transformer 块包含两个核心子模块：
1. **多头自注意力（Multi-Head Self-Attention）** — "博采众长"
2. **前馈神经网络（FFN）** — "消化吸收"

### 3.2 自注意力机制（Self-Attention）

核心任务：对序列中每个词元，计算它与所有其他词元的关联程度，有选择地聚合信息。

#### QKV 模型类比

| 向量 | 类比 | 作用 |
|------|------|------|
| **Q**（Query） | 搜索查询 | 当前词元发出的"问题" |
| **K**（Key） | 网页标题/标签 | 每个词元的"索引" |
| **V**（Value） | 网页内容 | 每个词元的实际"信息" |

#### 计算四步

```
1. 生成 QKV:  输入向量 × W_Q/W_K/W_V → Q, K, V 三个向量
2. 注意力分数: Q · K^T（点积）→ 原始关联分数
3. 缩放归一化: 分数 / √d_k → Softmax → 注意力权重（和为1）
4. 加权求和:  权重 × V → 融合上下文的新向量
```

用代码表示：

```python
# Self-Attention 核心计算
scores = Q @ K.T / sqrt(d_k)    # 点积 + 缩放
weights = softmax(scores)         # 归一化为权重
output = weights @ V              # 加权求和
```

### 3.3 多头注意力（Multi-Head Attention）

不是用一组 QKV，而是用**多组**（如 12 组）不同的权重矩阵并行计算。

```
         输入向量
        /    |    \
    Head1  Head2  ... Head12
    (Q1,K1,V1) (Q2,K2,V2)  (Q12,K12,V12)
        \    |    /
       拼接 → 线性变换 → 最终输出
```

每个"头"捕捉一种不同类型的关联：
- 语法结构关系
- 语义关联
- 指代关系
- 位置关系
- ...

### 3.4 前馈神经网络（FFN）

在注意力聚合后，FFN 对每个词元的向量进行**非线性变换**。

```
结构: 线性层 → 激活函数(GeLU/ReLU) → 线性层

作用: "信息加工站"
  注意力 = "博采众长"（收集信息）
  FFN    = "消化吸收"（深度加工）
```

### 3.5 残差连接 + 层归一化

这两个"辅助组件"是深度网络能训练的关键。

**残差连接**：
```
输出 = 模块处理(输入) + 输入    ← 输入直接"跳"过去
```
- 解决梯度消失问题
- 提供信息"高速公路"，保证原始信息不丢失
- 至少学到恒等变换，网络加深不会变差

**层归一化**：
```
残差连接后 → 将向量数值调整到标准分布范围
```
- 稳定训练过程
- 加速收敛
- 让每层在稳定数据环境下工作

#### Transformer 子层完整结构

```
输入 → [多头自注意力] → 残差连接+ → 层归一化 → [FFN] → 残差连接+ → 层归一化 → 输出
         ↑                                    ↑
         └─────── 输入直接跳过 ───────────────┘
```

### 3.6 混合专家模型（MoE）

MoE 将标准 FFN 替换为**多个专家网络 + 门控网络**。

```
输入向量 → 门控网络(路由器) → 选择 Top-K 个专家
                         ↓
            Expert1 ─┐
            Expert3 ─┼→ 加权组合 → 输出
            (从8个中选2个，其他不激活)
```

优势：总参数量大，但单次计算只激活部分专家 → "更大"与"更快"兼得。

代表：DeepSeek-V3/R1、GPT-5 等先进模型。

MoE 的权衡：
- **训练**：更高效，但需负载均衡防止专家"闲忙不均"
- **推理**：速度快，但所有专家参数都需加载到显存 → 硬件门槛高

### 3.7 状态空间模型（SSM / Mamba）

Transformer 的固有瓶颈：自注意力计算复杂度 **O(n²)**，长序列代价大。

SSM（以 Mamba 为代表）：
- 计算复杂度接近 **O(n)**（线性）
- 通过循环机制处理序列
- 处理长序列效率远超 Transformer
- 是否成为主流尚未定论

### 3.8 缩放法则（Scaling Law）

模型性能与三个因素呈幂律关系：
- **模型参数量（N）**
- **训练数据量（D）**
- **计算量（C）**

**Chinchilla 缩放定律**：为达最佳计算效率，参数量和数据量应**同步增长**。

```
错误策略: 大参数 + 少数据（"大脑袋饿肚子"）
正确策略: 参数量 ≈ 数据量 同比例扩展（"Chinchilla 最优"）
```

### 3.9 涌现能力（Emergent Abilities）

规模达到临界点后，能力从随机猜测水平**突然跃升**。

典型涌现能力：
| 能力 | 说明 |
|------|------|
| 上下文学习 | 几个例子就能执行新任务（GPT-3 标志） |
| 思维链 | 一步步推理解决复杂逻辑问题 |
| 多步算术 | 小模型算不对，大模型能完成 |

类比：水在 99°C 是液体，100°C 突然沸腾 — 规模的量变引起质变。

## 关键图解

### Transformer 块内部结构

```
┌─────────────────────────────────┐
│         Transformer 块          │
│                                 │
│  ┌───────────────────────────┐  │
│  │   多头自注意力 (MHA)      │  │
│  │   Q·K^T/√d → softmax → V │  │
│  └─────────┬─────────────────┘  │
│            ↓                    │
│     残差连接 + 层归一化          │
│            ↓                    │
│  ┌───────────────────────────┐  │
│  │   前馈网络 (FFN)          │  │
│  │   线性→GeLU→线性          │  │
│  └─────────┬─────────────────┘  │
│            ↓                    │
│     残差连接 + 层归一化          │
│                                 │
└─────────────────────────────────┘
         ↓ (重复 N 次)
```

### Transformer vs MoE 对比

```
标准 Transformer:              MoE:
  输入 → FFN → 输出             输入 → 路由器 → Expert1 →┐
  (所有参数都参与计算)                  ├────────→ 加权 → 输出
                                       → Expert3 →┘
                                 (只有部分专家被激活)
```

## 代码示例

### 手写 Self-Attention

```python
import torch
import torch.nn.functional as F
import math

def self_attention(Q, K, V):
    """
    Q, K, V: (batch, seq_len, d_k)
    """
    d_k = Q.size(-1)

    # 1. 计算注意力分数
    scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(d_k)

    # 2. Softmax 归一化
    weights = F.softmax(scores, dim=-1)

    # 3. 加权求和
    output = torch.matmul(weights, V)

    return output, weights

# 示例
seq_len, d_k = 6, 64
Q = torch.randn(1, seq_len, d_k)
K = torch.randn(1, seq_len, d_k)
V = torch.randn(1, seq_len, d_k)

output, attn_weights = self_attention(Q, K, V)
print(f"输入: ({seq_len}, {d_k})")
print(f"输出: {output.shape}")
print(f"注意力权重矩阵:\n{attn_weights[0].detach().numpy().round(2)}")
```

### 多头注意力

```python
class MultiHeadAttention(torch.nn.Module):
    def __init__(self, d_model=512, n_heads=8):
        super().__init__()
        self.n_heads = n_heads
        self.d_k = d_model // n_heads

        self.W_Q = torch.nn.Linear(d_model, d_model)
        self.W_K = torch.nn.Linear(d_model, d_model)
        self.W_V = torch.nn.Linear(d_model, d_model)
        self.W_O = torch.nn.Linear(d_model, d_model)

    def forward(self, x):
        batch, seq_len, d_model = x.shape

        # 生成 QKV 并分头
        Q = self.W_Q(x).view(batch, seq_len, self.n_heads, self.d_k).transpose(1, 2)
        K = self.W_K(x).view(batch, seq_len, self.n_heads, self.d_k).transpose(1, 2)
        V = self.W_V(x).view(batch, seq_len, self.n_heads, self.d_k).transpose(1, 2)

        # 每个头独立计算注意力
        scores = torch.matmul(Q, K.transpose(-2, -1)) / math.sqrt(self.d_k)
        weights = F.softmax(scores, dim=-1)
        attn = torch.matmul(weights, V)

        # 拼接所有头 + 线性变换
        attn = attn.transpose(1, 2).contiguous().view(batch, seq_len, d_model)
        return self.W_O(attn)

mha = MultiHeadAttention()
x = torch.randn(1, 10, 512)
print(f"输入: {x.shape} → 输出: {mha(x).shape}")
```

## 个人思考

### 思考题

1. 为什么点积能衡量两个向量的相似度？（方向越一致，点积越大）
2. 为什么要除以 √d_k？（防止 d_k 大时点积值过大，Softmax 梯度消失）
3. MoE 的门控网络如何训练？（负载均衡损失函数，防止专家闲忙不均）
4. Chinchilla 定律对你的实际工作有什么指导意义？（选模型时要看训练数据量是否匹配）
5. SSM/Mamba 能完全替代 Transformer 吗？各自适用什么场景？

### 知识串联

```
第1章: LLM 概览 → "大" + "语言" + "模型"
第2章: Token + 嵌入 → 文字变向量
第3章: Transformer → 向量如何被加工（本章）
  ├── 自注意力: 词元间信息交流（QKV → 权重 → 加权求和）
  ├── 多头注意力: 多角度并行捕捉关联
  ├── FFN: 非线性深度加工
  ├── 残差+归一化: 让深度网络可训练
  ├── MoE: 稀疏激活，更大更快
  └── Scaling Law: 规模越大，能力越强，涌现质变
```
