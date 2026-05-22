# A 股低频量化策略研究、回测、模拟盘与实盘执行系统 API 接口设计文档

## 1. 引言

### 1.1 文档目的

本文档用于描述“A 股低频量化策略研究、回测、模拟盘与实盘执行系统”的 API 接口设计方案，明确系统对外提供的接口分组、接口路径、请求方式、请求参数、响应格式、错误码、鉴权预留、分页规范、任务状态规范以及核心业务接口设计。

本文档主要回答以下问题：

```text
1. 系统有哪些 API 接口？
2. API 如何分组？
3. 请求和响应格式如何统一？
4. 数据、策略、回测、优化、模拟盘、实盘、风控和报告模块分别提供哪些接口？
5. 实盘下单接口如何保证必须经过风控？
6. API 如何返回任务状态、错误信息和业务结果？
7. 后续如何扩展认证、权限和前端调用？
```

---

## 2. API 总体设计

### 2.1 API 风格

系统采用 RESTful API 风格。

默认数据格式：

```text
Content-Type: application/json
```

基础路径建议：

```http
/api/v1
```

完整接口示例：

```http
POST /api/v1/backtest/run
GET  /api/v1/backtest/result/{task_id}
POST /api/v1/live/order/place
POST /api/v1/risk/emergency-stop
```

### 2.2 API 分组

系统 API 按业务模块划分为以下几组：

```text
1. 系统健康接口
2. 数据接口
3. 指标与因子接口
4. 策略接口
5. 回测接口
6. 参数优化接口
7. 模拟盘接口
8. 实盘账户接口
9. 实盘订单接口
10. 风控接口
11. 报告接口
12. 监控告警接口
13. 审计日志接口
```

---

## 3. 通用接口规范

### 3.1 通用响应格式

所有接口统一返回 JSON。

成功响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {}
}
```

失败响应：

```json
{
  "code": 40001,
  "message": "invalid parameters",
  "data": null
}
```

分页响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 100,
      "total_pages": 5
    }
  }
}
```

任务创建响应：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 1001,
    "status": "pending"
  }
}
```

---

## 3.2 通用请求头

建议所有请求支持以下请求头：

```http
Content-Type: application/json
X-Request-Id: unique-request-id
Authorization: Bearer <token>
```

第一版可以暂不实现完整登录鉴权，但接口结构应预留 Authorization。

### 3.3 请求 ID

`X-Request-Id` 用于追踪请求链路。

如果客户端不传，服务端可以自动生成。

实盘相关接口必须记录 request_id，方便审计和排查。

---

## 3.4 通用错误码

| 错误码   | 类型                       | 说明      |
| ----- | ------------------------ | ------- |
| 0     | success                  | 成功      |
| 40001 | invalid_parameters       | 参数错误    |
| 40002 | missing_required_field   | 缺少必填字段  |
| 40003 | invalid_date_range       | 日期范围错误  |
| 40004 | invalid_symbol           | 股票代码错误  |
| 40101 | unauthorized             | 未认证     |
| 40301 | forbidden                | 无权限     |
| 40401 | resource_not_found       | 资源不存在   |
| 40901 | resource_conflict        | 资源冲突    |
| 40902 | duplicated_request       | 重复请求    |
| 50001 | internal_error           | 系统内部错误  |
| 50002 | database_error           | 数据库错误   |
| 50003 | task_error               | 任务执行错误  |
| 60001 | broker_connection_failed | 券商连接失败  |
| 60002 | broker_query_failed      | 券商查询失败  |
| 60003 | broker_order_failed      | 券商下单失败  |
| 60004 | broker_cancel_failed     | 券商撤单失败  |
| 70001 | risk_rejected            | 风控拒绝    |
| 70002 | emergency_stopped        | 已触发一键停止 |
| 70003 | trading_disabled         | 实盘交易未启用 |
| 70004 | account_sync_failed      | 账户同步失败  |
| 70005 | position_not_enough      | 可卖持仓不足  |
| 70006 | cash_not_enough          | 可用资金不足  |
| 70007 | frequency_violation      | 交易频率异常  |
| 80001 | market_data_unavailable  | 行情不可用   |
| 80002 | market_data_delayed      | 行情延迟    |
| 80003 | non_trading_day          | 非交易日    |
| 80004 | non_trading_time         | 非交易时段   |
| 80005 | stock_suspended          | 股票停牌    |
| 80006 | price_out_of_limit       | 价格超出涨跌停 |

---

## 3.5 通用分页参数

列表接口统一支持：

```text
page
page_size
```

示例：

```http
GET /api/v1/strategy/list?page=1&page_size=20
```

默认值：

```text
page = 1
page_size = 20
```

最大 page_size 建议限制为：

```text
100
```

---

## 3.6 通用时间格式

日期格式：

```text
YYYY-MM-DD
```

时间格式：

```text
YYYY-MM-DD HH:mm:ss
```

示例：

```json
{
  "start_date": "2020-01-01",
  "end_date": "2024-12-31",
  "created_at": "2026-05-21 10:30:00"
}
```

---

## 3.7 通用任务状态

异步任务统一使用以下状态：

```text
pending
running
success
failed
canceled
```

说明：

| 状态       | 说明   |
| -------- | ---- |
| pending  | 等待执行 |
| running  | 正在执行 |
| success  | 执行成功 |
| failed   | 执行失败 |
| canceled | 已取消  |

---

# 4. 系统健康接口

## 4.1 健康检查

### 接口说明

用于检查 API 服务是否正常。

```http
GET /api/v1/health
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "status": "ok",
    "service": "quant-backend",
    "version": "1.0.0",
    "time": "2026-05-21 10:30:00"
  }
}
```

---

## 4.2 系统组件状态

### 接口说明

检查数据库、Redis、任务队列、行情源、券商接口等组件状态。

```http
GET /api/v1/health/components
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "database": "ok",
    "redis": "ok",
    "task_queue": "ok",
    "market_data": "ok",
    "broker_gateway": "disconnected"
  }
}
```

---

# 5. 数据接口

## 5.1 导入股票基础信息

### 接口说明

导入股票基础信息。

```http
POST /api/v1/data/stocks/import
```

### 请求参数

```json
{
  "file_path": "data/stock_info.csv",
  "source": "local_csv"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 1001,
    "status": "pending"
  }
}
```

---

## 5.2 查询股票列表

```http
GET /api/v1/data/stocks
```

### 查询参数

| 参数        | 类型      | 必填 | 说明              |
| --------- | ------- | -- | --------------- |
| exchange  | string  | 否  | 交易所，如 SH、SZ     |
| industry  | string  | 否  | 行业              |
| status    | string  | 否  | listed、delisted |
| page      | integer | 否  | 页码              |
| page_size | integer | 否  | 每页数量            |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "symbol": "600519.SH",
        "name": "贵州茅台",
        "exchange": "SH",
        "industry": "食品饮料",
        "list_date": "2001-08-27",
        "status": "listed"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 1,
      "total_pages": 1
    }
  }
}
```

---

## 5.3 查询单只股票基础信息

```http
GET /api/v1/data/stocks/{symbol}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "symbol": "600519.SH",
    "name": "贵州茅台",
    "exchange": "SH",
    "market": "main",
    "industry": "食品饮料",
    "list_date": "2001-08-27",
    "status": "listed"
  }
}
```

---

## 5.4 导入日线行情数据

```http
POST /api/v1/data/daily/import
```

### 请求参数

```json
{
  "file_path": "data/stock_daily.csv",
  "source": "local_csv",
  "adjust_type": "qfq",
  "overwrite": false
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 1002,
    "status": "pending"
  }
}
```

---

## 5.5 查询日线行情

```http
GET /api/v1/data/daily/{symbol}
```

### 查询参数

| 参数         | 类型      | 必填 | 说明       |
| ---------- | ------- | -- | -------- |
| start_date | string  | 是  | 开始日期     |
| end_date   | string  | 是  | 结束日期     |
| adjusted   | boolean | 否  | 是否使用复权价格 |

### 示例

```http
GET /api/v1/data/daily/600519.SH?start_date=2024-01-01&end_date=2024-12-31
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "symbol": "600519.SH",
    "items": [
      {
        "trade_date": "2024-01-02",
        "open": 1700.0,
        "high": 1720.0,
        "low": 1680.0,
        "close": 1710.0,
        "volume": 1200000,
        "amount": 2050000000,
        "is_suspended": false,
        "is_limit_up": false,
        "is_limit_down": false
      }
    ]
  }
}
```

---

## 5.6 查询交易日历

```http
GET /api/v1/data/calendar
```

### 查询参数

| 参数         | 类型     | 必填 | 说明         |
| ---------- | ------ | -- | ---------- |
| start_date | string | 是  | 开始日期       |
| end_date   | string | 是  | 结束日期       |
| market     | string | 否  | 默认 A_SHARE |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "trade_date": "2026-05-21",
        "market": "A_SHARE",
        "is_trading_day": true,
        "day_type": "normal"
      }
    ]
  }
}
```

---

# 6. 指标与因子接口

## 6.1 计算技术指标

```http
POST /api/v1/indicator/calculate
```

### 请求参数

```json
{
  "symbols": ["600519.SH", "000001.SZ"],
  "start_date": "2020-01-01",
  "end_date": "2024-12-31",
  "indicators": ["ma", "macd", "rsi", "boll", "atr"],
  "save_result": true
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 1101,
    "status": "pending"
  }
}
```

---

## 6.2 查询技术指标

```http
GET /api/v1/indicator/{symbol}
```

### 查询参数

```text
start_date
end_date
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "symbol": "600519.SH",
    "items": [
      {
        "trade_date": "2024-01-02",
        "ma5": 1700.1,
        "ma20": 1688.5,
        "ma60": 1650.2,
        "macd": 1.23,
        "rsi14": 56.7,
        "boll_upper": 1750.0,
        "boll_mid": 1688.5,
        "boll_lower": 1620.0
      }
    ]
  }
}
```

---

## 6.3 计算因子

```http
POST /api/v1/factor/calculate
```

### 请求参数

```json
{
  "symbols": ["600519.SH", "000001.SZ"],
  "start_date": "2020-01-01",
  "end_date": "2024-12-31",
  "factors": [
    "return_5d",
    "return_20d",
    "return_60d",
    "volatility_20d",
    "volume_ratio_20d",
    "max_drawdown_20d"
  ],
  "save_result": true
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 1201,
    "status": "pending"
  }
}
```

---

## 6.4 查询因子

```http
GET /api/v1/factor/{symbol}
```

### 查询参数

```text
start_date
end_date
factor_version
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "symbol": "600519.SH",
    "items": [
      {
        "trade_date": "2024-01-02",
        "return_5d": 0.012,
        "return_20d": 0.045,
        "return_60d": 0.102,
        "volatility_20d": 0.021,
        "factor_score": 0.76
      }
    ]
  }
}
```

---

# 7. 策略接口

## 7.1 创建策略

```http
POST /api/v1/strategy/create
```

### 请求参数

```json
{
  "name": "trend_following_v1",
  "type": "trend_following",
  "frequency": "daily",
  "description": "基于均线、MACD 和成交量的趋势跟踪策略"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "strategy_id": 1
  }
}
```

---

## 7.2 查询策略列表

```http
GET /api/v1/strategy/list
```

### 查询参数

| 参数        | 类型      | 必填 | 说明                   |
| --------- | ------- | -- | -------------------- |
| type      | string  | 否  | 策略类型                 |
| status    | string  | 否  | active、disabled      |
| frequency | string  | 否  | daily、weekly、monthly |
| page      | integer | 否  | 页码                   |
| page_size | integer | 否  | 每页数量                 |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "strategy_id": 1,
        "name": "trend_following_v1",
        "type": "trend_following",
        "frequency": "daily",
        "status": "active",
        "created_at": "2026-05-21 10:30:00"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 1,
      "total_pages": 1
    }
  }
}
```

---

## 7.3 查询策略详情

```http
GET /api/v1/strategy/{strategy_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "strategy_id": 1,
    "name": "trend_following_v1",
    "type": "trend_following",
    "frequency": "daily",
    "description": "基于均线、MACD 和成交量的趋势跟踪策略",
    "status": "active",
    "created_at": "2026-05-21 10:30:00"
  }
}
```

---

## 7.4 保存策略参数

```http
POST /api/v1/strategy/{strategy_id}/params
```

### 请求参数

```json
{
  "description": "趋势策略第一版参数",
  "params": {
    "ma_short": 20,
    "ma_long": 60,
    "buy_macd_threshold": 0,
    "buy_volume_multiplier": 1.2,
    "sell_ma_period": 20,
    "stop_loss_ratio": 0.08,
    "take_profit_ratio": 0.25,
    "trailing_stop_ratio": 0.10,
    "initial_position_ratio": 0.10,
    "max_position_per_stock": 0.20,
    "max_total_position": 0.80,
    "min_cash_ratio": 0.20,
    "max_signals_per_day": 50,
    "max_orders_per_day": 20
  }
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "params_id": 10,
    "param_hash": "a8f3d9..."
  }
}
```

---

## 7.5 查询策略参数列表

```http
GET /api/v1/strategy/{strategy_id}/params
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "params_id": 10,
        "description": "趋势策略第一版参数",
        "param_hash": "a8f3d9...",
        "created_at": "2026-05-21 10:30:00"
      }
    ]
  }
}
```

---

## 7.6 创建策略实例

```http
POST /api/v1/strategy/instance/create
```

### 请求参数

```json
{
  "strategy_id": 1,
  "params_id": 10,
  "mode": "simulation",
  "capital_limit": 100000,
  "risk_config": {
    "max_position_per_stock": 0.1,
    "max_orders_per_day": 10
  }
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "strategy_instance_id": 100
  }
}
```

---

## 7.7 启用策略实例

```http
POST /api/v1/strategy/instance/{instance_id}/start
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "strategy_instance_id": 100,
    "status": "running"
  }
}
```

---

## 7.8 暂停策略实例

```http
POST /api/v1/strategy/instance/{instance_id}/pause
```

### 请求参数

```json
{
  "reason": "manual pause"
}
```

---

## 7.9 停止策略实例

```http
POST /api/v1/strategy/instance/{instance_id}/stop
```

### 请求参数

```json
{
  "reason": "manual stop"
}
```

---

## 7.10 查询策略信号

```http
GET /api/v1/strategy/signals
```

### 查询参数

| 参数                   | 类型      | 必填 | 说明                    |
| -------------------- | ------- | -- | --------------------- |
| strategy_id          | integer | 否  | 策略 ID                 |
| strategy_instance_id | integer | 否  | 策略实例 ID               |
| symbol               | string  | 否  | 股票代码                  |
| signal_type          | string  | 否  | buy、sell、reduce、clear |
| start_date           | string  | 否  | 开始日期                  |
| end_date             | string  | 否  | 结束日期                  |

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "signal_id": 501,
        "strategy_id": 1,
        "strategy_instance_id": 100,
        "symbol": "600519.SH",
        "signal_date": "2026-05-20",
        "signal_type": "buy",
        "signal_strength": 0.75,
        "target_position": 0.1,
        "reason": "close > ma20 and ma20 > ma60",
        "source_mode": "simulation"
      }
    ]
  }
}
```

---

# 8. 回测接口

## 8.1 创建回测任务

```http
POST /api/v1/backtest/run
```

### 请求参数

```json
{
  "strategy_id": 1,
  "params_id": 10,
  "symbols": ["600519.SH", "000001.SZ"],
  "start_date": "2020-01-01",
  "end_date": "2024-12-31",
  "initial_cash": 1000000,
  "benchmark": "000300.SH",
  "cost_config": {
    "buy_commission_rate": 0.0003,
    "sell_commission_rate": 0.0003,
    "stamp_tax_rate": 0.0005,
    "slippage_rate": 0.001,
    "min_commission": 5
  },
  "execution_config": {
    "price_type": "next_open",
    "simulate_t_plus_one": true,
    "simulate_limit_up_down": true,
    "simulate_suspension": true
  }
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 2001,
    "status": "pending"
  }
}
```

---

## 8.2 查询回测任务状态

```http
GET /api/v1/backtest/task/{task_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 2001,
    "status": "running",
    "progress": 0.45,
    "created_at": "2026-05-21 10:30:00",
    "started_at": "2026-05-21 10:31:00",
    "finished_at": null,
    "error_message": null
  }
}
```

---

## 8.3 取消回测任务

```http
POST /api/v1/backtest/task/{task_id}/cancel
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 2001,
    "status": "canceled"
  }
}
```

---

## 8.4 查询回测结果

```http
GET /api/v1/backtest/result/{task_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "task_id": 2001,
    "strategy_id": 1,
    "params_id": 10,
    "total_return": 0.86,
    "annual_return": 0.18,
    "max_drawdown": 0.16,
    "sharpe_ratio": 1.25,
    "calmar_ratio": 1.12,
    "win_rate": 0.56,
    "profit_loss_ratio": 1.48,
    "trade_count": 180,
    "turnover_rate": 0.72,
    "total_commission": 5200.35,
    "final_value": 1860000,
    "score": 0.68
  }
}
```

---

## 8.5 查询回测交易记录

```http
GET /api/v1/backtest/trades/{task_id}
```

### 查询参数

```text
symbol
side
start_date
end_date
page
page_size
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "trade_date": "2021-03-01",
        "symbol": "600519.SH",
        "side": "buy",
        "price": 1600.0,
        "quantity": 100,
        "amount": 160000,
        "commission": 48,
        "tax": 0,
        "slippage": 160,
        "reason": "strategy_buy"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 180,
      "total_pages": 9
    }
  }
}
```

---

## 8.6 查询回测资产曲线

```http
GET /api/v1/backtest/equity-curve/{task_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "trade_date": "2021-03-01",
        "total_value": 1005000,
        "cash": 840000,
        "market_value": 165000,
        "daily_return": 0.005,
        "drawdown": 0.0,
        "position_ratio": 0.164
      }
    ]
  }
}
```

---

# 9. 参数优化接口

## 9.1 创建参数优化任务

```http
POST /api/v1/optimize/run
```

### 请求参数

```json
{
  "strategy_id": 1,
  "symbols": ["600519.SH", "000001.SZ"],
  "train_start_date": "2018-01-01",
  "train_end_date": "2022-12-31",
  "valid_start_date": "2023-01-01",
  "valid_end_date": "2024-12-31",
  "initial_cash": 1000000,
  "method": "grid",
  "max_trials": 100,
  "param_space": {
    "ma_short": [5, 10, 20],
    "ma_long": [30, 60, 120],
    "stop_loss_ratio": [0.05, 0.08, 0.10],
    "take_profit_ratio": [0.15, 0.25, 0.35],
    "initial_position_ratio": [0.05, 0.10, 0.15]
  },
  "objective_config": {
    "annual_return_weight": 0.35,
    "sharpe_weight": 0.25,
    "max_drawdown_weight": 0.25,
    "turnover_penalty_weight": 0.10,
    "concentration_penalty_weight": 0.05
  }
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "optimize_task_id": 3001,
    "status": "pending"
  }
}
```

---

## 9.2 查询优化任务状态

```http
GET /api/v1/optimize/task/{task_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "optimize_task_id": 3001,
    "status": "running",
    "progress": 0.36,
    "finished_trials": 36,
    "max_trials": 100,
    "best_score": 0.71
  }
}
```

---

## 9.3 查询最优结果

```http
GET /api/v1/optimize/best-result/{task_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "optimize_task_id": 3001,
    "best_score": 0.71,
    "best_params": {
      "ma_short": 10,
      "ma_long": 60,
      "stop_loss_ratio": 0.08,
      "take_profit_ratio": 0.25,
      "initial_position_ratio": 0.10
    },
    "train_result": {
      "annual_return": 0.22,
      "max_drawdown": 0.15,
      "sharpe_ratio": 1.42
    },
    "valid_result": {
      "annual_return": 0.16,
      "max_drawdown": 0.18,
      "sharpe_ratio": 1.10
    }
  }
}
```

---

## 9.4 查询优化历史

```http
GET /api/v1/optimize/history/{task_id}
```

### 查询参数

```text
page
page_size
order_by
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "trial_no": 1,
        "params": {
          "ma_short": 5,
          "ma_long": 60
        },
        "annual_return": 0.15,
        "max_drawdown": 0.20,
        "sharpe_ratio": 0.95,
        "turnover_rate": 0.80,
        "score": 0.51,
        "status": "success"
      }
    ],
    "pagination": {
      "page": 1,
      "page_size": 20,
      "total": 100,
      "total_pages": 5
    }
  }
}
```

---

# 10. 模拟盘接口

## 10.1 创建模拟盘账户

```http
POST /api/v1/simulation/account/create
```

### 请求参数

```json
{
  "name": "trend_strategy_sim_account",
  "initial_cash": 1000000
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "simulation_account_id": 4001
  }
}
```

---

## 10.2 查询模拟盘账户

```http
GET /api/v1/simulation/account/{account_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "simulation_account_id": 4001,
    "name": "trend_strategy_sim_account",
    "initial_cash": 1000000,
    "current_cash": 800000,
    "market_value": 210000,
    "total_asset": 1010000,
    "status": "active"
  }
}
```

---

## 10.3 启动模拟盘策略

```http
POST /api/v1/simulation/start
```

### 请求参数

```json
{
  "simulation_account_id": 4001,
  "strategy_instance_id": 100,
  "symbols": ["600519.SH", "000001.SZ"],
  "start_date": "2026-05-21"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "simulation_account_id": 4001,
    "strategy_instance_id": 100,
    "status": "running"
  }
}
```

---

## 10.4 停止模拟盘策略

```http
POST /api/v1/simulation/stop
```

### 请求参数

```json
{
  "simulation_account_id": 4001,
  "strategy_instance_id": 100,
  "reason": "manual stop"
}
```

---

## 10.5 查询模拟盘订单

```http
GET /api/v1/simulation/orders/{account_id}
```

### 查询参数

```text
symbol
order_status
start_date
end_date
page
page_size
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "order_id": 5001,
        "symbol": "600519.SH",
        "side": "buy",
        "price": 1700,
        "quantity": 100,
        "filled_quantity": 100,
        "order_status": "filled",
        "created_at": "2026-05-21 10:30:00"
      }
    ]
  }
}
```

---

## 10.6 查询模拟盘成交

```http
GET /api/v1/simulation/trades/{account_id}
```

---

## 10.7 查询模拟盘报告

```http
GET /api/v1/simulation/report/{account_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "simulation_account_id": 4001,
    "total_return": 0.05,
    "max_drawdown": 0.03,
    "trade_count": 20,
    "win_rate": 0.55,
    "current_cash": 800000,
    "market_value": 250000,
    "total_asset": 1050000
  }
}
```

---

# 11. 实盘账户接口

## 11.1 连接券商接口

```http
POST /api/v1/live/connect
```

### 请求参数

```json
{
  "broker_account_id": 1
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "connection_status": "connected"
  }
}
```

---

## 11.2 断开券商接口

```http
POST /api/v1/live/disconnect
```

### 请求参数

```json
{
  "broker_account_id": 1
}
```

---

## 11.3 查询券商连接状态

```http
GET /api/v1/live/connection-status
```

### 查询参数

```text
broker_account_id
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "connection_status": "connected",
    "last_heartbeat_at": "2026-05-21 10:30:00"
  }
}
```

---

## 11.4 同步实盘账户

```http
POST /api/v1/live/account/sync
```

### 请求参数

```json
{
  "broker_account_id": 1
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "sync_status": "success",
    "snapshot_time": "2026-05-21 10:30:00"
  }
}
```

---

## 11.5 查询实盘账户资金

```http
GET /api/v1/live/account
```

### 查询参数

```text
broker_account_id
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "total_asset": 1000000,
    "available_cash": 300000,
    "frozen_cash": 0,
    "market_value": 700000,
    "snapshot_time": "2026-05-21 10:30:00"
  }
}
```

---

## 11.6 查询实盘持仓

```http
GET /api/v1/live/positions
```

### 查询参数

```text
broker_account_id
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "items": [
      {
        "symbol": "600519.SH",
        "total_quantity": 200,
        "available_quantity": 100,
        "frozen_quantity": 0,
        "today_buy_quantity": 100,
        "cost_price": 1600,
        "market_price": 1700,
        "market_value": 340000,
        "profit_loss": 20000
      }
    ]
  }
}
```

---

# 12. 实盘交易计划接口

## 12.1 生成实盘交易计划

```http
POST /api/v1/live/plan/generate
```

### 请求参数

```json
{
  "strategy_instance_id": 100,
  "plan_date": "2026-05-21",
  "execute_date": "2026-05-22",
  "symbols": ["600519.SH", "000001.SZ"]
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "plan_id": 9001,
    "plan_status": "pending",
    "order_count": 2
  }
}
```

---

## 12.2 查询交易计划详情

```http
GET /api/v1/live/plan/{plan_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "plan_id": 9001,
    "strategy_instance_id": 100,
    "plan_date": "2026-05-21",
    "execute_date": "2026-05-22",
    "plan_status": "pending",
    "orders": [
      {
        "symbol": "600519.SH",
        "side": "buy",
        "target_position": 0.1,
        "estimated_amount": 100000,
        "reason": "trend_signal"
      }
    ],
    "pre_risk_check": {
      "passed": true
    }
  }
}
```

---

## 12.3 审批交易计划

```http
POST /api/v1/live/plan/{plan_id}/approve
```

### 请求参数

```json
{
  "approved_by": "admin",
  "comment": "approved for execution"
}
```

---

## 12.4 取消交易计划

```http
POST /api/v1/live/plan/{plan_id}/cancel
```

### 请求参数

```json
{
  "reason": "manual cancel"
}
```

---

# 13. 实盘订单接口

## 13.1 实盘交易开关开启

```http
POST /api/v1/live/trading/enable
```

### 请求参数

```json
{
  "broker_account_id": 1,
  "confirm": true,
  "reason": "start small capital live trading"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "live_trading_enabled": true
  }
}
```

---

## 13.2 实盘交易开关关闭

```http
POST /api/v1/live/trading/disable
```

### 请求参数

```json
{
  "broker_account_id": 1,
  "reason": "manual disable"
}
```

---

## 13.3 实盘下单

### 接口说明

该接口用于创建真实订单。

注意：

```text
该接口必须先调用风控模块。
风控不通过时，不得调用券商接口。
策略模块不得直接调用该接口。
```

```http
POST /api/v1/live/order/place
```

### 请求参数

```json
{
  "broker_account_id": 1,
  "strategy_instance_id": 100,
  "plan_id": 9001,
  "signal_id": 501,
  "symbol": "600519.SH",
  "side": "buy",
  "price": 1700.00,
  "quantity": 100,
  "order_type": "limit"
}
```

### 成功响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "live_order_id": 7001,
    "client_order_id": "100-9001-600519.SH-buy-20260522-a8f3",
    "order_status": "submitted",
    "broker_order_id": "B202605220001",
    "risk_check": {
      "passed": true
    }
  }
}
```

### 风控拒绝响应示例

```json
{
  "code": 70001,
  "message": "risk rejected",
  "data": {
    "passed": false,
    "risk_level": "error",
    "action": "reject_order",
    "reject_reason": "available_cash_not_enough",
    "risk_items": [
      {
        "rule": "OR-005",
        "message": "买入金额超过可用资金"
      }
    ]
  }
}
```

---

## 13.4 实盘撤单

```http
POST /api/v1/live/order/cancel
```

### 请求参数

```json
{
  "broker_account_id": 1,
  "live_order_id": 7001,
  "reason": "manual cancel"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "live_order_id": 7001,
    "order_status": "cancel_requested"
  }
}
```

---

## 13.5 查询实盘订单详情

```http
GET /api/v1/live/order/{order_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "live_order_id": 7001,
    "client_order_id": "100-9001-600519.SH-buy-20260522-a8f3",
    "broker_order_id": "B202605220001",
    "symbol": "600519.SH",
    "side": "buy",
    "price": 1700,
    "quantity": 100,
    "filled_quantity": 0,
    "remaining_quantity": 100,
    "order_status": "submitted",
    "submitted_at": "2026-05-22 09:31:00"
  }
}
```

---

## 13.6 查询实盘订单列表

```http
GET /api/v1/live/orders
```

### 查询参数

```text
broker_account_id
strategy_instance_id
symbol
order_status
start_date
end_date
page
page_size
```

---

## 13.7 同步实盘订单状态

```http
POST /api/v1/live/order/sync
```

### 请求参数

```json
{
  "broker_account_id": 1,
  "trading_date": "2026-05-22"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "broker_account_id": 1,
    "synced_orders": 5,
    "unknown_orders": 0,
    "sync_status": "success"
  }
}
```

---

## 13.8 查询实盘成交

```http
GET /api/v1/live/trades
```

### 查询参数

```text
broker_account_id
strategy_instance_id
symbol
start_date
end_date
page
page_size
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "live_trade_id": 8001,
        "live_order_id": 7001,
        "broker_trade_id": "T202605220001",
        "symbol": "600519.SH",
        "side": "buy",
        "price": 1700,
        "quantity": 100,
        "amount": 170000,
        "commission": 51,
        "tax": 0,
        "trade_time": "2026-05-22 09:32:00"
      }
    ]
  }
}
```

---

# 14. 风控接口

## 14.1 查询风控配置

```http
GET /api/v1/risk/config
```

### 查询参数

```text
scope_type
scope_id
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "scope_type": "account",
    "scope_id": 1,
    "config": {
      "max_total_position": 0.8,
      "max_position_per_stock": 0.1,
      "min_cash_ratio": 0.2,
      "max_daily_loss_ratio": 0.05,
      "max_account_drawdown": 0.2,
      "max_order_amount": 100000,
      "max_orders_per_day": 20
    },
    "status": "active"
  }
}
```

---

## 14.2 更新风控配置

```http
POST /api/v1/risk/config/update
```

### 请求参数

```json
{
  "scope_type": "account",
  "scope_id": 1,
  "config": {
    "max_total_position": 0.8,
    "max_position_per_stock": 0.1,
    "min_cash_ratio": 0.2,
    "max_daily_loss_ratio": 0.05,
    "max_account_drawdown": 0.2,
    "max_order_amount": 100000,
    "max_orders_per_day": 20
  },
  "reason": "update live risk config"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "risk_config_id": 1,
    "status": "active"
  }
}
```

---

## 14.3 订单风控检查

```http
POST /api/v1/risk/check-order
```

### 请求参数

```json
{
  "broker_account_id": 1,
  "strategy_instance_id": 100,
  "symbol": "600519.SH",
  "side": "buy",
  "price": 1700,
  "quantity": 100,
  "order_type": "limit"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "passed": true,
    "risk_level": "info",
    "action": "allow",
    "risk_items": []
  }
}
```

---

## 14.4 一键停止

```http
POST /api/v1/risk/emergency-stop
```

### 请求参数

```json
{
  "reason": "manual emergency stop",
  "cancel_open_orders": false
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "global_trading_status": "stopped",
    "cancel_open_orders": false,
    "stopped_at": "2026-05-21 10:30:00"
  }
}
```

---

## 14.5 恢复交易

```http
POST /api/v1/risk/resume
```

### 请求参数

```json
{
  "confirm": true,
  "reason": "risk checked and manually resumed"
}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "global_trading_status": "running",
    "resumed_at": "2026-05-21 10:40:00"
  }
}
```

---

## 14.6 查询风控事件

```http
GET /api/v1/risk/events
```

### 查询参数

```text
broker_account_id
strategy_instance_id
event_type
severity
start_date
end_date
page
page_size
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "event_id": 9001,
        "event_type": "order_rejected",
        "severity": "error",
        "passed": false,
        "description": "买入金额超过可用资金",
        "action": "reject_order",
        "created_at": "2026-05-21 10:30:00"
      }
    ]
  }
}
```

---

# 15. 报告接口

## 15.1 查询回测报告

```http
GET /api/v1/report/backtest/{task_id}
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "summary": {
      "total_return": 0.86,
      "annual_return": 0.18,
      "max_drawdown": 0.16,
      "sharpe_ratio": 1.25,
      "win_rate": 0.56
    },
    "equity_curve_url": "/api/v1/backtest/equity-curve/2001",
    "trades_url": "/api/v1/backtest/trades/2001"
  }
}
```

---

## 15.2 查询优化报告

```http
GET /api/v1/report/optimize/{task_id}
```

---

## 15.3 查询模拟盘报告

```http
GET /api/v1/report/simulation/{account_id}
```

---

## 15.4 查询实盘日报

```http
GET /api/v1/report/live/daily
```

### 查询参数

```text
broker_account_id
date
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "date": "2026-05-21",
    "broker_account_id": 1,
    "total_asset": 1000000,
    "available_cash": 300000,
    "market_value": 700000,
    "daily_pnl": 5000,
    "daily_return": 0.005,
    "order_count": 5,
    "trade_count": 3,
    "risk_event_count": 1
  }
}
```

---

## 15.5 查询实盘订单统计

```http
GET /api/v1/report/live/orders
```

### 查询参数

```text
broker_account_id
start_date
end_date
```

---

## 15.6 查询实盘成交统计

```http
GET /api/v1/report/live/trades
```

---

# 16. 监控告警接口

## 16.1 查询告警列表

```http
GET /api/v1/alerts
```

### 查询参数

```text
alert_type
severity
status
start_date
end_date
page
page_size
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "alert_id": 10001,
        "alert_type": "broker_disconnected",
        "severity": "critical",
        "title": "券商接口断开",
        "message": "broker connection lost",
        "status": "open",
        "created_at": "2026-05-21 10:30:00"
      }
    ]
  }
}
```

---

## 16.2 处理告警

```http
POST /api/v1/alerts/{alert_id}/resolve
```

### 请求参数

```json
{
  "resolution": "broker reconnected",
  "resolved_by": "admin"
}
```

---

# 17. 审计日志接口

## 17.1 查询审计日志

```http
GET /api/v1/audit/logs
```

### 查询参数

```text
actor
action
target_type
target_id
start_date
end_date
page
page_size
```

### 响应示例

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "items": [
      {
        "audit_id": 1,
        "actor": "admin",
        "action": "enable_live_trading",
        "target_type": "broker_account",
        "target_id": "1",
        "detail_json": {
          "reason": "start small capital test"
        },
        "created_at": "2026-05-21 10:30:00"
      }
    ]
  }
}
```

---

# 18. 实盘接口安全限制

## 18.1 实盘下单限制

实盘下单接口必须满足：

```text
1. 实盘交易开关已开启。
2. 未触发一键停止。
3. 券商接口连接正常。
4. 行情状态正常。
5. 账户资金同步成功。
6. 持仓同步成功。
7. 风控检查通过。
8. 不存在重复订单风险。
```

否则接口必须返回错误，不得提交券商。

---

## 18.2 禁止策略直接下单

策略模块不得直接调用：

```http
POST /api/v1/live/order/place
```

策略只能生成：

```text
策略信号
交易计划
目标仓位
```

真实订单必须由实盘交易服务调用。

---

## 18.3 实盘接口必须审计

以下接口必须写入审计日志：

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

# 19. API 鉴权与权限预留

第一版可不实现完整多用户权限，但接口设计应预留权限。

建议权限类型：

```text
data:read
data:write
strategy:read
strategy:write
backtest:run
optimize:run
simulation:run
live:read
live:trade
risk:read
risk:write
risk:emergency_stop
audit:read
```

实盘交易相关接口必须要求更高权限：

```text
live:trade
risk:emergency_stop
risk:write
```

---

# 20. API 调用流程示例

## 20.1 回测流程 API 调用

```text
1. POST /api/v1/data/daily/import
2. POST /api/v1/indicator/calculate
3. POST /api/v1/strategy/create
4. POST /api/v1/strategy/{strategy_id}/params
5. POST /api/v1/backtest/run
6. GET  /api/v1/backtest/task/{task_id}
7. GET  /api/v1/backtest/result/{task_id}
8. GET  /api/v1/backtest/trades/{task_id}
9. GET  /api/v1/backtest/equity-curve/{task_id}
```

---

## 20.2 参数优化流程 API 调用

```text
1. POST /api/v1/optimize/run
2. GET  /api/v1/optimize/task/{task_id}
3. GET  /api/v1/optimize/history/{task_id}
4. GET  /api/v1/optimize/best-result/{task_id}
```

---

## 20.3 模拟盘流程 API 调用

```text
1. POST /api/v1/simulation/account/create
2. POST /api/v1/strategy/instance/create
3. POST /api/v1/simulation/start
4. GET  /api/v1/simulation/account/{account_id}
5. GET  /api/v1/simulation/orders/{account_id}
6. GET  /api/v1/simulation/trades/{account_id}
7. GET  /api/v1/simulation/report/{account_id}
```

---

## 20.4 实盘流程 API 调用

```text
1. POST /api/v1/live/connect
2. POST /api/v1/live/account/sync
3. GET  /api/v1/live/account
4. GET  /api/v1/live/positions
5. POST /api/v1/live/plan/generate
6. GET  /api/v1/live/plan/{plan_id}
7. POST /api/v1/live/plan/{plan_id}/approve
8. POST /api/v1/live/trading/enable
9. POST /api/v1/risk/check-order
10. POST /api/v1/live/order/place
11. POST /api/v1/live/order/sync
12. GET  /api/v1/live/orders
13. GET  /api/v1/live/trades
14. GET  /api/v1/report/live/daily
```

---

# 21. API 低频交易边界设计

本系统永久不支持高频交易，因此 API 层不提供以下接口：

```text
Tick 行情接口
Level-2 盘口接口
逐笔成交接口
高频报单接口
高频撤单接口
做市报价接口
订单簿队列接口
低延迟专线接口
毫秒级行情订阅接口
```

系统允许的行情和交易接口仅服务于低频策略：

```text
日线数据查询
可选分钟级数据查询
实时或准实时价格确认
低频交易计划生成
低频实盘订单提交
低频订单状态同步
```

---

# 22. 总结

本文档完成了 A 股低频量化策略研究、回测、模拟盘与实盘执行系统的 API 接口设计。

本文档覆盖了：

```text
1. 通用 API 规范
2. 错误码设计
3. 分页规范
4. 任务状态规范
5. 系统健康接口
6. 数据接口
7. 指标与因子接口
8. 策略接口
9. 回测接口
10. 参数优化接口
11. 模拟盘接口
12. 实盘账户接口
13. 实盘交易计划接口
14. 实盘订单接口
15. 风控接口
16. 报告接口
17. 监控告警接口
18. 审计日志接口
19. 实盘接口安全限制
20. 权限预留
21. API 调用流程示例
22. 低频交易边界设计
```

系统 API 设计的核心原则是：

```text
策略只生成信号；
交易计划不等于订单；
实盘订单必须经过风控；
风控不通过绝不下单；
异常状态默认拒绝交易；
所有实盘关键操作必须审计；
系统长期只服务低频交易，不扩展高频接口。
```
