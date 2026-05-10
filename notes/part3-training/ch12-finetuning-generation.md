# 第12章 微调生成模型

## 核心概念

> 从"通用助手"到"领域专家"——微调生成模型的三步曲：预训练、监督微调、偏好对齐。

### 12.1 LLM 训练三步走

```
第1步: 预训练 (Pre-training)
  → 海量无标注文本 → 预测下一个词 → 基座模型
  → 成本: 数百万美元，数万 GPU

第2步: 监督微调 (SFT, Supervised Fine-Tuning)
  → 高质量指令-回答对 → 学会"听指令" → 对话模型
  → 成本: 数千~数万美元

第3步: 偏好对齐 (Alignment / Preference Tuning)
  → 人类偏好反馈 → 学会"什么回答更好" → 安全有用的模型
  → 成本: 数千美元
```

本章重点：第2步和第3步。

### 12.2 监督微调（SFT）

用 **(指令, 回答)** 对微调基座模型，让它学会遵循指令。

**训练数据格式**：
```json
{"instruction": "将以下文本翻译为英文", "input": "今天天气很好", "output": "The weather is nice today."}
```

**SFT 的关键决策**：

| 决策 | 选项 | 推荐 |
|------|------|------|
| 训练哪些参数 | 全量 vs LoRA | 数据多→全量，数据少→LoRA |
| 学习率 | 1e-5 ~ 1e-4 | 从小开始，用余弦退火 |
| Epoch 数 | 1-5 | 1-3（过拟合风险大） |
| 批大小 | 受 GPU 限制 | 梯度累积模拟大 batch |

**SFT 的常见陷阱**：
- 数据质量 > 数据数量（100条高质量 >> 10000条噪声数据）
- 过拟合导致多样性丧失（模型只会回答训练集格式）
- 指令多样性不足（模型泛化能力差）

### 12.3 使用 QLoRA 进行指令微调

**LoRA（Low-Rank Adaptation）**：不修改原始参数，而是添加小规模可训练矩阵。

```
原始权重 W (d×d) → 冻结 ❄️
LoRA 矩阵 A (d×r) × B (r×d) → 训练 🔥  (r << d)

输出 = W·x + (A·B)·x
```

**r（秩）** 通常取 8~64，参数量仅为全量微调的 0.1%~1%。

**QLoRA** = 量化 + LoRA：

```
原始模型 (FP16)
  → 4-bit 量化加载 (NF4)
  → 冻结 ❄️
  → 添加 LoRA 适配器 (FP16) 🔥
  → 训练 LoRA 参数
  → 合并回模型
```

**QLoRA 的三大创新**：
1. **NF4 量化**：4-bit 正态分布量化，信息损失最小
2. **双重量化**：量化参数本身也量化，进一步节省显存
3. **分页优化器**：GPU 显存不足时自动卸载到 CPU

| 方法 | 显存需求 (7B模型) | 训练速度 |
|------|------------------|---------|
| 全量微调 (FP16) | ~28 GB | 快 |
| LoRA (FP16) | ~16 GB | 较快 |
| QLoRA (4-bit) | ~6 GB | 中等 |

### 12.4 评估生成模型

生成模型的评估比分类模型复杂得多——没有唯一正确答案。

**评估方法**：

| 方法 | 描述 | 优缺点 |
|------|------|--------|
| 人工评估 | 人类打分 | 准确但昂贵、慢 |
| 自动指标 | BLEU/ROUGE/BERTScore | 快但不完全反映质量 |
| LLM-as-Judge | 用强模型评判弱模型 | 平衡成本和质量 |
| Benchmark | MMLU/HumanEval/GSM8K | 标准化但可能泄露 |

**评估维度**：
- **有用性**（Helpfulness）：回答是否有用
- **准确性**（Correctness）：事实是否正确
- **安全性**（Safety）：是否产生有害内容
- **流畅性**（Fluency）：语言是否自然

### 12.5 偏好调优、对齐

让模型学会"什么回答更好"，而不仅是"怎么回答"。

```
问题: "解释量子计算"
回答A: 通俗易懂，例子贴切  → 人类偏好: 更好 ✓
回答B: 充满术语，晦涩难懂  → 人类偏好: 较差 ✗

偏好调优: 让模型学会生成 A 而非 B
```

### 12.6 RLHF：使用奖励模型实现偏好评估自动化

**RLHF（Reinforcement Learning from Human Feedback）**：

```
第1步: 收集偏好数据
  同一问题 → 模型生成多个回答 → 人工排序

第2步: 训练奖励模型 (Reward Model)
  学习预测人类偏好分数

第3步: PPO 强化学习
  用奖励模型指导模型优化
  生成高分回答 → 获得奖励 → 强化该行为
```

**奖励模型**：
```
问题 + 回答 → Reward Model → 标量分数 (越高越好)
```

**PPO 的挑战**：
- 训练不稳定（强化学习固有难题）
- 奖励黑客（模型学会钻奖励模型的空子）
- 计算成本高（需要同时运行 4 个模型）

### 12.7 DPO：直接偏好优化

DPO 用数学证明了：不需要奖励模型，直接用偏好数据就能对齐。

```
RLHF: 偏好数据 → 训练奖励模型 → PPO 优化 (3步, 4个模型)
DPO:  偏好数据 → 直接优化策略模型 (1步, 1个模型)
```

**DPO 损失函数核心思想**：
- 增大被偏好回答的概率
- 减小不被偏好回答的概率
- 保持与参考模型的相对关系

**DPO vs RLHF**：

| 维度 | RLHF | DPO |
|------|------|-----|
| 复杂度 | 高（4个模型） | 低（1个模型） |
| 稳定性 | 不稳定 | 稳定 |
| 效果 | 强（理论上限高） | 接近 RLHF |
| 数据需求 | 偏好对 + 奖励模型 | 仅偏好对 |

**DPO 的变体**：
- **IPO**：更强的理论保证，避免过拟合
- **KTO**：只需"好/坏"标签，不需要成对偏好
- **ORPO**：将 SFT 和对齐合并为一步

## 代码示例

```python
# QLoRA 微调示例（使用 transformers + peft）
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training

# 4-bit 量化配置
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
)

# 加载模型
model = AutoModelForCausalLM.from_pretrained(
    "Qwen/Qwen2-7B", quantization_config=bnb_config, device_map="auto"
)
model = prepare_model_for_kbit_training(model)

# LoRA 配置
lora_config = LoraConfig(
    r=16,
    lora_alpha=32,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    lora_dropout=0.05,
    task_type="CAUSAL_LM",
)
model = get_peft_model(model, lora_config)
model.print_trainable_parameters()
# → trainable params: 13M || all params: 7B || trainable%: 0.19%

# DPO 训练示例（使用 TRL）
from trl import DPOTrainer, DPOConfig

# 偏好数据格式
train_dataset = [
    {"prompt": "解释重力", "chosen": "重力是物体间的吸引力...", "rejected": "重力就是东西往下掉"},
    # ...
]

dpo_config = DPOConfig(
    output_dir="./dpo-output",
    per_device_train_batch_size=4,
    learning_rate=5e-5,
    num_train_epochs=1,
    beta=0.1,  # DPO 温度参数
)

trainer = DPOTrainer(
    model=model,
    args=dpo_config,
    train_dataset=train_dataset,
    tokenizer=tokenizer,
)
trainer.train()
```

## 个人思考

1. SFT 数据质量 > 数量——如何构建高质量的指令数据集？
2. QLoRA 的秩 r 如何选择？（简单任务 r=8，复杂任务 r=64）
3. DPO 是否正在取代 RLHF 成为新的对齐标准？
4. 微调后的模型如何持续迭代更新？（数据迭代 + 增量微调）
5. 开源微调工具链的成熟度：TRL + PEFT + BitsAndBytes 已经非常完善
