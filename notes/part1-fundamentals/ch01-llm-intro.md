# 第1章 大语言模型简介

## 核心概念

### LLM 的定义

大语言模型（Large Language Model，LLM）是在**海量文本数据**上训练的、参数规模达到数十亿至数万亿量级的深度学习模型。核心任务是理解和生成人类语言。

三个关键要素：
- **大（Large）**：巨大的参数量 + 庞大的训练数据量
- **语言（Language）**：处理自然语言（理解、分析、摘要、翻译、生成）
- **模型（Model）**：一个复杂函数，计算给定前文后下一个最可能出现的词

### 类比理解

> 大模型就像一个在人类所有文字资料里不间断阅读了数年的"超级读书人"。它不仅记住了内容，还学会了语言的规律、逻辑的结构、知识的关联。

### LLM vs 传统程序

| 特性 | 传统程序 | 大模型 |
|------|----------|--------|
| 工作方式 | 预设规则匹配 | 统计规律预测 |
| 输入 | 精确关键词 | 自然语言对话 |
| 输出 | 固定结果 | 生成式回答 |
| 灵活性 | 只能处理预设场景 | 泛化能力强 |
| 缺点 | 无 | 幻觉问题（一本正经地胡说八道） |

## 关键图解

### 1. 语言模型进化阶梯

```
词袋模型 (BoW) → Word2Vec → RNN/LSTM → 注意力机制 → Transformer → GPT/BERT → ChatGPT
 (2013前)       (2013)     (1997-2014)    (2015)      (2017)       (2018-2019)   (2022)
```

每一代都是在前一代基础上的重大突破。

### 2. Transformer 三种架构路线

```
                    Transformer
                   /     |      \
          Encoder-Only  Enc-Dec  Decoder-Only
              |           |          |
            BERT        T5         GPT系列
         (阅读理解)   (翻译摘要)  (对话生成)
```

- **Encoder-Only**：擅长理解文本深层含义（分类、情感分析）
- **Decoder-Only**：擅长根据上文生成下文（当前主流对话模型）
- **Encoder-Decoder**：既能理解又能生成（翻译、摘要等转换任务）

### 3. 大模型训练三阶段

```
预训练 (Pre-training) → 监督微调 (SFT) → 对齐 (RLHF)
   "读万卷书"            "学会沟通"       "品德教育"
```

**预训练**：在海量无标签文本上通过"预测下一个词"学习语法、语义和知识
**监督微调**：用高质量"指令-回答"数据训练，学会遵循指令
**对齐（RLHF）**：基于人类反馈的强化学习，使输出符合人类价值观（有用、诚实、无害）

## 要点摘要

### 1. 核心原理：预测下一个词

> 给定前面所有的词，预测下一个词最可能是什么。

```
输入：今天天气真的很___
模型预测：好（42%）、差（18%）、热（15%）、棒（12%）……
```

就这一句话，撑起了整个大模型的世界。模型通过考虑**所有上下文**后做预测，连续生成一整段有意义的文字。

### 2. 涌现能力（Emergent Ability）

当模型规模足够大时，会自发出现训练目标中**从未明确教过**的能力：

- 写诗、解数学题、编程
- 逻辑推理、知识关联
- 类比水分子组合形成浪潮 —— 单个分子没什么特别，组合起来出现宏观特性

**关键结论**：大模型的能力不是线性增长的，而是在某个规模门槛之后**突然跃升**。

### 3. 深度推理（System 2）

Kahneman 《思考，快与慢》框架：
- **System 1（快思考）**：直觉、自动、快速（早期大模型擅长）
- **System 2（慢思考）**：深思熟虑、逻辑推理（o1、o3、DeepSeek-R1 等新模型方向）

DeepSeek-R1（2025.1）用 RLVR（可验证奖励强化学习）让模型学会自我检验推理过程。

### 4. AI 发展三起三落

| 浪潮 | 驱动力 | 结果 |
|------|--------|------|
| 第一次 | 机器学习取代手工规则 | 遇到瓶颈→第一次寒冬 |
| 第二次 | 深度学习 + GPU | 算力不足→第二次寒冬 |
| 第三次 | Transformer 架构 | 大模型时代（当前） |

### 5. 开源 vs 闭源模型

- **闭源**：ChatGPT、Claude、Gemini 等，通过 API 调用，能力领先
- **开源**：LLaMA、Qwen、DeepSeek 等，权重公开，社区活跃

## 代码示例

### 调用第一个大模型 API

```python
from openai import OpenAI

client = OpenAI(
    api_key="your_api_key_here",
    base_url="https://api.deepseek.com"  # 可替换为其他兼容接口
)

messages = [
    {"role": "system", "content": "你是一个幽默风趣的AI助手，擅长用生活比喻解释复杂概念。"},
    {"role": "user", "content": "用一个有趣的比喻，解释什么是神经网络的'权重'。"}
]

response = client.chat.completions.create(
    model="deepseek-chat",
    messages=messages,
    temperature=0.8,   # 0=保守稳定，1=富有创意
    max_tokens=300
)

answer = response.choices[0].message.content
usage = response.usage
print(answer)
print(f"Token 用量：输入 {usage.prompt_tokens}，输出 {usage.completion_tokens}")
```

### 感受 temperature 的效果

```python
question = "用一个词形容今天的心情"

for temp in [0.0, 0.7, 1.5]:
    response = client.chat.completions.create(
        model="deepseek-chat",
        messages=[{"role": "user", "content": question}],
        temperature=temp,
        max_tokens=20
    )
    result = response.choices[0].message.content.strip()
    print(f"temperature={temp}: {result}")
```

运行结果：
- `temperature=0.0`：每次几乎相同，非常稳定
- `temperature=0.7`：有变化，但都比较合理
- `temperature=1.5`：可能出现奇怪但有趣的词

## 个人思考

<!-- 学习过程中的疑问、联想和见解 -->

### 思考题

1. "预测下一个词"能算作真正的"理解"吗？
2. 涌现能力让人联想到哪些生活中的现象？（蚂蚁群体、城市交通、社交网络……）
3. 为什么 Transformer 的并行计算能力如此重要？（GPU 架构的契合）
4. 开源模型和闭源模型，未来谁会占主导？
