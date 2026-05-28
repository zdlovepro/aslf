# AGENTS.md

本文件用于说明本项目中 AI Agent、自动化脚本、代码生成工具以及协作开发者在修改代码、生成文档、调整架构和实现功能时必须遵守的规则。

本项目是一个面向 A 股低频量化交易的后端系统，包含数据管理、指标因子、策略研究、历史回测、参数优化、模拟盘、实盘交易接入、风控系统、监控告警和审计日志等模块。

---

## 1. 项目定位

项目名称：

```text
A 股低频量化策略研究、回测、模拟盘与实盘执行系统
```

项目类型：

```text
纯后端系统
低频量化交易系统
A 股交易系统
策略研究与实盘执行一体化系统
```

系统目标：

```text
1. 支持 A 股历史行情数据管理。
2. 支持技术指标计算。
3. 支持量化因子计算。
4. 支持低频策略开发。
5. 支持历史回测。
6. 支持参数优化。
7. 支持模拟盘验证。
8. 支持 A 股实盘交易接入。
9. 支持实盘风控。
10. 支持订单、成交、资金、持仓同步。
11. 支持监控告警。
12. 支持审计日志。
```

本项目的核心设计思想：

```text
策略只生成信号；
交易计划不等于订单；
实盘订单必须经过风控；
状态不确定时不自动交易；
系统长期只做低频交易，不做高频交易。
```

---

## 2. 永久排除范围

本项目永久不做高频交易。

禁止实现以下能力：

```text
1. Tick 级策略。
2. Tick 级回测。
3. Tick 级行情链路。
4. Level-2 盘口策略。
5. Level-2 盘口数据链路。
6. 逐笔成交策略。
7. 做市策略。
8. 盘口套利策略。
9. 高频撤单策略。
10. 高频报单系统。
11. 毫秒级或微秒级低延迟交易。
12. 极速柜台交易。
13. 订单簿排队模拟。
14. 高频撮合仿真。
15. 高频延迟监控。
```

如果代码、接口、数据库表或文档中出现以下关键词，需要谨慎检查：

```text
tick
level2
level_2
hft
high_frequency
order_book
market_making
latency
microsecond
millisecond
quote
bid_ask_queue
```

除非只是明确说明“不支持”，否则不应加入相关功能。

---

## 3. 技术栈约定

推荐后端技术栈：

```text
Python
FastAPI
PostgreSQL
Redis
Celery 或 Dramatiq
SQLAlchemy
Pydantic
Alembic
Pandas
NumPy
Optuna
Pytest
Docker
Docker Compose
```

说明：

```text
1. 本地开发阶段可以使用 SQLite 做快速验证。
2. 正式设计和生产环境以 PostgreSQL 为主。
3. Redis 只用于缓存、任务状态、分布式锁等辅助场景。
4. Redis 不得作为实盘订单、成交、资金、持仓的唯一存储。
5. 实盘交易相关数据必须持久化到 PostgreSQL。
```

---

## 4. 推荐项目目录结构

建议项目根目录结构如下：

```text
.
├── AGENTS.md
├── README.md
├── .gitignore
├── pyproject.toml
├── requirements.txt
├── docker-compose.yml
├── .env.example
├── app/
│   ├── main.py
│   ├── api/
│   ├── core/
│   ├── db/
│   ├── models/
│   ├── schemas/
│   ├── services/
│   ├── repositories/
│   ├── tasks/
│   └── utils/
├── quant/
│   ├── data/
│   ├── indicator/
│   ├── factor/
│   ├── strategy/
│   ├── backtest/
│   ├── optimizer/
│   ├── simulation/
│   ├── live/
│   ├── risk/
│   ├── report/
│   └── monitor/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── mock_broker/
│   └── fixtures/
├── docs/
│   ├── SRS.md
│   ├── system_design.md
│   ├── database_design.md
│   ├── strategy_design.md
│   ├── risk_design.md
│   ├── api_design.md
│   ├── backtest_design.md
│   ├── live_trading_design.md
│   ├── simulation_design.md
│   └── test_plan.md
├── scripts/
├── migrations/
├── data/
│   ├── raw/
│   ├── processed/
│   └── sample/
└── logs/
```

---

## 5. 核心模块职责

### 5.1 data 数据模块

负责：

```text
股票基础信息
交易日历
日线行情
可选分钟行情
复权因子
停牌状态
涨跌停状态
数据导入
数据清洗
数据质量检查
```

要求：

```text
1. stock_daily 表中 symbol + trade_date 必须唯一。
2. 不得导入 Tick 数据。
3. 不得建设 Level-2 数据链路。
4. 可选分钟行情仅用于低频策略辅助执行和行情确认。
5. 数据异常必须被识别，不得静默入库。
```

---

### 5.2 indicator 指标模块

负责：

```text
MA
EMA
MACD
RSI
BOLL
ATR
收益率
波动率
成交量均线
```

要求：

```text
1. 指标计算必须按时间顺序。
2. 指标不得使用未来数据。
3. rolling 窗口不得包含未来日期。
4. 指标参数必须可配置。
5. 指标结果应支持落库和查询。
```

---

### 5.3 factor 因子模块

负责：

```text
动量因子
反转因子
波动率因子
成交量因子
趋势因子
风险因子
综合因子评分
```

要求：

```text
1. 因子不能使用未来收益作为特征。
2. 因子计算必须支持版本追踪。
3. 因子结果应可落库。
4. 因子标准化和去极值逻辑应可配置。
5. 因子打分应可解释。
```

---

### 5.4 strategy 策略模块

负责：

```text
策略定义
策略参数
策略实例
策略信号
目标仓位
信号强度
信号解释
```

策略只能输出：

```text
buy
sell
reduce
clear
hold
watch
target_position
signal_strength
reason
```

策略不得：

```text
1. 直接调用实盘下单接口。
2. 直接调用券商接口。
3. 直接修改账户资金。
4. 直接修改持仓。
5. 绕过风控。
6. 在策略内部写订单提交逻辑。
```

正确链路：

```text
策略信号
   ↓
交易计划
   ↓
仓位管理
   ↓
风控检查
   ↓
订单模块
   ↓
券商接口
```

错误链路：

```text
策略信号
   ↓
券商接口
```

该错误链路必须禁止。

---

### 5.5 backtest 回测模块

负责：

```text
历史回测
模拟订单
模拟成交
交易成本计算
A 股规则模拟
资产曲线
绩效指标
回测报告
```

默认回测规则：

```text
T 日收盘后生成信号
T+1 日开盘或指定价格执行
```

必须模拟：

```text
1. 手续费。
2. 印花税。
3. 滑点。
4. 停牌。
5. 涨停。
6. 跌停。
7. T+1。
8. 买入 100 股整数倍。
9. 卖出不得超过可卖数量。
10. 最低现金比例。
11. 单票仓位限制。
12. 总仓位限制。
```

不得：

```text
1. 默认不计交易成本。
2. 默认允许涨停买入。
3. 默认允许跌停卖出。
4. 默认忽略 T+1。
5. 默认使用未来价格成交。
```

---

### 5.6 optimizer 参数优化模块

负责：

```text
参数空间定义
网格搜索
随机搜索
Optuna 优化
目标函数
样本内训练
样本外验证
优化结果记录
```

优化目标不能只看收益率，必须考虑：

```text
年化收益
最大回撤
夏普比率
换手率
集中度
交易频率
样本外表现
```

默认综合评分建议：

```text
score = annual_return * 0.35
      + sharpe_ratio * 0.25
      - max_drawdown * 0.25
      - turnover_penalty * 0.10
      - concentration_penalty * 0.05
```

---

### 5.7 simulation 模拟盘模块

负责：

```text
模拟账户
模拟交易计划
模拟订单
模拟成交
模拟持仓
模拟风控
模拟盘报告
模拟盘资产曲线
```

要求：

```text
1. 模拟盘不得调用真实券商下单接口。
2. 模拟盘应尽量复用实盘风控逻辑。
3. 模拟盘应模拟 A 股 T+1、停牌、涨跌停、手续费、印花税、滑点。
4. 模拟盘结果应作为进入实盘前的重要依据。
```

---

### 5.8 live 实盘模块

负责：

```text
券商接口适配
券商连接管理
账户同步
持仓同步
交易计划执行
实盘订单
实盘撤单
成交回报
订单状态同步
异常恢复
重复下单防护
```

实盘下单必须满足：

```text
1. 实盘交易开关开启。
2. 未触发一键停止。
3. 策略实例处于 running。
4. 交易计划已 approved。
5. 当前为交易日。
6. 当前在允许交易时段。
7. 券商连接正常。
8. 行情状态正常。
9. 账户同步成功。
10. 持仓同步成功。
11. 风控检查通过。
12. 不存在重复下单风险。
```

---

### 5.9 risk 风控模块

负责：

```text
全局风控
账户级风控
策略级风控
订单级风控
市场状态风控
A 股交易规则校验
交易频率限制
一键停止
风控事件记录
```

风控原则：

```text
状态不确定时，一律不自动交易。
```

风控拒绝时：

```text
不得调用券商下单接口。
```

以下情况必须拒绝实盘下单：

```text
1. 实盘交易开关关闭。
2. 一键停止已触发。
3. 券商连接异常。
4. 行情异常。
5. 账户同步失败。
6. 持仓同步失败。
7. 可用资金不足。
8. 可卖数量不足。
9. 股票停牌。
10. 非交易日。
11. 非交易时段。
12. 价格超出涨跌停范围。
13. 数量不符合规则。
14. 重复下单风险。
15. 订单状态 unknown。
```

---

### 5.10 report 报告模块

负责：

```text
回测报告
参数优化报告
模拟盘报告
实盘日报
交易统计
风控事件统计
资产曲线
收益回撤统计
```

报告至少应包含：

```text
总收益率
年化收益率
最大回撤
夏普比率
胜率
盈亏比
交易次数
换手率
手续费
印花税
滑点成本
最终资产
风控事件
策略频率
```

---

### 5.11 monitor 监控告警模块

负责监控：

```text
API 服务状态
数据库状态
Redis 状态
任务队列状态
行情状态
券商连接状态
账户同步状态
持仓同步状态
订单状态
策略状态
风控事件
异常频率
```

以下情况必须告警：

```text
1. 券商断线。
2. 行情异常。
3. 账户同步失败。
4. 持仓同步失败。
5. 订单状态 unknown。
6. 重复下单风险。
7. 一键停止触发。
8. 策略异常高频。
9. 风控模块异常。
```

---

### 5.12 audit 审计模块

必须审计：

```text
启用实盘交易
关闭实盘交易
修改策略参数
修改风控配置
生成交易计划
审批交易计划
取消交易计划
实盘下单
实盘撤单
一键停止
恢复交易
订单 unknown 恢复
成交回报处理
```

审计日志字段至少包括：

```text
actor
action
target_type
target_id
before_json
after_json
detail_json
created_at
```

---

## 6. 实盘安全规则

所有 Agent、自动化工具和开发者必须遵守：

```text
1. 实盘交易默认关闭。
2. 策略不得直接下单。
3. 交易计划不等于订单。
4. 风控不通过绝不下单。
5. 账户同步失败不得下单。
6. 持仓同步失败不得下单。
7. 行情异常不得下单。
8. 券商断线不得下单。
9. 订单 unknown 时不得继续自动下单。
10. 一键停止后不得生成新实盘订单。
11. 下单、撤单、风控拒绝必须记录审计日志。
12. 成交回报不得重复入库。
13. 程序重启后必须先同步订单和成交，再恢复交易。
```

---

## 7. A 股交易规则要求

系统必须遵守以下 A 股规则：

```text
1. 非交易日不交易。
2. 非交易时段不交易。
3. 停牌股票不交易。
4. 买入数量应符合 100 股整数倍。
5. 卖出数量不得超过可卖数量。
6. 当日买入股票不可当日卖出。
7. 委托价格不得超出涨跌停范围。
8. 涨停买入和跌停卖出需要保守处理。
```

卖出判断必须使用：

```text
available_quantity
```

不得使用：

```text
total_quantity
```

代替可卖数量。

---

## 8. 数据库规则

数据库设计以 PostgreSQL 为主。

关键唯一约束：

```text
stock_daily(symbol, trade_date) UNIQUE
stock_indicator_daily(symbol, trade_date) UNIQUE
factor_daily(symbol, trade_date, factor_version) UNIQUE
strategy_params(strategy_id, param_hash) UNIQUE
live_order(client_order_id) UNIQUE
live_trade(broker_account_id, broker_trade_id) UNIQUE，若 broker_trade_id 可用
```

金融数值字段应使用：

```text
numeric
```

不要使用 float 存储以下数据：

```text
金额
价格
收益率
仓位比例
手续费
印花税
滑点
因子分数
```

涉及以下操作必须使用事务：

```text
1. 创建实盘订单和记录风控结果。
2. 写入成交记录和更新订单状态。
3. 一键停止和写入审计日志。
4. 风控配置变更和写入审计日志。
5. 订单恢复和写入审计日志。
```

---

## 9. API 设计规则

API 统一前缀建议：

```text
/api/v1
```

通用响应格式：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

API 层职责：

```text
1. 参数校验。
2. 权限预留。
3. 调用 service。
4. 返回统一响应。
```

API 层不得：

```text
1. 直接写复杂业务逻辑。
2. 直接操作数据库细节。
3. 绕过 service 调用券商接口。
4. 绕过 risk 提交实盘订单。
```

实盘相关接口必须审计：

```text
POST /api/v1/live/trading/enable
POST /api/v1/live/trading/disable
POST /api/v1/live/order/place
POST /api/v1/live/order/cancel
POST /api/v1/risk/emergency-stop
POST /api/v1/risk/resume
POST /api/v1/risk/config/update
```

---

## 10. 代码生成规则

Agent 生成代码时必须遵守：

```text
1. 代码必须清晰、可维护。
2. 不要把业务逻辑写在 API 路由中。
3. API 层只做参数校验和调用 service。
4. service 层负责业务流程。
5. repository 层负责数据库访问。
6. strategy 层不得调用 live/order。
7. risk 层必须独立存在。
8. live/order 提交前必须调用 risk。
9. 所有实盘关键操作必须写审计日志。
10. 所有异常必须明确处理，不得静默失败。
11. 资金、价格、收益率等金融数值不得使用 float 持久化。
12. 不得引入高频交易相关能力。
```

推荐分层：

```text
api      接口层
service  应用服务层
domain   领域逻辑层
repo     数据访问层
model    ORM 模型
schema   请求响应模型
task     异步任务
```

---

## 11. 测试要求

新增或修改核心模块时，应补充测试。

必须测试：

```text
数据导入
指标计算
因子计算
策略信号
仓位计算
回测交易
交易成本
A 股规则
模拟盘订单
实盘 Mock 下单
风控拒单
一键停止
重复下单
订单 unknown 恢复
审计日志
```

P0 测试包括：

```text
1. 风控拒绝后不得下单。
2. 一键停止后不得下单。
3. 资金不足不得买入。
4. 可卖数量不足不得卖出。
5. 订单 unknown 后不得继续自动下单。
6. 成交回报不得重复入库。
7. 策略不得直接调用券商接口。
8. 账户同步失败不得下单。
9. 持仓同步失败不得下单。
10. 非交易日和非交易时段不得下单。
```

测试失败处理：

```text
1. P0 失败不得进入实盘。
2. P1 失败不得验收通过。
3. Critical 缺陷必须修复。
4. Blocker 缺陷必须立即处理。
```

---

## 12. 配置和密钥规则

禁止提交：

```text
.env
.env.local
.env.production
券商账号密码
API Key
Token
私钥
真实账户号
真实交易日志
真实成交记录
真实持仓截图
```

必须提供：

```text
.env.example
```

`.env.example` 中只能放示例值，不得放真实密钥。

示例：

```env
DATABASE_URL=postgresql://user:password@localhost:5432/quant
REDIS_URL=redis://localhost:6379/0
LIVE_TRADING_ENABLED=false
BROKER_NAME=mock
```

---

## 13. 日志规则

日志允许记录：

```text
任务状态
策略 ID
策略实例 ID
订单 ID
风控结果
错误摘要
审计事件
```

日志禁止记录：

```text
券商密码
API 密钥
完整账户号
身份证号
验证码
真实 token
真实私钥
```

实盘日志必须能追溯：

```text
策略信号
交易计划
风控结果
本地订单
券商订单号
成交记录
账户快照
持仓快照
```

---

## 14. 文档规则

项目文档建议放在：

```text
docs/
```

已有文档：

```text

docs/SRS 软件需求规格说明书.md
docs/系统概要设计说明书.md
docs/数据库设计说明书.md
docs/策略设计说明书.md
docs/回测引擎设计说明书.md
docs/模拟盘设计说明书.md
docs/实盘交易接入设计说明书.md
docs/风控设计说明书.md
docs/API 接口设计文档.md
docs/测试计划与验收标准.md
```

修改核心设计时，应同步更新相关文档。

---

## 15. Git 提交建议

提交信息建议格式：

```text
feat: add backtest engine
fix: prevent duplicate live orders
docs: update risk design
test: add T+1 rule tests
refactor: split order service
chore: update dependencies
```

常用类型：

```text
feat      新功能
fix       修复问题
docs      文档修改
test      测试相关
refactor  重构
chore     构建或杂项
style     格式调整
perf      性能优化
```

---

## 16. 禁止行为清单

禁止 Agent 或开发者执行以下行为：

```text
1. 绕过风控调用券商下单。
2. 在策略模块中直接写实盘下单逻辑。
3. 提交真实账户密钥。
4. 删除审计日志。
5. 忽略订单 unknown 状态。
6. 在账户同步失败时继续下单。
7. 在持仓同步失败时继续卖出。
8. 使用未来数据做回测。
9. 忽略手续费、印花税和滑点。
10. 增加高频交易相关功能。
11. 默认开启实盘交易。
12. 在风控异常时放行订单。
13. 在行情异常时继续交易。
14. 在券商断线时继续交易。
15. 让交易计划直接变成订单而不经过风控。
```

---

## 17. Agent 工作方式建议

当 AI Agent 修改本项目时，应遵守以下流程：

```text
1. 先阅读 AGENTS.md。
2. 再阅读相关 docs 文档。
3. 明确本次修改属于哪个模块。
4. 检查是否涉及实盘、安全、风控、资金、持仓。
5. 如果涉及实盘，必须检查风控链路。
6. 修改代码后补充或更新测试。
7. 修改核心设计后更新文档。
8. 不确定时选择保守行为，不自动交易。
```

涉及实盘模块时，Agent 必须优先确认：

```text
1. 是否会绕过风控？
2. 是否会重复下单？
3. 是否会在 unknown 状态继续交易？
4. 是否会在账户或持仓不同步时交易？
5. 是否有审计日志？
```

如果答案不明确，应停止实现并要求人工确认。

---

## 18. 最重要的原则

本项目最重要的原则：

```text
能回测，不代表能实盘；
能生成信号，不代表能下单；
能下单，也必须先过风控；
风控不通过，绝不下单；
状态不确定时，一律不自动交易；
系统长期只做低频交易。
```

---

## 19. Codex Vibe Coding 开发总规则

本项目决定主要依赖 Codex 进行 vibe coding 开发，但必须把“生成速度”约束在“交易安全、可追溯、可测试”的边界内。

Codex 在修改本项目时必须遵守：

```text
1. AGENTS.md 是最高优先级项目规则。
2. docs/ 是业务与架构事实来源。
3. 不额外维护其他模型或编辑器的专用规则。
4. 不默认新增 Git hooks 或其他写入 .git 的自动化。
5. 所有安全边界以本文件和 docs/ 为准。
```

当用户只提出模糊需求时，Agent 应主动把需求归入以下模块之一：

```text
data
indicator
factor
strategy
portfolio
backtest
optimizer
simulation
live
risk
monitor
audit
report
api
db
task
docs
test
```

如果需求涉及 `live`、`risk`、`broker`、`order`、`trade`、`account`、`position`、`emergency stop`、`unknown`、`真实资金`、`实盘开关`，必须按实盘安全规则处理。

---

## 20. Codex 必读文档路由

Codex 不需要每次通读全部文档，但必须按任务读取相关文档。

```text
需求、范围、阶段规划        -> docs/SRS 软件需求规格说明书.md
总体架构、模块边界          -> docs/系统概要设计说明书.md
数据库表、约束、事务        -> docs/数据库设计说明书.md
策略、信号、目标仓位        -> docs/策略设计说明书.md
回测、成交模拟、未来函数    -> docs/回测引擎设计说明书.md
模拟盘、模拟风控            -> docs/模拟盘设计说明书.md
实盘账户、订单、券商接入    -> docs/实盘交易接入设计说明书.md
风控、一键停止、拒单        -> docs/风控设计说明书.md
API 路径、响应、错误码      -> docs/API 接口设计文档.md
测试优先级、验收、不通过项  -> docs/测试计划与验收标准.md
```

涉及核心设计变更时，必须同步更新相应文档。只修改实现、不影响设计时，可以不改文档，但最终说明中要明确“不涉及设计文档变更”。

---

## 21. Vibe Coding 交付检查清单

每次代码修改完成前，Codex 必须自查：

```text
1. 是否仍然只支持低频交易？
2. 是否引入了 Tick、Level-2、订单簿、做市、毫秒级低延迟等能力？
3. 策略层是否只生成信号和目标仓位？
4. 交易计划是否仍然不等于订单？
5. 实盘下单前是否经过风控？
6. 风控拒绝时是否保证不调用券商接口？
7. 一键停止后是否不会生成新实盘订单？
8. unknown 订单是否暂停相关策略并等待恢复或人工确认？
9. 下单前是否强制同步账户资金和持仓？
10. 卖出判断是否使用 available_quantity？
11. 成交回报是否具备去重逻辑？
12. 实盘关键操作是否写审计日志？
13. 金融持久化字段是否使用 numeric/Decimal，而不是 float？
14. 是否补充或更新了对应测试？
15. 是否需要同步更新 docs/？
```

只要其中任一项答案不确定，默认采用保守行为：拒绝交易、停止自动化、要求人工确认或补充测试。

---

## 22. Codex 生成代码的硬性边界

### 22.1 策略层硬边界

策略模块允许：

```text
读取当前时点及以前的数据
计算指标、因子和信号
输出 target_position
输出 signal_strength
输出 reason
记录信号日志
```

策略模块禁止：

```text
导入 live order service
导入 broker adapter
调用 place_order
调用 cancel_order
修改账户资金
修改实盘持仓
生成真实订单
绕过交易计划
绕过风控
```

### 22.2 实盘订单硬边界

实盘订单模块必须遵守：

```text
1. risk check 通过之前，不得调用 broker。
2. 风控结果、订单创建、审计日志应在事务边界内保持一致。
3. broker 超时或无法确认时，订单状态必须为 unknown。
4. unknown 存在时，不得继续自动下单。
5. client_order_id 必须唯一。
6. 重试不得造成重复下单。
```

### 22.3 风控硬边界

风控模块必须默认拒绝不确定状态：

```text
行情未知 -> 拒绝
账户未知 -> 拒绝
持仓未知 -> 拒绝
券商连接未知 -> 拒绝
订单状态 unknown -> 停止自动下单
风控配置缺失 -> 拒绝
风控模块异常 -> 拒绝
```

---

## 23. Codex 自查边界

本项目当前不强制安装 Git hooks，也不维护其他模型或编辑器的专用 rules。Codex、自动化脚本和协作开发者应通过 `AGENTS.md` 和 `docs/` 自查以下风险：

```text
1. 不提交密钥、真实账户信息和私钥。
2. 不在非文档代码中引入高频交易能力。
3. strategy 模块不直接调用 live/broker/order。
4. 金融持久化字段不使用 Float/FLOAT/REAL/DOUBLE。
5. LIVE_TRADING_ENABLED 不得默认开启。
6. 核心代码变更后确认是否需要同步 docs/。
7. 提交信息遵守 feat/fix/docs/test/refactor/chore 等格式建议。
```

如果未来需要启用 hooks 或其他模型专用规则，应先得到项目负责人明确确认，再新增相关脚本或配置。

---

## 24. Vibe Coding 输出要求

Codex 完成任务时，应在最终回复中说明：

```text
1. 修改了哪些文件。
2. 属于哪些模块。
3. 是否涉及实盘、资金、持仓、风控或审计。
4. 运行了哪些检查或测试。
5. 如果没有测试，说明原因。
6. 如果未改文档，说明是否因为不涉及核心设计变更。
```

如果任务是 code review，应优先指出风险、缺陷、缺少测试和违反项目边界的问题。

---

## 25. 规则冲突处理

规则优先级如下：

```text
1. 用户明确的安全要求和人工确认要求。
2. AGENTS.md。
3. docs/ 中的设计文档。
4. 当前代码实现。
```

如果现有代码与 AGENTS.md 或 docs/ 冲突，Agent 不得默认沿用错误实现，应指出冲突并按更保守、更安全的规则修正。

如果用户要求实现的功能触碰永久排除范围，应拒绝实现该部分，并给出符合低频交易定位的替代方案。
