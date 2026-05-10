# 第7章 高级文本生成技术与工具

## 核心概念

> 从"单次问答"到"智能体系统"——LangChain 如何将 LLM 组织为复杂应用。

### 7.1 模型输入/输出：基于 LangChain 加载量化模型

LangChain 是构建 LLM 应用的核心框架。第一步是学会加载和使用模型。

**量化模型**：通过降低参数精度（如 FP16 → INT4），在消费级 GPU 上运行大模型。

```
原始模型 (FP16) → 量化 (INT4/INT8) → 显存降低 2-4 倍 → 推理速度提升
```

LangChain 统一了不同模型提供商的接口：
- OpenAI / DeepSeek 等商业 API
- HuggingFace 本地模型
- Ollama 本地推理

### 7.2 链（Chains）：扩展 LLM 的能力

链是 LangChain 的核心抽象——将多个组件串联成管线。

```
用户输入 → Prompt 模板 → LLM → 输出解析器 → 结构化结果
```

**常见链类型**：

| 链类型 | 作用 | 示例 |
|--------|------|------|
| LLMChain | Prompt + LLM + 解析 | 基础问答 |
| SequentialChain | 多步骤串行 | 翻译 → 摘要 → 评分 |
| RouterChain | 动态路由到子链 | 根据问题类型选择处理链 |

**Prompt 模板**：将固定格式与变量分离，支持复用。

### 7.3 记忆（Memory）：构建 LLM 的对话回溯能力

LLM 本身无状态，记忆模块让多轮对话成为可能。

**记忆类型对比**：

| 记忆类型 | 机制 | 优缺点 |
|----------|------|--------|
| ConversationBufferMemory | 保存完整对话历史 | 简单但 token 消耗大 |
| ConversationBufferWindowMemory | 只保留最近 K 轮 | 控制成本，丢失早期上下文 |
| ConversationSummaryMemory | 用 LLM 压缩历史 | 节省 token，但有信息损失 |
| VectorStoreMemory | 对话存入向量库检索 | 长期记忆，适合长对话 |

```
记忆选择标准：
  短对话(< 10轮) → BufferMemory
  中等对话 → WindowMemory
  长对话/客服 → SummaryMemory 或 VectorStoreMemory
```

### 7.4 智能体（Agents）：构建 LLM 系统

智能体 = LLM + 工具 + 决策循环。LLM 不再是被动回答，而是自主选择工具并执行。

```
用户目标 → Agent(大模型作为"大脑")
              ↓
          思考：需要什么工具？
              ↓
         ┌────┼────┐
       搜索  计算  代码执行  数据库查询
         └────┼────┘
              ↓
          观察结果 → 继续思考或返回答案
```

**ReAct 模式**（Reasoning + Acting）：
1. **Thought**：分析当前情况，决定下一步
2. **Action**：调用工具执行操作
3. **Observation**：观察工具返回结果
4. 循环直到得出最终答案

**常用工具类型**：
- 搜索引擎（Tavily、SerpAPI）
- 代码执行（Python REPL）
- 数据库查询（SQL）
- 文件操作
- API 调用

## 代码示例

```python
# LangChain 基础：加载模型 + 对话
from langchain_community.chat_models import ChatOllama
from langchain_core.messages import HumanMessage

llm = ChatOllama(model="llama3")
response = llm.invoke([HumanMessage(content="用一句话解释量子计算")])
print(response.content)

# 带 Prompt 模板的链
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template(
    "你是{role}。请用通俗易懂的语言解释：{topic}"
)
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"role": "资深物理老师", "topic": "黑洞"})
print(result)

# 带记忆的对话
from langchain_community.chat_models import ChatOllama
from langchain_core.messages import SystemMessage
from langchain.memory import ConversationBufferMemory

memory = ConversationBufferMemory(return_messages=True)
memory.chat_memory.add_message(SystemMessage(content="你是友好的助手"))
memory.chat_memory.add_user_message("你好，我叫小明")
memory.chat_memory.add_ai_message("你好小明！有什么可以帮你的？")

# Agent + 工具
from langchain.agents import create_react_agent, Tool
from langchain_community.tools import DuckDuckGoSearchRun

search = DuckDuckGoSearchRun()
tools = [
    Tool(name="search", func=search.run, description="搜索互联网获取信息"),
]
```

## 个人思考

1. LangChain 的链 vs 智能体如何选择？（确定性流程用链，开放性任务用智能体）
2. Agent 的可靠性问题——如何防止"死循环"或"幻觉工具调用"？
3. 量化模型的质量损失在实际应用中是否可接受？
