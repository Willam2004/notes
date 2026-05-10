# 第6章 提示工程

## 核心概念

> 不改模型参数，仅凭"说话方式"的改变，就能让大模型表现天壤之别。

### 6.1 好 Prompt 的五要素

| 要素 | 作用 | 示例 |
|------|------|------|
| **角色** | 激活对应领域知识 | "你是资深 Python 后端工程师" |
| **任务** | 清晰具体，避免模糊 | "审查代码，找出性能瓶颈并优化" |
| **上下文** | 提供决策所需背景 | "日均百万请求，PostgreSQL，32核" |
| **格式** | 明确输出结构 | "问题→原因→方案→收益" |
| **约束** | 边界条件和禁止项 | "兼容 Python 3.8+，不改函数签名" |

### 6.2 Zero-shot / One-shot / Few-shot

```
Zero-shot: 直接提问，不给示例
One-shot:  给 1 个示例，理解格式和风格
Few-shot:  给 3-8 个示例，学习输入-输出映射模式
```

Few-shot 的本质是**上下文学习（In-Context Learning）**：模型做模式匹配和类比推理，不是真的"重新学习"。

### 6.3 思维链（Chain-of-Thought, CoT）

让模型把推理步骤写出来，而不是直接给答案。

**触发方式**：
- **Zero-shot CoT**：末尾加"让我们一步步思考"
- **Few-shot CoT**：在示例中展示完整推理过程

**适用场景**：

| 任务 | CoT 效果 |
|------|---------|
| 多步数学推理 | 显著提升 |
| 逻辑推断 | 显著提升 |
| 代码调试 | 显著提升 |
| 简单事实问答 | 几乎无效 |
| 创意写作 | 可能有害 |

### 6.4 自我一致性（Self-Consistency）

```
同一问题 → 多次 CoT（temperature>0）→ 多数投票 → 最频繁答案
```

准确率提升 5-10%，代价是 N 倍 API 调用。

### 6.5 思维树（Tree-of-Thought, ToT）

CoT 是线性的（一条链走到底），ToT 在每个节点探索**多条分支**，评估并剪枝。适合复杂规划任务，成本比 CoT 高 10 倍以上。

### 6.6 System Prompt 设计层次

```
身份角色定位（"我是谁"）
  → 能力边界（"我能做什么"）
    → 行为准则（"我怎么做"）
      → 输出格式规范（"用什么形式输出"）
        → 硬性约束（"绝对不做什么"）
```

### 6.7 结构化输出（三层）

| 层次 | 方式 | 可靠性 |
|------|------|--------|
| 基础 | Prompt 中要求 JSON + Schema | 一般 |
| 中级 | Prompt 中提供 JSON 示例 | 较好 |
| 高级 | API `response_format` 参数 + Pydantic 验证 | 最强 |

### 6.8 推理模型的 Prompt 策略（System 2）

针对 DeepSeek-R1、o1/o3 等推理模型：

- **不要指定推理过程** — 让模型自行决定怎么思考
- **不要加 CoT** — 推理模型已内置思维链，额外 CoT 可能干扰
- **简单任务用普通模型** — 推理模型更慢更贵

### 6.9 四类高频 Prompt 错误

```
❌ 任务不明确："帮我改进这段代码"
✅ 具体化："重构代码，目标：①嵌套≤2层 ②提取重复逻辑为函数"

❌ 忘记格式样本："翻译成儿童版"
✅ 加一个改写示例，让模型理解"儿童版"的含义

❌ 否定式指令："不要太啰嗦，不要术语，不要分点"
✅ 肯定式："150字口语化表达，像朋友聊天，一段话说清"

❌ 一次塞太多任务："翻译+总结+提取关键词+评分"
✅ 拆分多轮或明确优先级
```

## 代码示例

```python
# Self-Consistency 实现
from collections import Counter

def self_consistent_answer(question, n_samples=5):
    answers = []
    for _ in range(n_samples):
        response = call_llm(f"{question}\n让我们一步步思考：", temperature=0.7)
        answers.append(extract_final_answer(response))
    return Counter(answers).most_common(1)[0][0]

# 结构化输出 + Pydantic 验证
from pydantic import BaseModel
from typing import Literal

class ReviewAnalysis(BaseModel):
    sentiment: Literal["正面", "负面", "中性"]
    score: int
    key_points: list[str]
    summary: str

# TutorBot 完整示例
class TutorBot:
    def __init__(self, subject, grade):
        self.system = f"你是学习伙伴，风格亲切耐心，善用类比。科目：{subject}，年级：{grade}"
        self.history = []

    def chat(self, user_input):
        self.history.append({"role": "user", "content": user_input})
        response = client.chat.completions.create(
            model="deepseek-chat",
            messages=[{"role": "system", "content": self.system}, *self.history],
            temperature=0.7,
        )
        reply = response.choices[0].message.content
        self.history.append({"role": "assistant", "content": reply})
        return reply
```

## 个人思考

1. Prompt 调试的系统方法：缩小问题 → 加示例 → 加 CoT → 调 temperature → 加验证
2. System Prompt 设计对长期对话质量影响很大
3. 推理模型 vs 普通模型的选择标准：任务复杂度 + 成本预算
