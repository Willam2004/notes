# Mac mini：Multica AI 基金/股票投资智能体搭建操作手册

> 版本：v0.1  
> 目标：先在 Mac mini 上搭出一个可运行的个人投研系统。第一阶段只做“数据获取 → 基金持仓 → 量化分析 → 财报研究 → 风险审查 → 报告”，**不自动下单**。

---

## 1. 最终要搭成什么

```text
招商银行基金持仓
        │
        ▼
 portfolio.csv
        │
        ▼
                    Multica
               Investment Squad
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
   Market Data    Financials      Research
       │              │              │
 AkShare/东财     Dayu Agent       LLM/Web
 股票/ETF/基金     巨潮资讯网       行业/新闻
       │              │              │
       └──────────────┼──────────────┘
                      ▼
                  Quant Agent
                      │
                      ▼
                   Risk Agent
                      │
                      ▼
               Portfolio Manager
                      │
                      ▼
                每日/每周报告
                      │
                      ▼
                   人工决策
```

### 各层职责

- **Multica**：总控、Agent/Squad 编排、任务运行。
- **AkShare / 东方财富相关数据**：A股、ETF、指数、基金净值、成交量、涨跌等市场数据。
- **stock-skills**：给 Agent 提供 A 股行情、技术指标、资金等能力。
- **stock-analysis**：股票、ETF、基金和组合研究。
- **Dayu Agent**：财报分析引擎；A股财报重点利用巨潮资讯网。
- **pandas**：确定性数据计算。
- **vectorbt**：第二阶段用于批量回测。
- **Qlib**：第三阶段用于因子、机器学习和 AI 量化研究。

---

## 2. 第一阶段不要做什么

暂时不要：

- 自动连接招商银行执行交易；
- 自动买卖基金/股票；
- 一开始就训练复杂 AI 模型；
- 一开始就部署 PostgreSQL、Kafka 等重基础设施；
- 同时安装十几个量化框架。

第一阶段目标只有一个：

> **每天能够基于真实持仓和真实市场数据，生成一份有用的基金组合分析报告。**

---

## 3. Mac mini 环境检查

打开 Terminal / Ghostty。

### 3.1 检查 Git

```bash
git --version
```

### 3.2 检查 Python

```bash
python3 --version
```

建议 Python 3.11 或 3.12。

### 3.3 检查 Homebrew

```bash
brew --version
```

如果以上工具已有，不需要重复安装。

### 3.4 安装 uv（推荐）

```bash
brew install uv
```

检查：

```bash
uv --version
```

---

## 4. 创建项目

建议放在：

```bash
mkdir -p ~/Projects
cd ~/Projects
mkdir investment-agent
cd investment-agent
git init
```

创建目录：

```bash
mkdir -p \
  portfolio \
  data/raw \
  data/processed \
  research/funds \
  research/sectors \
  research/companies \
  strategies/trend \
  strategies/drawdown \
  strategies/rotation \
  backtest/results \
  reports/daily \
  reports/weekly \
  skills \
  scripts
```

最终：

```text
investment-agent/
├── portfolio/
│   ├── portfolio.csv
│   └── transactions.csv
├── data/
│   ├── raw/
│   └── processed/
├── research/
│   ├── funds/
│   ├── sectors/
│   └── companies/
├── strategies/
├── backtest/
│   └── results/
├── reports/
│   ├── daily/
│   └── weekly/
├── skills/
└── scripts/
```

---

## 5. 建立 Python 环境

在 `investment-agent` 根目录：

```bash
uv venv
source .venv/bin/activate
```

安装第一批依赖：

```bash
uv pip install pandas numpy matplotlib akshare requests
```

确认：

```bash
python -c "import pandas, akshare; print('OK')"
```

如果显示：

```text
OK
```

说明基础数据环境已经成功。

---

## 6. 第一个数据测试：获取 A 股行情

创建：

```bash
touch scripts/test_market_data.py
```

写入：

```python
import akshare as ak

df = ak.stock_zh_a_spot_em()

cols = [
    "代码",
    "名称",
    "最新价",
    "涨跌幅",
    "成交额",
]

print(df[cols].head(20))
```

运行：

```bash
python scripts/test_market_data.py
```

如果能看到股票代码、名称、最新价和涨跌幅，说明：

```text
Mac mini
   ↓
Python
   ↓
AkShare
   ↓
A股行情
```

这条链已经打通。

---

## 7. 基金净值测试

对于招商银行购买的场外公募基金，核心数据通常不是实时股价，而是每日基金净值 NAV。

创建：

```bash
touch scripts/test_fund.py
```

示例：

```python
import akshare as ak

FUND_CODE = "替换成你的基金代码"

df = ak.fund_open_fund_info_em(
    symbol=FUND_CODE,
    indicator="单位净值走势",
)

print(df.tail(20))
```

运行：

```bash
python scripts/test_fund.py
```

以后 Quant Agent 可以基于净值计算：

```text
1日收益
5日收益
20日收益
60日收益
120日收益
近期回撤
MA20
MA60
波动率
最大回撤
```

---

## 8. 建立招商银行真实持仓文件

创建：

```bash
touch portfolio/portfolio.csv
```

格式：

```csv
fund_code,fund_name,market_value,cost_value,profit,profit_rate
012345,某半导体基金,50000,45000,5000,0.1111
006789,某人工智能基金,30000,28000,2000,0.0714
001234,某沪深300基金,80000,76000,4000,0.0526
```

第一阶段可以从招商银行 App 截图后手工整理。

**不要在项目里保存：**

- 银行卡号
- 身份证号码
- 银行密码
- 招商银行登录信息
- 短信验证码
- 其他账户凭据

---

## 9. 安装 stock-skills

项目：`tetap/stock-skills`

用途：

- A股行情；
- 技术指标；
- 资金数据；
- 基本面；
- 市场热点；
- Agent / CLI / MCP 能力。

建议先阅读项目最新 README，再按照仓库当前安装方式执行，避免文档版本变化。

克隆：

```bash
cd ~/Projects
git clone https://github.com/tetap/stock-skills.git
cd stock-skills
```

然后：

```bash
ls
```

阅读：

```bash
less README.md
```

安装完成后，再将其能力提供给 Multica 中的 Data/Quant Agent。

---

## 10. 安装 stock-analysis

项目：`AdvancingTitans/stock-analysis`

用途：

- 股票研究；
- ETF/基金研究；
- Portfolio Review；
- 估值；
- 情景分析；
- Agent 投资研究。

克隆：

```bash
cd ~/Projects
git clone https://github.com/AdvancingTitans/stock-analysis.git
cd stock-analysis
less README.md
```

按照仓库最新 README 安装。

第一阶段主要给：

```text
Research Agent
Portfolio Manager
```

使用。

---

## 11. 安装 Dayu Agent

项目：`noho/dayu-agent`

定位：

> 财报分析引擎

主要用于：

- 下载/读取上市公司财报；
- A股财报研究；
- 财务指标提取；
- 财报问答；
- 财报风险发现；
- 生成可追踪的研究结果。

A股财报重点来源：

> **巨潮资讯网**

克隆：

```bash
cd ~/Projects
git clone https://github.com/noho/dayu-agent.git
cd dayu-agent
less README.md
```

**安装命令以项目当前 README 为准。**

不要急着把 Dayu 和 Multica 集成。

先完成一个最简单的验证：

```text
指定一家 A 股公司
       ↓
找到/下载最近一期财报
       ↓
Dayu Agent 成功读取
       ↓
回答：
“最近三年营收、净利润、经营现金流有什么变化？”
```

这一步成功以后，再接 Multica。

---

## 12. 财报研究链路

未来针对某只基金：

```text
基金
 ↓
获取基金重仓股
 ↓
Top 10 股票
 ↓
例如：
北方华创
中芯国际
海光信息
澜起科技
 ↓
Dayu Agent
 ↓
巨潮资讯网财报
 ↓
分析：
营收
净利润
毛利率
研发费用
库存
应收账款
经营现金流
管理层风险提示
 ↓
Financial Statement Agent
```

最终不是简单说：

> “半导体最近上涨。”

而应该能解释：

> 哪些重仓公司盈利改善、哪些公司现金流恶化、产业趋势是否支持当前估值。

---

## 13. Multica 创建 Investment Squad

建议建立：

```text
Investment Squad
```

第一版只创建 5 个 Agent。

### Data Agent

负责：

```text
portfolio.csv
基金净值
股票行情
ETF行情
指数
基金持仓
```

原则：

> 只负责事实，不给买卖建议。

### Quant Agent

负责：

```text
收益率
MA20/60/120
趋势
动量
波动率
最大回撤
行业相对强弱
回测
```

原则：

> 数字尽量由 Python 计算，不让 LLM 猜。

### Research Agent

负责：

```text
行业
新闻
政策
基金季报
上市公司研究
Dayu 财报结果
```

### Risk Agent

专门唱反调：

```text
基金重叠
行业集中度
科技暴露
估值风险
追涨风险
回撤风险
结论的不确定性
```

### Portfolio Manager

只负责汇总：

```text
Data
+
Quant
+
Research
+
Risk
        ↓
持有 / 观察 / 加仓候选 / 暂停加仓 / 降低仓位
```

最终操作仍由人完成。

---

## 14. Investment Constitution

创建：

```bash
touch skills/investment_constitution.md
```

第一版写入：

```markdown
# Investment Constitution

1. AI 不得自动执行真实交易。
2. 加仓建议必须同时参考 Quant、Research 和 Risk。
3. 禁止仅根据新闻给出买入建议。
4. 所有数字必须说明数据来源或计算方式。
5. 必须分析基金底层持仓重叠。
6. 必须分析组合层面的行业集中度。
7. 禁止因为短期大涨产生 FOMO 追涨建议。
8. 每个投资判断必须提供 Bull Case。
9. 每个投资判断必须提供 Bear Case。
10. 必须说明什么情况证明当前判断错误。
11. 回测结果不代表未来收益。
12. 所有输出只作为投资决策参考。
```

这个文件以后是整个 Investment Squad 的最高规则之一。

---

## 15. 第一版每日工作流

```text
09:00 / 收盘后
      ↓
Data Agent
      ↓
更新：
股票 / ETF / 指数 / 基金净值
      ↓
Quant Agent
      ↓
计算：
趋势 / 动量 / 回撤 / 风险
      ↓
Research Agent
      ↓
行业 / 新闻 / 财报变化
      ↓
Risk Agent
      ↓
检查：
集中度 / 重叠 / 追涨 / 风险
      ↓
Portfolio Manager
      ↓
reports/daily/YYYY-MM-DD.md
```

对于场外基金，净值往往在收盘后更新，因此第一阶段更适合做**每日收盘后的组合报告**，而不是盘中高频判断。

---

## 16. 每日报告格式

建议固定格式：

```markdown
# 每日投资报告

## 组合概览

总资产：
今日收益：
本月收益：

## 行业暴露

科技：
半导体：
AI：
消费：
医药：
黄金：
宽基：

## 基金分析

### 基金 A

Quant：72/100
Research：81/100
Risk：55/100

结论：继续持有

原因：

Bull Case：

Bear Case：

重新评估条件：

## 组合风险

- 基金重叠
- 行业集中度
- 最大风险

## 今日行动

- 不操作
- 进入观察
- 加仓候选
- 暂停加仓
```

---

## 17. 第二阶段：回测

系统运行稳定后安装：

```bash
uv pip install vectorbt
```

回测例如：

```text
MA20 > MA60
+
距离60日高点回撤 > 8%
        ↓
进入加仓观察区
```

检查：

- 历史出现次数；
- 1个月后收益；
- 3个月后收益；
- 胜率；
- 最大亏损；
- 最大回撤；
- 与 Buy & Hold 比较。

**先验证规则，再考虑用于实际投资判断。**

---

## 18. 第三阶段：Qlib

基础系统稳定后再加入 Qlib。

用途：

```text
因子
 ↓
多因子
 ↓
LightGBM / ML
 ↓
收益预测
 ↓
ETF / 股票评分
 ↓
行业轮动
 ↓
Portfolio
```

不要把 Qlib 当第一阶段的阻塞项。

---

## 19. 推荐实施顺序

### 第一天

完成：

```text
[ ] 创建 investment-agent
[ ] 创建 Python venv
[ ] 安装 AkShare / pandas
[ ] 成功获取 A 股行情
[ ] 成功获取一只基金历史净值
```

### 第二步

完成：

```text
[ ] 整理招商银行基金持仓
[ ] 生成 portfolio.csv
[ ] 写第一个 portfolio analyzer
```

### 第三步

完成：

```text
[ ] 安装 stock-skills
[ ] 安装 stock-analysis
[ ] 验证股票/基金研究
```

### 第四步

完成：

```text
[ ] 安装 Dayu Agent
[ ] 下载/读取一份 A 股财报
[ ] 完成一次财报分析
```

### 第五步

完成：

```text
[ ] Multica 创建 Investment Squad
[ ] 创建 5 个 Agent
[ ] 加入 Investment Constitution
[ ] 生成第一份真实每日投资报告
```

### 后续

```text
[ ] vectorbt 回测
[ ] 基金持仓穿透
[ ] 自动生成周报
[ ] Qlib
[ ] 因子/机器学习
```

---

## 20. 第一次在 Mac mini 上操作时的验收标准

第一轮不要追求完整。

只要做到下面四件事，就算成功：

```text
1. Python 可以获取 A 股行情
          ✓

2. Python 可以获取一只真实基金净值
          ✓

3. portfolio.csv 有你的真实基金组合
          ✓

4. Dayu Agent 能成功分析一份 A 股财报
          ✓
```

四个基础能力都成功后，再开始 Multica Agent 编排。

这样出了问题也很容易判断到底是：

```text
数据问题
Python问题
Dayu问题
还是 Multica 问题
```

而不会把所有系统一次性连起来以后难以排查。

---

## 21. 安全边界

整个项目建议始终坚持：

```text
AI负责：
数据整理
研究
回测
风险分析
投资建议

人负责：
最终判断
真实下单
资金管理
```

第一阶段不要给 Agent：

- 银行密码；
- 短信验证码；
- 交易密码；
- 自动下单权限；
- 不受限制的真实账户写权限。

这套系统首先应该成为一个**高质量个人投研助手**，而不是自动交易机器人。
