# Han Lee: Agent ≠ Model

> 作者：Han Lee (Lee Hanchung)
> 来源：个人博客 "Han, Not Solo" (leehanchung.github.io)
> 核心论点：Agent 本质是决策工作流系统，模型只是其中一个组件

## 核心框架

```
Agent = Model + Harness + Tools + Environment + Rewards + Evaluation
         ↑
       只是其中一个组件
```

## 一、Agent 本质是工作流，不是模型

> Agent 本质上是 workflows（工作流）、决策流程和自动执行。换了 LLM 作为引擎，并不改变 Agent 的本质。

- "Agent" 这个词在 1990 年代就被广泛讨论（Wooldridge & Jennings, 1995），当时已警告它可能成为"噪音词"
- 今天的 Agent 和过去的 BPA/RPA 在本质上是同一件事
- 区别只是：引擎从**规则系统**换成了 **LLM**
- 类比：把汽车的燃油发动机换成电动发动机——发动机变了，但仍然是汽车

### Agent 的定义

Agent = 代表用户自主执行任务的**软件实体**

Agent 不是简单的 while 循环调 API，而是设计复杂的**决策系统**，可以用**有限状态机（FSM）**建模。

### 核心价值

> 不是追求 AGI，而是创造实用的 AUI（Actually Useful Intelligence）。

## 二、Agentic AI 系统 ≠ 单个模型

用 Mario 类比 Agentic AI 全栈：

| Mario 概念 | Agentic AI 对应 | 说明 |
|---|---|---|
| Small Mario（小马里奥） | 预训练基础模型 | 脆弱、不可靠，碰一个 Goomba 就死 |
| Super Mushroom（蘑菇） | 模型 Harness + 后训练 | 指令微调、安全护栏、系统提示词、记忆管理 |
| Fire Flower（火焰花） | 代码执行工具 | 远程解决模型无法用文本解决的问题 |
| Frog Suit（蛙装） | Web 搜索 | 导航模型未训练过的信息环境 |
| Raccoon Leaf（浣熊尾） | 文件系统访问 | 探索和修改代码库 |
| Hammer Suit（锤子装） | Shell/终端访问 | 最强大的工具，与完整系统交互 |
| Star（无敌星） | 扩展思考/推理模式 | 高计算成本暴力破解复杂问题 |
| Cape Feather（斗篷） | MCP 服务器 | 持续的、可扩展的外部服务访问 |
| 关卡（Levels） | 任务环境 | 不同难度的任务 |
| 旗杆 | 奖励信号和评估 | 衡量任务完成质量 |
| World Map | Agent 框架 | 组织 Agent 在子任务和领域间导航 |

### 关键洞察

1. **蘑菇不改变 Mario 是谁，改变的是他能承受什么** — Harness 不改变模型的核心知识，改变的是模型在生产环境中能处理什么
2. **道具不替代核心能力，而是扩展** — LLM 仍然做推理、语言理解、规划；工具扩展模型无法单独触达的环境
3. **Mario 必须学会何时用哪个道具** — 蛙装在水关很强，在陆地无用。模型需要学习**工具选择**，知道什么场景用什么工具

## 三、RPA 式的"画框框"不是 Agent

> Langflow、Make、n8n 这些工作流编排工具本质上只是 RPA，跟 Agent 毫无关系。

### 工作流编排工具存在的根本原因

> Agent 框架和工作流构建器之所以存在，是因为 LLM 模型本身还不够强，无法自主完成任务。

### ComfyUI 案例：模型能力提升使编排工具过时

- 2023 年初：图像生成需要用 ComfyUI 编排复杂工作流（Stable Diffusion + LoRA + ControlNet + ...）
- 2025 年 3 月：ChatGPT 图像生成能力发布，一个 prompt 即可达到同样效果
- 结论：**The model _is_ the product. 模型本身就是产品。**

### 对"低代码 Agent 平台"的判断

- 这些工具对软件工程顾问快速拼接方案卖给非技术客户有用
- 但往往 0→1 捕获大部分价值，没有足够价值从 1→100
- 结果：维护负担重但又有足够价值舍不得丢的"小白象"

## 四、Agent 的数学本质：MDP + Bellman 方程

Agent 的顺序决策过程可以用强化学习框架严格建模：

### Bellman 方程

```
v^π(s) = E[r(s, a) + γ·v_π(s')]
```

当前状态的价值 = 即时奖励的期望 + 折扣后的下一状态价值

### MDP 循环

```
观察当前状态 → 根据策略决定动作 → 执行并接收环境反馈 → 更新策略
```

这和所有 Agentic AI 系统的运行循环完全一致（ReAct、Self-Refine、Reflexion 等都可以纳入这个框架）。

### 推理工具 = 动作空间

所有"推理框架"论文（ReAct、Self-Reflection、Self-Refine、Reflexion）都可以塌缩为一组**推理工具**：

```
动作空间 = { reason, act, reflect, refine, vote }
奖励函数 = verification pattern（验证模式）
```

| 论文/框架 | 动作序列 |
|---|---|
| ReAct | reason → act → reason → act → ... |
| Self-Refine | generate → refine → refine → ... |
| Reflexion | act → reflect → act → reflect → ... |
| Majority Voting | sample × N → vote |

> 这些框架要么是 Agent 自主选择动作（actions determined by agent），要么是开发者预设动作展开（actions pre-defined by developers）。

## 五、奖励设计：Agent 开发中杠杆最高的决策

| 评估维度 | Mario 度量 | Agent 度量 |
|---|---|---|
| 速度 | 剩余时间 | 任务完成延迟 |
| 分数 | 累积分数 | 整体输出质量 |
| 收集 | 金币数 | 信息检索效率 |
| 完整性 | 隐藏方块数 | 边缘情况覆盖 |
| 效率 | 每条命击败敌人数 | 每任务正确工具调用数 |

> 奖励设计不当会产出"技术性完成任务但方式扭曲"的 Agent——就像 Mario 速通玩家穿墙一样， impressive，但不是我们真正想要的。

**Goodhart 定律**：当一个度量成为目标，它就不再是好的度量。

## 六、开发 Agent 系统是 99% 工程 + 1% AI

> 分解、抽象、组织——这些一直是软件工程的核心。换了 LLM 引擎，不改变这个事实。

### 两个关键角色

| 角色 | 职责 |
|---|---|
| ML Engineer（游戏设计师） | 设计环境、定义任务、设计奖励函数 |
| ML Systems Engineer（运行基础设施） | 大规模分布式训练、数据流水线、推理轨迹收集 |

## 七、Agent 框架是临时产物

### 预测

> 随着模型能力提升，Agent 框架和低代码工作流工具将逐渐过时。

类比：
- Super Mario 的 World Map（框架）是有用的脚手架
- 但 Mario 才是真正打游戏的
- **今天的 World 8（最难的领域）是明天的 World 1（基础任务）**

### 两条路

1. 打上一场战争（继续画框框编排工作流）
2. 滑向冰球要去的地方（投资模型后训练和能力提升）

## 个人思考

### 关键启发

1. **不要把 Agent 和 Model 混为一谈** — 评估 Agent 系统不能只看底层模型 benchmark
2. **奖励设计是最高杠杆的决策** — 设计好 reward function 比调模型参数重要得多
3. **工具选择能力是 Agent 的核心难点** — 不是"能不能用工具"，而是"知道什么时候用什么工具"
4. **框架是临时脚手架** — 不要过度投资在编排工具上，投资模型能力本身
5. **Agent 的本质问题没有变** — 30 年前的问题今天仍然存在，只是工具更好了

### 待探索

- Process Reward Model (PRM) vs Outcome Reward Model (ORM) 的实际应用
- 如何为非确定性任务（如内容生成）设计奖励函数
- Agent 自对齐（类似 AlphaZero self-play）的可行路径

## 参考资料

- [What The Hype and Reality of Agents](https://leehanchung.github.io/blogs/2024/10/26/thoughts-on-agents/) — Agent 的历史与现实
- [It's-a Me, Agentic AI](https://leehanchung.github.io/blogs/2026/02/18/mario-agentic-ai/) — Mario 类比 Agentic AI 全栈
- [No Code, Low Code, Real Code](https://leehanchung.github.io/blogs/2025/06/26/no-code-low-code-full-code/) — 为什么工作流编排工具终将过时
- [Reasoning with Compound AI Systems and Post-Training](https://leehanchung.github.io/blogs/2024/11/22/reasoning-agents-post-training/) — 复合 AI 系统与后训练推理
- Wooldridge, M., & Jennings, N. R. (1995). Intelligent agents: Theory and practice.
- Franklin, S., & Graesser, A. (1996). Is it an agent, or just a program?
- Etzioni, O. (1996). Moving up the information food chain.
