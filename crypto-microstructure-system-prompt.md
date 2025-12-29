# 加密货币微观结构量化交易系统 - 完整需求文档

## 一、系统定位与核心理念

### 1.1 系统定位
构建一个 **Microstructure-based Event-Driven Trading System**（微观结构驱动的事件交易系统），属于量化交易系统中的垂直分支——**高频微观结构分析与事件驱动系统**。

### 1.2 核心理念
- **不依赖传统技术指标预测**：摒弃 RSI/MACD 等滞后指标的"预测"逻辑
- **基于订单流毒性检测**：Order Flow Toxicity - 识别知情交易者的行为痕迹
- **基于流动性失衡分析**：Liquidity Imbalance - 捕捉买卖力量的结构性失衡
- **三层数据融合**：
  - 链上数据 (On-chain)：因 - 资金流向的根源
  - 盘面数据 (Market)：果 - 价格变动的表现
  - 隐藏订单 (Iceberg/Dark Pool)：暗 - 主力意图的隐藏信号

### 1.3 目标交易对
主要监控：`[交易对，如 SHELL/USDT]`
参考标的：`BTC/USDT`（用于联动分析）

---

## 二、子系统架构

### 2.1 System M - Market Watcher（盘面监控层）

**职责**：实时市场数据采集与基础指标计算

**功能模块**：
- 实时价格追踪（WebSocket 推送）
- 成交量异动检测（Volume Spike Detection）
- 基础技术指标计算（RSI, VWAP, ATR）
- 大额成交识别（Whale Transaction Detection）

**输出字段**：
```
[时间戳] ◉ 监控中 | 价: X.XXXX | RSI: XX.X | 量: X,XXX
```

**配置参数**：
```python
CONFIG_MARKET = {
    "refresh_interval": 5,          # 刷新频率（秒）
    "whale_threshold_usd": 5000,    # 大额成交阈值（美元）
    "volume_spike_multiplier": 3,   # 成交量异动倍数
    "rsi_period": 14,               # RSI 周期
    "log_path": "storage/logs/market.log"
}
```

---

### 2.2 System I - Iceberg Detector（冰山单检测层）

**职责**：隐藏大单识别与订单簿深度分析

**功能模块**：
- 隐藏大单识别（Hidden Order Detection）
- 挂单深度分析（Order Book Depth Analysis）
- 累积成交量追踪（Cumulative Fill Tracking）
- 冰山强度计算（Iceberg Intensity Score）

**检测逻辑**：
```
当某价位满足以下条件时，判定为冰山单：
1. 该价位反复成交，但可见挂单量不减少或恢复迅速
2. 累计成交量 / 可见挂单深度 > 阈值（如 5x）
3. 成交频率高于正常水平
```

**输出字段**：
```
[ICEBERG] ◉发现隐藏卖单 | 价格: X.XXXXXX | 累计成交: XXXX.XXU | 挂单深度: XXX.XXU | 强度: X.XXx
```

**配置参数**：
```python
CONFIG_ICEBERG = {
    "detection_window": 60,         # 检测窗口（秒）
    "intensity_threshold": 5.0,     # 强度阈值
    "min_cumulative_volume": 1000,  # 最小累计成交量（U）
    "price_tolerance": 0.0001       # 价格容差
}
```

---

### 2.3 System A - Chain Analyzer（链上分析层）

**职责**：链上数据监控与资金流向分析

**功能模块**：
- 交易所流入流出监控（Exchange Inflow/Outflow）
- 大额转账追踪（Large Transfer Tracking）
- 持仓分布变化（Holder Distribution Change）
- 链上活跃度指标（On-chain Activity Metrics）

**输出字段**：
```
[CHAIN] ▲ 大额流入交易所 | 数量: XXX,XXX | 来源: 0xXXXX... | 目标: Binance
```

**配置参数**：
```python
CONFIG_CHAIN = {
    "large_transfer_threshold": 100000,  # 大额转账阈值
    "exchange_wallets": [...],           # 已知交易所钱包列表
    "monitoring_interval": 30            # 监控间隔（秒）
}
```

**[链上] 状态标签**：

```python
def calculate_chain_state(chain_data: dict) -> str:
    """
    计算链上状态标签
    """
    inflow = chain_data.get('exchange_inflow', 0)    # 流入交易所
    outflow = chain_data.get('exchange_outflow', 0)  # 流出交易所
    net_flow = inflow - outflow

    # 大额转账检测
    large_transfers = chain_data.get('large_transfers', [])
    recent_large_inflow = sum([t['amount'] for t in large_transfers
                               if t['direction'] == 'inflow' and t['age_minutes'] < 30])
    recent_large_outflow = sum([t['amount'] for t in large_transfers
                                if t['direction'] == 'outflow' and t['age_minutes'] < 30])

    # 状态判定
    if recent_large_inflow > 100000:
        return '流入'      # 大额流入交易所（利空）
    elif recent_large_outflow > 100000:
        return '流出'      # 大额流出交易所（利多）
    elif net_flow > 50000:
        return '小幅流入'
    elif net_flow < -50000:
        return '小幅流出'
    else:
        return '中性'
```

**链上状态标签表**：

| 标签 | 含义 | 信号方向 | 说明 |
|-----|------|---------|------|
| `流入` | 大额流入交易所 | 看空 | 可能有抛压 |
| `流出` | 大额流出交易所 | 看多 | 可能有囤币 |
| `小幅流入` | 净流入但量不大 | 偏空 | 轻微抛压 |
| `小幅流出` | 净流出但量不大 | 偏多 | 轻微囤积 |
| `中性` | 无明显净流向 | 中性 | 观望 |

**显示格式**：
```
[链上: 中性]     # 无明显流向
[链上: 流入]     # 大额流入交易所，利空信号
[链上: 流出]     # 大额流出交易所，利多信号
```

---

### 2.4 System C - Command Center（战情指挥中心）

**职责**：多维度信号融合与共振判定

**功能模块**：
- 多源信号聚合（Signal Aggregation）
- 三维共振判定（M + I + A Resonance）
- 置信度评分（Confidence Score Calculation）
- 信号优先级排序（Signal Priority Ranking）

**共振判定逻辑**：
```
单一信号 (1/3)  → 仅记录，不触发交易    | 置信度: 20-40%
双重共振 (2/3)  → 低置信度警报，可选交易 | 置信度: 50-70%
三重共振 (3/3)  → 高置信度信号，建议交易 | 置信度: 80-95%
```

**状态显示**：
```
[SYSTEM C] ◉ 战情指挥中心已上线 | 模式: 全域三维共振判定
[INFO] 正在监听: Market(M) + Iceberg(I) + Chain(A)
[扫描中] HH:MM:SS | 状态: 全系统静默, 等待共振信号...
```

**配置参数**：
```python
CONFIG_COMMAND = {
    "resonance_weights": {
        "market": 0.35,
        "iceberg": 0.35,
        "chain": 0.30
    },
    "min_confidence_to_alert": 50,
    "min_confidence_to_trade": 70
}
```

---

### 2.5 System R - Replay Engine（复盘引擎）

**职责**：历史信号回测与策略评估

**功能模块**：
- 历史信号加载（Signal History Loader）
- 高频采样回测（High-frequency Backtest）
- 各类型信号统计（Signal Type Statistics）
- 报告生成（Report Generator）

**输出报告**：
```
┌─────────────────┬────────┬────────┬─────────┬───────────┬───────────┬───────────┐
│ 类型            │ 样本数 │ 胜率%  │ 盈亏比  │ 期望收益% │ 平均盈利% │ 平均亏损% │
├─────────────────┼────────┼────────┼─────────┼───────────┼───────────┼───────────┤
│ WHALE_BUY       │ 56     │ 3.57   │ 0.819   │ -0.424    │ 0.371     │ -0.453    │
│ WHALE_SELL      │ 51     │ 29.41  │ 0.515   │ -0.244    │ 0.227     │ -0.440    │
│ ICEBERG_BUY     │ 152    │ 1.32   │ 0.395   │ -0.376    │ 0.151     │ -0.383    │
│ ICEBERG_SELL    │ 154    │ 19.48  │ 0.868   │ -0.180    │ 0.245     │ -0.282    │
└─────────────────┴────────┴────────┴─────────┴───────────┴───────────┴───────────┘
```

---

## 三、信号类型定义

### 3.1 信号分类表

| 信号代码 | 名称 | 触发条件 | 方向 | 优先级 |
|---------|------|---------|------|--------|
| `WHALE_BUY` | 巨鲸买入 | 单笔成交 > 阈值 且为主动买入 (Taker Buy) | 多 | P1 |
| `WHALE_SELL` | 巨鲸卖出 | 单笔成交 > 阈值 且为主动卖出 (Taker Sell) | 空 | P1 |
| `ICEBERG_BUY` | 冰山买单 | 检测到隐藏买盘，强度 > 5x | 多 | P2 |
| `ICEBERG_SELL` | 冰山卖单 | 检测到隐藏卖盘，强度 > 5x | 空 | P2 |
| `STRONG_BULLISH` | 强势看多 | OBI > 0.7 且 CVD 持续为正 | 多 | P2 |
| `STRONG_BEARISH` | 强势看空 | OBI < -0.7 且 CVD 持续为负 | 空 | P2 |
| `SYMMETRY_BREAK_UP` | 对称性破坏(上) | OBI/CVD/VWAP 同步看多背离 | 多 | P1 |
| `SYMMETRY_BREAK_DOWN` | 对称性破坏(下) | OBI/CVD/VWAP 同步看空背离 | 空 | P1 |
| `LIQUIDITY_GRAB` | 流动性猎杀 | 快速突破关键位后立即反转 | 反向 | P1 |
| `CHAIN_INFLOW` | 链上流入 | 大额代币流入交易所 | 空 | P3 |
| `CHAIN_OUTFLOW` | 链上流出 | 大额代币流出交易所 | 多 | P3 |

### 3.2 复合信号

| 复合信号 | 组成 | 置信度加成 |
|---------|------|-----------|
| `WHALE_ICEBERG_COMBO` | WHALE + ICEBERG 同向 | +20% |
| `FULL_RESONANCE` | M + I + A 三重共振 | +35% |
| `TRAP_SIGNAL` | 诱多/诱空陷阱检测 | 反向信号 |

---

## 四、核心指标计算

### 4.1 订单簿失衡 (OBI - Order Book Imbalance)

```python
def calculate_obi(orderbook: dict) -> float:
    """
    计算订单簿失衡度
    返回值范围: [-1, 1]
    正值表示买盘强势，负值表示卖盘强势
    """
    bid_volume = sum([level['quantity'] for level in orderbook['bids'][:10]])
    ask_volume = sum([level['quantity'] for level in orderbook['asks'][:10]])

    if bid_volume + ask_volume == 0:
        return 0

    obi = (bid_volume - ask_volume) / (bid_volume + ask_volume)
    return round(obi, 4)
```

### 4.2 累积成交量差值 (CVD - Cumulative Volume Delta)

```python
def calculate_cvd(trades: list) -> float:
    """
    计算累积成交量差值
    正值表示主动买入占优，负值表示主动卖出占优
    """
    buy_volume = sum([t['quantity'] for t in trades if t['is_buyer_maker'] == False])
    sell_volume = sum([t['quantity'] for t in trades if t['is_buyer_maker'] == True])

    cvd = buy_volume - sell_volume
    return cvd
```

### 4.3 成交量加权平均价 (VWAP)

```python
def calculate_vwap(trades: list) -> float:
    """
    计算成交量加权平均价
    """
    total_value = sum([t['price'] * t['quantity'] for t in trades])
    total_volume = sum([t['quantity'] for t in trades])

    if total_volume == 0:
        return 0

    vwap = total_value / total_volume
    return round(vwap, 6)
```

### 4.4 订单流毒性 (Flow Toxicity)

```python
def calculate_flow_toxicity(trades: list) -> float:
    """
    计算订单流毒性
    高毒性表示知情交易者活跃
    返回值范围: [0, 1]
    """
    cvd = calculate_cvd(trades)
    total_volume = sum([t['quantity'] for t in trades])

    if total_volume == 0:
        return 0

    toxicity = abs(cvd) / total_volume
    return round(toxicity, 4)
```

### 4.5 量比 (VR - Volume Ratio)

```python
def calculate_volume_ratio(current_volume: float, avg_volume: float) -> float:
    """
    计算量比
    VR > 1: 当前成交量高于平均水平（放量）
    VR < 1: 当前成交量低于平均水平（缩量）
    VR > 2: 显著放量，可能有异动
    """
    if avg_volume == 0:
        return 0

    vr = current_volume / avg_volume
    return round(vr, 2)
```

**量比状态判定**：

| VR 范围 | 状态 | 含义 |
|--------|------|------|
| < 0.5 | 极度缩量 | 市场冷清，观望为主 |
| 0.5 - 1.0 | 缩量 | 成交低迷 |
| 1.0 - 2.0 | 正常/温和放量 | 正常交易状态 |
| 2.0 - 5.0 | 显著放量 | 有资金异动 |
| > 5.0 | 极端放量 | 重大事件或主力行为 |

---

### 4.6 价格斜率 (Price Slope)

```python
def calculate_price_slope(prices: list, window: int = 10) -> float:
    """
    计算价格斜率（趋势强度）
    正值表示上涨趋势，负值表示下跌趋势
    绝对值越大，趋势越强
    """
    if len(prices) < window:
        return 0

    recent_prices = prices[-window:]

    # 使用线性回归计算斜率
    x = list(range(window))
    x_mean = sum(x) / window
    y_mean = sum(recent_prices) / window

    numerator = sum((x[i] - x_mean) * (recent_prices[i] - y_mean) for i in range(window))
    denominator = sum((x[i] - x_mean) ** 2 for i in range(window))

    if denominator == 0:
        return 0

    slope = numerator / denominator

    # 标准化为百分比变化率
    normalized_slope = slope / y_mean
    return round(normalized_slope, 4)
```

**斜率状态判定**：

| 斜率范围 | 状态 | 含义 |
|---------|------|------|
| > 0.05 | 强势上涨 | 明确上升趋势 |
| 0.01 ~ 0.05 | 温和上涨 | 缓慢上升 |
| -0.01 ~ 0.01 | 横盘 | 无明显趋势 |
| -0.05 ~ -0.01 | 温和下跌 | 缓慢下降 |
| < -0.05 | 强势下跌 | 明确下降趋势 |

---

### 4.7 鲸鱼占比 (Whale Percentage)

```python
def calculate_whale_percentage(trades: list, whale_threshold_usd: float = 5000) -> float:
    """
    计算鲸鱼成交占比
    大额交易占总成交量的百分比
    """
    total_volume = sum([t['quantity'] * t['price'] for t in trades])
    whale_volume = sum([t['quantity'] * t['price'] for t in trades
                        if t['quantity'] * t['price'] >= whale_threshold_usd])

    if total_volume == 0:
        return 0

    whale_pct = whale_volume / total_volume * 100
    return round(whale_pct, 2)
```

**鲸鱼占比状态判定**：

| 鲸鱼占比 | 状态 | 含义 |
|---------|------|------|
| < 5% | 散户主导 | 小额交易为主 |
| 5% - 10% | 正常 | 混合交易状态 |
| 10% - 20% | 鲸鱼活跃 | 大户参与度高 |
| > 20% | 鲸鱼主导 | 主力控盘迹象 |

---

### 4.8 综合评分 (Composite Score)

```python
def calculate_composite_score(indicators: dict) -> int:
    """
    计算综合评分（0-100）
    综合多个指标生成单一分数，便于快速判断
    """
    score = 50  # 基准分

    # OBI 贡献（-15 ~ +15）
    obi = indicators.get('obi', 0)
    score += int(obi * 15)

    # CVD 方向贡献（-10 ~ +10）
    cvd = indicators.get('cvd', 0)
    if cvd > 0:
        score += min(10, int(cvd / 1000))
    else:
        score += max(-10, int(cvd / 1000))

    # 斜率贡献（-10 ~ +10）
    slope = indicators.get('slope', 0)
    score += int(slope * 100)

    # 量比贡献（0 ~ +10）
    vr = indicators.get('vr', 1)
    if vr > 2:
        score += min(10, int((vr - 1) * 5))

    # 鲸鱼占比贡献（-5 ~ +5，结合方向）
    whale_pct = indicators.get('whale_pct', 0)
    whale_direction = indicators.get('whale_direction', 'neutral')
    if whale_pct > 10:
        if whale_direction == 'buy':
            score += 5
        elif whale_direction == 'sell':
            score -= 5

    # 限制范围
    return max(0, min(100, score))
```

**综合评分解读**：

| 分数范围 | 状态 | 建议 |
|---------|------|------|
| 0 - 20 | 极度看空 | 强烈做空信号 |
| 20 - 40 | 看空 | 偏空操作 |
| 40 - 60 | 中性 | 观望或轻仓 |
| 60 - 80 | 看多 | 偏多操作 |
| 80 - 100 | 极度看多 | 强烈做多信号 |

---

### 4.9 对称性破坏检测 (Symmetry Break Detection)

```python
def check_symmetry_break(obi: float, cvd: float, price: float, vwap: float) -> dict:
    """
    检测市场对称性是否被破坏
    当三个指标同时指向同一方向时，触发信号
    """
    signals = {
        'obi_bullish': obi > 0.3,
        'obi_bearish': obi < -0.3,
        'cvd_bullish': cvd > 0,
        'cvd_bearish': cvd < 0,
        'price_above_vwap': price > vwap,
        'price_below_vwap': price < vwap
    }

    # 三重看多共振
    if signals['obi_bullish'] and signals['cvd_bullish'] and signals['price_above_vwap']:
        return {'break': True, 'direction': 'UP', 'confidence': 0.8}

    # 三重看空共振
    if signals['obi_bearish'] and signals['cvd_bearish'] and signals['price_below_vwap']:
        return {'break': True, 'direction': 'DOWN', 'confidence': 0.8}

    return {'break': False, 'direction': None, 'confidence': 0}
```

---

## 五、风险管理层

### 5.1 仓位管理

```python
POSITION_CONFIG = {
    "max_position_pct": 0.1,        # 单笔最大仓位：总资金的 10%
    "max_total_exposure": 0.3,      # 最大总敞口：总资金的 30%
    "use_kelly": True,              # 是否使用 Kelly 公式
    "kelly_fraction": 0.5           # Kelly 系数（半 Kelly 更保守）
}

def calculate_kelly_position(win_rate: float, win_loss_ratio: float) -> float:
    """
    Kelly 公式计算最优仓位
    f = (bp - q) / b
    其中: b = 盈亏比, p = 胜率, q = 1 - p
    """
    b = win_loss_ratio
    p = win_rate
    q = 1 - p

    kelly = (b * p - q) / b
    return max(0, min(kelly, POSITION_CONFIG['max_position_pct']))
```

### 5.2 止损机制

```python
STOP_LOSS_CONFIG = {
    "fixed_stop_pct": 0.02,         # 固定止损：-2%
    "atr_multiplier": 2.0,          # ATR 止损倍数
    "time_stop_minutes": 30,        # 时间止损：30分钟
    "trailing_stop_pct": 0.015      # 移动止损：1.5%
}

def calculate_stop_loss(entry_price: float, atr: float, method: str = 'atr') -> float:
    """
    计算止损价格
    """
    if method == 'fixed':
        return entry_price * (1 - STOP_LOSS_CONFIG['fixed_stop_pct'])
    elif method == 'atr':
        return entry_price - (atr * STOP_LOSS_CONFIG['atr_multiplier'])
    elif method == 'trailing':
        # 移动止损需要当前最高价
        pass
```

### 5.3 风控熔断

```python
RISK_LIMITS = {
    "max_daily_loss_pct": 0.05,     # 单日最大亏损：-5%
    "max_consecutive_losses": 5,    # 最大连续亏损次数
    "max_drawdown_pct": 0.15,       # 最大回撤：-15%
    "cooldown_minutes": 60          # 熔断冷却时间（分钟）
}

class RiskManager:
    def __init__(self):
        self.daily_pnl = 0
        self.consecutive_losses = 0
        self.peak_equity = 0
        self.is_halted = False

    def check_risk_limits(self, current_equity: float) -> dict:
        """
        检查是否触发风控限制
        """
        # 检查单日亏损
        if self.daily_pnl < -RISK_LIMITS['max_daily_loss_pct']:
            return {'halt': True, 'reason': 'MAX_DAILY_LOSS'}

        # 检查连续亏损
        if self.consecutive_losses >= RISK_LIMITS['max_consecutive_losses']:
            return {'halt': True, 'reason': 'CONSECUTIVE_LOSSES'}

        # 检查最大回撤
        drawdown = (self.peak_equity - current_equity) / self.peak_equity
        if drawdown > RISK_LIMITS['max_drawdown_pct']:
            return {'halt': True, 'reason': 'MAX_DRAWDOWN'}

        return {'halt': False, 'reason': None}
```

### 5.4 盈亏比要求

```python
TRADE_REQUIREMENTS = {
    "min_risk_reward": 1.5,         # 最低盈亏比 1.5:1
    "min_expected_value": 0,        # 期望值必须为正
    "min_confidence": 0.6           # 最低置信度 60%
}

def validate_trade(signal: dict) -> bool:
    """
    验证交易是否满足盈亏比要求
    """
    risk_reward = signal['target'] / signal['stop_loss']
    expected_value = (signal['win_rate'] * signal['target']) - ((1 - signal['win_rate']) * signal['stop_loss'])

    if risk_reward < TRADE_REQUIREMENTS['min_risk_reward']:
        return False
    if expected_value < TRADE_REQUIREMENTS['min_expected_value']:
        return False
    if signal['confidence'] < TRADE_REQUIREMENTS['min_confidence']:
        return False

    return True
```

---

## 六、市场环境识别

### 6.1 市场状态分类

| 状态代码 | 名称 | 特征 | 策略适配 |
|---------|------|------|---------|
| `TRENDING_UP` | 上升趋势 | ADX > 25, +DI > -DI | 顺势做多 |
| `TRENDING_DOWN` | 下降趋势 | ADX > 25, -DI > +DI | 顺势做空 |
| `RANGING` | 震荡市 | ADX < 20, 区间波动 | 高抛低吸/观望 |
| `HIGH_VOLATILITY` | 高波动 | ATR 突增 > 2倍均值 | 降低仓位 |
| `LOW_LIQUIDITY` | 流动性枯竭 | 盘口深度 < 阈值 | 暂停交易 |
| `SQUEEZE` | 波动收敛 | BB 带宽收窄 | 等待突破 |

### 6.2 环境判定代码

```python
def detect_market_regime(data: dict) -> str:
    """
    判定当前市场状态
    """
    adx = data['adx']
    plus_di = data['plus_di']
    minus_di = data['minus_di']
    atr = data['atr']
    atr_ma = data['atr_ma']
    orderbook_depth = data['orderbook_depth']

    # 流动性检查（最高优先级）
    if orderbook_depth < THRESHOLDS['min_liquidity']:
        return 'LOW_LIQUIDITY'

    # 波动率检查
    if atr > atr_ma * 2:
        return 'HIGH_VOLATILITY'

    # 趋势检查
    if adx > 25:
        if plus_di > minus_di:
            return 'TRENDING_UP'
        else:
            return 'TRENDING_DOWN'

    # 默认震荡
    return 'RANGING'
```

### 6.3 策略适配矩阵

```python
REGIME_STRATEGY_MAP = {
    'TRENDING_UP': {
        'allowed_signals': ['WHALE_BUY', 'ICEBERG_BUY', 'STRONG_BULLISH', 'SYMMETRY_BREAK_UP'],
        'position_multiplier': 1.0,
        'stop_loss_multiplier': 1.0
    },
    'TRENDING_DOWN': {
        'allowed_signals': ['WHALE_SELL', 'ICEBERG_SELL', 'STRONG_BEARISH', 'SYMMETRY_BREAK_DOWN'],
        'position_multiplier': 1.0,
        'stop_loss_multiplier': 1.0
    },
    'RANGING': {
        'allowed_signals': ['ICEBERG_BUY', 'ICEBERG_SELL'],  # 仅允许冰山单信号
        'position_multiplier': 0.5,
        'stop_loss_multiplier': 0.8
    },
    'HIGH_VOLATILITY': {
        'allowed_signals': ['SYMMETRY_BREAK_UP', 'SYMMETRY_BREAK_DOWN'],
        'position_multiplier': 0.3,
        'stop_loss_multiplier': 1.5
    },
    'LOW_LIQUIDITY': {
        'allowed_signals': [],  # 禁止所有交易
        'position_multiplier': 0,
        'stop_loss_multiplier': 0
    }
}
```

---

## 七、多时间框架共振

### 7.1 时间框架层级

```python
TIMEFRAME_CONFIG = {
    "macro": {
        "interval": "1d",           # 大周期：日线
        "purpose": "trend_filter",  # 用途：趋势过滤
        "weight": 0.5               # 权重：50%
    },
    "meso": {
        "interval": "4h",           # 中周期：4小时
        "purpose": "structure",     # 用途：结构判断（支撑阻力）
        "weight": 0.3               # 权重：30%
    },
    "micro": {
        "interval": "15m",          # 小周期：15分钟
        "purpose": "entry_trigger", # 用途：入场触发
        "weight": 0.2               # 权重：20%
    }
}
```

### 7.2 共振规则

```python
def check_mtf_resonance(signals: dict) -> dict:
    """
    多时间框架共振检查
    signals = {
        'macro': {'direction': 'LONG', 'strength': 0.8},
        'meso': {'direction': 'LONG', 'strength': 0.6},
        'micro': {'direction': 'LONG', 'strength': 0.7}
    }
    """
    directions = [s['direction'] for s in signals.values()]
    weights = [TIMEFRAME_CONFIG[tf]['weight'] for tf in signals.keys()]
    strengths = [s['strength'] for s in signals.values()]

    # 检查方向一致性
    if len(set(directions)) == 1:
        # 全部一致
        weighted_strength = sum([w * s for w, s in zip(weights, strengths)])
        return {
            'resonance': True,
            'direction': directions[0],
            'confidence': weighted_strength,
            'level': 'FULL'
        }
    elif directions.count(directions[0]) >= 2:
        # 2/3 一致
        majority_direction = max(set(directions), key=directions.count)
        return {
            'resonance': True,
            'direction': majority_direction,
            'confidence': 0.6,
            'level': 'PARTIAL'
        }
    else:
        return {
            'resonance': False,
            'direction': None,
            'confidence': 0,
            'level': 'NONE'
        }
```

### 7.3 极端状态指标 ([极买]/[极卖])

**定义**：在各时间框架方向判定后，附加的极端状态标签

```python
def calculate_extreme_state(indicators: dict, timeframe: str) -> str:
    """
    计算极端状态标签
    返回: '[极买]' / '[极卖]' / '[超买]' / '[超卖]' / ''
    """
    obi = indicators.get('obi', 0)
    cvd = indicators.get('cvd', 0)
    rsi = indicators.get('rsi', 50)
    price = indicators.get('price', 0)
    vwap = indicators.get('vwap', 0)

    # 极买条件：多个指标同时显示极端买入信号
    extreme_buy_conditions = [
        obi > 0.7,                          # 订单簿严重失衡（买盘强）
        cvd > 0 and abs(cvd) > 10000,       # CVD 极端正值
        rsi > 70,                           # RSI 超买区
        price > vwap * 1.02                 # 价格显著高于 VWAP
    ]

    # 极卖条件：多个指标同时显示极端卖出信号
    extreme_sell_conditions = [
        obi < -0.7,                         # 订单簿严重失衡（卖盘强）
        cvd < 0 and abs(cvd) > 10000,       # CVD 极端负值
        rsi < 30,                           # RSI 超卖区
        price < vwap * 0.98                 # 价格显著低于 VWAP
    ]

    buy_count = sum(extreme_buy_conditions)
    sell_count = sum(extreme_sell_conditions)

    # 判定逻辑
    if buy_count >= 3:
        return '[极买]'
    elif buy_count >= 2:
        return '[超买]'
    elif sell_count >= 3:
        return '[极卖]'
    elif sell_count >= 2:
        return '[超卖]'
    else:
        return ''
```

**极端状态标签表**：

| 标签 | 含义 | 触发条件 | 操作建议 |
|-----|------|---------|---------|
| `[极买]` | 极端买入信号 | 3+ 指标同时看多极端 | 谨慎追多，可能见顶 |
| `[极卖]` | 极端卖出信号 | 3+ 指标同时看空极端 | 谨慎追空，可能见底 |
| `[超买]` | 超买状态 | 2 指标看多极端 | 注意回调风险 |
| `[超卖]` | 超卖状态 | 2 指标看空极端 | 注意反弹机会 |
| (无标签) | 中性状态 | 不满足极端条件 | 正常按信号操作 |

**显示格式**：
```
15M: 中性 [极买]    # 15分钟级别中性，但处于极买状态
4H: 多 [超买]       # 4小时级别看多，但已超买
1D: 空              # 日线级别看空，无极端状态
```

---

### 7.4 刷新倒计时机制

**用途**：提示用户各时间框架数据何时更新

```python
from datetime import datetime, timedelta

class RefreshCountdown:
    """
    多时间框架刷新倒计时
    """

    INTERVALS = {
        '15m': 15 * 60,      # 15分钟 = 900秒
        '4h': 4 * 60 * 60,   # 4小时 = 14400秒
        '1d': 24 * 60 * 60   # 1天 = 86400秒
    }

    def __init__(self):
        self.last_refresh = {}

    def get_next_candle_close(self, timeframe: str) -> int:
        """
        计算距离下一根K线收盘的剩余秒数
        """
        now = datetime.utcnow()
        interval_seconds = self.INTERVALS.get(timeframe, 3600)

        # 计算当前周期开始时间
        if timeframe == '1d':
            # 日线以 UTC 0:00 为准
            current_period_start = now.replace(hour=0, minute=0, second=0, microsecond=0)
        elif timeframe == '4h':
            # 4小时线以 0, 4, 8, 12, 16, 20 点为准
            hour = (now.hour // 4) * 4
            current_period_start = now.replace(hour=hour, minute=0, second=0, microsecond=0)
        elif timeframe == '15m':
            # 15分钟线
            minute = (now.minute // 15) * 15
            current_period_start = now.replace(minute=minute, second=0, microsecond=0)
        else:
            current_period_start = now

        # 下一根K线收盘时间
        next_close = current_period_start + timedelta(seconds=interval_seconds)

        # 剩余秒数
        remaining_seconds = int((next_close - now).total_seconds())
        return max(0, remaining_seconds)

    def format_countdown(self) -> str:
        """
        格式化倒计时显示
        """
        countdown_4h = self.get_next_candle_close('4h')
        countdown_1d = self.get_next_candle_close('1d')

        return f"[下次刷新] 4H: {countdown_4h}s | 1D: {countdown_1d}s"
```

**倒计时显示格式**：
```
[扫描中] 15:37:19 | 1D:中性 4H:多 | 状态:静默 | [下次刷新] 4H: 96s | 1D: 855s
```

**倒计时用途说明**：

| 时间框架 | 刷新意义 | 操作建议 |
|---------|---------|---------|
| 4H 倒计时 | 4小时K线即将收盘 | 收盘前后可能有方向变化 |
| 1D 倒计时 | 日线即将收盘 | 日线收盘确认趋势更可靠 |

**临近刷新时的注意事项**：
- 倒计时 < 60s：暂缓开仓，等待新K线确认
- 刷新后方向改变：重新评估持仓
- 刷新后方向不变：趋势得到确认

---

### 7.5 入场规则

```
✅ 有效入场条件：
- 大周期 (1D) 看多 + 中周期 (4H) 回调到支撑位 + 小周期 (15M) 出现买入信号 → 做多
- 大周期 (1D) 看空 + 中周期 (4H) 反弹到阻力位 + 小周期 (15M) 出现卖出信号 → 做空

❌ 无效/降级条件：
- 大周期与小周期方向矛盾 → 放弃交易
- 仅有小周期信号，大周期中性 → 降低仓位 50%
- 中周期处于关键位置内部（非支撑非阻力）→ 等待
```

---

## 八、信号衰减与时效性

### 8.1 信号生命周期

| 信号类型 | 有效期 | 衰减方式 | 初始置信度 |
|---------|-------|---------|-----------|
| `WHALE_ALERT` | 5 分钟 | 线性衰减 | 80% |
| `ICEBERG` | 15 分钟 | 阶梯衰减 | 70% |
| `SYMMETRY_BREAK` | 30 分钟 | 指数衰减 | 85% |
| `CHAIN_SIGNAL` | 60 分钟 | 慢速线性 | 60% |

### 8.2 衰减公式

```python
import math

def calculate_signal_decay(signal_type: str, elapsed_seconds: float, initial_confidence: float) -> float:
    """
    计算信号衰减后的置信度
    """
    DECAY_CONFIG = {
        'WHALE_ALERT': {'ttl': 300, 'method': 'linear'},
        'ICEBERG': {'ttl': 900, 'method': 'step'},
        'SYMMETRY_BREAK': {'ttl': 1800, 'method': 'exponential'},
        'CHAIN_SIGNAL': {'ttl': 3600, 'method': 'linear'}
    }

    config = DECAY_CONFIG.get(signal_type, {'ttl': 600, 'method': 'linear'})
    ttl = config['ttl']
    method = config['method']

    if elapsed_seconds >= ttl:
        return 0

    if method == 'linear':
        decay_factor = 1 - (elapsed_seconds / ttl)
    elif method == 'exponential':
        decay_factor = math.exp(-3 * elapsed_seconds / ttl)
    elif method == 'step':
        # 阶梯衰减：每 1/3 生命周期降低 30%
        steps = int(elapsed_seconds / (ttl / 3))
        decay_factor = 1 - (steps * 0.3)

    return initial_confidence * max(0, decay_factor)
```

### 8.3 信号冲突处理

```python
def resolve_signal_conflict(active_signals: list) -> dict:
    """
    处理信号冲突
    """
    long_signals = [s for s in active_signals if s['direction'] == 'LONG']
    short_signals = [s for s in active_signals if s['direction'] == 'SHORT']

    long_score = sum([s['confidence'] for s in long_signals])
    short_score = sum([s['confidence'] for s in short_signals])

    net_score = long_score - short_score

    if abs(net_score) < 0.2:
        return {'action': 'HOLD', 'reason': 'SIGNAL_CONFLICT', 'net_score': net_score}
    elif net_score > 0:
        return {'action': 'LONG', 'confidence': min(net_score, 1.0), 'net_score': net_score}
    else:
        return {'action': 'SHORT', 'confidence': min(abs(net_score), 1.0), 'net_score': net_score}
```

---

## 九、执行层细节

### 9.1 滑点预估

```python
def estimate_slippage(order_size_usd: float, orderbook: dict, volatility: float) -> float:
    """
    预估滑点
    """
    # 获取盘口深度
    depth_usd = calculate_orderbook_depth(orderbook, levels=10)

    # 基础滑点 = 订单大小 / 盘口深度
    base_slippage = order_size_usd / depth_usd

    # 波动率调整
    volatility_factor = 1 + (volatility * 2)

    estimated_slippage = base_slippage * volatility_factor

    return min(estimated_slippage, 0.05)  # 上限 5%
```

### 9.2 执行策略

| 订单规模 | 执行策略 | 描述 |
|---------|---------|------|
| 小单 (<$1,000) | 市价单 | 直接成交，接受滑点 |
| 中单 ($1,000-$10,000) | 限价单 + 超时 | 限价挂单，超时 30s 转市价 |
| 大单 (>$10,000) | TWAP 拆分 | 拆分成多笔，每 30s 执行一笔 |

```python
def execute_order(order: dict) -> dict:
    """
    智能订单执行
    """
    size_usd = order['size'] * order['price']

    if size_usd < 1000:
        return execute_market_order(order)
    elif size_usd < 10000:
        return execute_limit_with_timeout(order, timeout=30)
    else:
        return execute_twap(order, num_slices=5, interval=30)
```

### 9.3 成交确认

```python
EXECUTION_CONFIG = {
    "partial_fill_threshold": 0.8,   # 部分成交阈值：80% 以上视为成功
    "order_timeout_seconds": 60,     # 订单超时时间
    "max_retries": 3,                # 最大重试次数
    "retry_delay_seconds": 5         # 重试间隔
}

def handle_partial_fill(order: dict, filled_pct: float) -> str:
    """
    处理部分成交
    """
    if filled_pct >= EXECUTION_CONFIG['partial_fill_threshold']:
        return 'ACCEPT'  # 接受部分成交
    elif filled_pct > 0.5:
        return 'CHASE'   # 追单补足
    else:
        return 'CANCEL'  # 取消剩余
```

### 9.4 手续费计算

```python
FEE_CONFIG = {
    "maker_fee": 0.0002,    # Maker 费率: 0.02%
    "taker_fee": 0.0004,    # Taker 费率: 0.04%
    "funding_interval": 8    # 资金费率结算间隔（小时）
}

def calculate_total_cost(entry_price: float, exit_price: float, size: float,
                         order_type: str, holding_hours: float, funding_rate: float) -> dict:
    """
    计算总交易成本
    """
    fee_rate = FEE_CONFIG['maker_fee'] if order_type == 'LIMIT' else FEE_CONFIG['taker_fee']

    entry_fee = entry_price * size * fee_rate
    exit_fee = exit_price * size * fee_rate

    # 资金费率（仅永续合约）
    funding_periods = int(holding_hours / FEE_CONFIG['funding_interval'])
    funding_cost = entry_price * size * funding_rate * funding_periods

    gross_pnl = (exit_price - entry_price) * size
    net_pnl = gross_pnl - entry_fee - exit_fee - funding_cost

    return {
        'gross_pnl': gross_pnl,
        'entry_fee': entry_fee,
        'exit_fee': exit_fee,
        'funding_cost': funding_cost,
        'total_cost': entry_fee + exit_fee + funding_cost,
        'net_pnl': net_pnl
    }
```

---

## 十、BTC 联动分析

### 10.1 相关性监控

```python
import numpy as np

def calculate_rolling_correlation(asset_returns: list, btc_returns: list, window: int = 30) -> float:
    """
    计算滚动相关系数
    """
    if len(asset_returns) < window or len(btc_returns) < window:
        return 0

    asset_window = asset_returns[-window:]
    btc_window = btc_returns[-window:]

    correlation = np.corrcoef(asset_window, btc_window)[0, 1]
    return round(correlation, 4)
```

### 10.2 联动规则表

| BTC 状态 | 条件 | 目标资产策略调整 |
|---------|------|-----------------|
| BTC 暴跌 | 15分钟跌幅 > 3% | 暂停做多，仅允许做空 |
| BTC 暴涨 | 15分钟涨幅 > 3% | 暂停做空，仅允许做多 |
| BTC 横盘 | 1小时波幅 < 0.5% | 正常执行所有信号 |
| 相关性脱钩 | 相关系数 < 0.3 | 目标资产独立行情，可加权 |
| 高度相关 | 相关系数 > 0.8 | 优先参考 BTC 方向 |

### 10.3 Beta 系数计算

```python
def calculate_beta(asset_returns: list, btc_returns: list) -> float:
    """
    计算 Beta 系数
    Beta > 1: 波动放大（更激进）
    Beta < 1: 波动收敛（更保守）
    Beta < 0: 负相关
    """
    covariance = np.cov(asset_returns, btc_returns)[0, 1]
    btc_variance = np.var(btc_returns)

    if btc_variance == 0:
        return 1

    beta = covariance / btc_variance
    return round(beta, 4)
```

### 10.4 BTC 状态适配

```python
def adjust_for_btc_state(signal: dict, btc_state: dict) -> dict:
    """
    根据 BTC 状态调整信号
    """
    adjusted_signal = signal.copy()

    # BTC 暴跌时
    if btc_state['change_15m'] < -0.03:
        if signal['direction'] == 'LONG':
            adjusted_signal['confidence'] *= 0.3  # 大幅降低做多置信度
            adjusted_signal['note'] = 'BTC_CRASH_WARNING'

    # BTC 暴涨时
    if btc_state['change_15m'] > 0.03:
        if signal['direction'] == 'SHORT':
            adjusted_signal['confidence'] *= 0.3  # 大幅降低做空置信度
            adjusted_signal['note'] = 'BTC_PUMP_WARNING'

    # 相关性脱钩时
    if btc_state['correlation'] < 0.3:
        adjusted_signal['confidence'] *= 1.2  # 独立行情，可适当加权
        adjusted_signal['note'] = 'DECOUPLED_SIGNAL'

    return adjusted_signal
```

---

## 十一、回测框架

### 11.1 数据分割

```python
BACKTEST_CONFIG = {
    "train_ratio": 0.7,              # 训练集比例: 70%
    "test_ratio": 0.3,               # 测试集比例: 30%
    "walk_forward_windows": 5,       # Walk-forward 窗口数
    "min_trades_for_significance": 100  # 统计显著性最低交易次数
}

def split_data(data: list) -> dict:
    """
    分割数据集
    """
    split_idx = int(len(data) * BACKTEST_CONFIG['train_ratio'])

    return {
        'train': data[:split_idx],
        'test': data[split_idx:]
    }
```

### 11.2 Walk-Forward 验证

```python
def walk_forward_validation(data: list, strategy: callable, windows: int = 5) -> list:
    """
    Walk-forward 滚动验证
    防止过拟合的核心方法
    """
    results = []
    window_size = len(data) // (windows + 1)

    for i in range(windows):
        train_start = i * window_size
        train_end = (i + 1) * window_size
        test_end = (i + 2) * window_size

        train_data = data[train_start:train_end]
        test_data = data[train_end:test_end]

        # 在训练集上优化参数
        optimized_params = optimize_strategy(train_data, strategy)

        # 在测试集上验证
        test_result = backtest_strategy(test_data, strategy, optimized_params)
        results.append(test_result)

    return results
```

### 11.3 真实成本模拟

```python
def simulate_realistic_execution(signal: dict, orderbook: dict, volatility: float) -> dict:
    """
    模拟真实执行成本
    """
    # 滑点模拟
    slippage = estimate_slippage(signal['size_usd'], orderbook, volatility)

    # 手续费
    fee = signal['size_usd'] * FEE_CONFIG['taker_fee']

    # 实际入场价
    if signal['direction'] == 'LONG':
        actual_entry = signal['price'] * (1 + slippage)
    else:
        actual_entry = signal['price'] * (1 - slippage)

    return {
        'intended_price': signal['price'],
        'actual_price': actual_entry,
        'slippage_pct': slippage,
        'fee': fee
    }
```

### 11.4 统计指标要求

| 指标 | 英文 | 合格线 | 优秀线 |
|-----|------|-------|-------|
| 夏普比率 | Sharpe Ratio | > 1.0 | > 2.0 |
| 最大回撤 | Max Drawdown | < 25% | < 15% |
| 盈亏比 | Profit Factor | > 1.3 | > 2.0 |
| 胜率 | Win Rate | > 35% | > 50% |
| 期望值 | Expected Value | > 0 | > 0.5% |
| 卡玛比率 | Calmar Ratio | > 1.0 | > 2.0 |
| 交易次数 | Trade Count | > 100 | > 500 |

```python
def calculate_backtest_metrics(trades: list) -> dict:
    """
    计算回测统计指标
    """
    if len(trades) == 0:
        return None

    returns = [t['pnl_pct'] for t in trades]
    wins = [r for r in returns if r > 0]
    losses = [r for r in returns if r < 0]

    metrics = {
        'total_trades': len(trades),
        'win_rate': len(wins) / len(trades) if trades else 0,
        'avg_win': np.mean(wins) if wins else 0,
        'avg_loss': np.mean(losses) if losses else 0,
        'profit_factor': abs(sum(wins) / sum(losses)) if losses else float('inf'),
        'expected_value': np.mean(returns),
        'sharpe_ratio': np.mean(returns) / np.std(returns) * np.sqrt(252) if np.std(returns) > 0 else 0,
        'max_drawdown': calculate_max_drawdown(returns),
        'calmar_ratio': np.mean(returns) * 252 / abs(calculate_max_drawdown(returns)) if calculate_max_drawdown(returns) != 0 else 0
    }

    return metrics
```

### 11.5 参数敏感性测试

```python
def parameter_sensitivity_test(data: list, strategy: callable,
                                base_params: dict, param_range: float = 0.2) -> dict:
    """
    参数敏感性测试
    检验策略对参数变化的稳健性
    """
    results = {}

    for param_name, base_value in base_params.items():
        param_results = []

        # 测试 ±20% 范围
        for multiplier in [0.8, 0.9, 1.0, 1.1, 1.2]:
            test_params = base_params.copy()
            test_params[param_name] = base_value * multiplier

            result = backtest_strategy(data, strategy, test_params)
            param_results.append({
                'multiplier': multiplier,
                'sharpe': result['sharpe_ratio'],
                'profit_factor': result['profit_factor']
            })

        results[param_name] = param_results

    return results
```

---

## 十二、异常处理

### 12.1 数据异常处理

```python
EXCEPTION_CONFIG = {
    "max_reconnect_attempts": 5,     # 最大重连次数
    "reconnect_delay_seconds": 5,    # 重连间隔
    "price_spike_threshold": 0.05,   # 价格跳变阈值: 5%
    "data_staleness_seconds": 30     # 数据过期阈值
}

class DataExceptionHandler:
    def __init__(self):
        self.last_valid_price = None
        self.last_update_time = None
        self.reconnect_attempts = 0

    def validate_price(self, new_price: float) -> dict:
        """
        验证价格数据有效性
        """
        if self.last_valid_price is None:
            self.last_valid_price = new_price
            return {'valid': True, 'price': new_price}

        change_pct = abs(new_price - self.last_valid_price) / self.last_valid_price

        if change_pct > EXCEPTION_CONFIG['price_spike_threshold']:
            return {
                'valid': False,
                'reason': 'PRICE_SPIKE',
                'change_pct': change_pct,
                'suggested_action': 'WAIT_FOR_CONFIRMATION'
            }

        self.last_valid_price = new_price
        return {'valid': True, 'price': new_price}

    def handle_disconnect(self) -> dict:
        """
        处理连接断开
        """
        self.reconnect_attempts += 1

        if self.reconnect_attempts > EXCEPTION_CONFIG['max_reconnect_attempts']:
            return {
                'action': 'HALT_TRADING',
                'reason': 'MAX_RECONNECT_EXCEEDED'
            }

        return {
            'action': 'RECONNECT',
            'delay': EXCEPTION_CONFIG['reconnect_delay_seconds'] * self.reconnect_attempts
        }
```

### 12.2 系统异常处理

```python
import logging
from enum import Enum

class LogLevel(Enum):
    DEBUG = 10
    INFO = 20
    WARNING = 30
    ERROR = 40
    CRITICAL = 50

class SystemMonitor:
    def __init__(self):
        self.signal_queue = []
        self.max_queue_size = 1000
        self.error_count = 0
        self.max_errors_before_halt = 10

    def add_signal(self, signal: dict) -> bool:
        """
        添加信号到队列，处理队列溢出
        """
        if len(self.signal_queue) >= self.max_queue_size:
            # 丢弃最旧的信号
            self.signal_queue = self.signal_queue[100:]
            logging.warning("Signal queue overflow, dropped oldest 100 signals")

        self.signal_queue.append(signal)
        return True

    def log_error(self, error: Exception, context: str) -> dict:
        """
        记录错误并决定是否继续
        """
        self.error_count += 1

        logging.error(f"[{context}] {type(error).__name__}: {str(error)}")

        if self.error_count >= self.max_errors_before_halt:
            return {
                'action': 'HALT',
                'reason': 'TOO_MANY_ERRORS',
                'error_count': self.error_count
            }

        return {
            'action': 'CONTINUE',
            'error_count': self.error_count
        }
```

### 12.3 日志配置

```python
LOGGING_CONFIG = {
    'version': 1,
    'handlers': {
        'console': {
            'class': 'logging.StreamHandler',
            'level': 'INFO',
            'formatter': 'standard'
        },
        'file': {
            'class': 'logging.handlers.RotatingFileHandler',
            'filename': 'storage/logs/system.log',
            'maxBytes': 10485760,  # 10MB
            'backupCount': 5,
            'level': 'DEBUG'
        },
        'error_file': {
            'class': 'logging.FileHandler',
            'filename': 'storage/logs/error.log',
            'level': 'ERROR'
        }
    },
    'formatters': {
        'standard': {
            'format': '[%(asctime)s] %(levelname)s [%(name)s:%(lineno)d] %(message)s'
        }
    },
    'root': {
        'level': 'DEBUG',
        'handlers': ['console', 'file', 'error_file']
    }
}
```

---

## 十三、账户风险监控（实时风险报告）

### 13.1 账户健康度指标

```python
ACCOUNT_RISK_CONFIG = {
    "margin_level_safe": 500,        # 极度安全阈值
    "margin_level_normal": 200,      # 正常阈值
    "margin_level_warning": 150,     # 预警阈值
    "margin_level_danger": 120,      # 危险阈值
    "margin_level_liquidation": 100, # 强平线
    "max_leverage": 5,               # 最大允许杠杆
    "max_debt_ratio": 0.5            # 最大负债比率
}
```

### 13.2 账户状态分级

| 风险率 (Margin Level) | 状态 | 颜色 | 操作建议 |
|----------------------|------|------|---------|
| > 500 | 极度安全 | 绿色 | 可正常开仓 |
| 200 - 500 | 安全 | 绿色 | 可正常开仓 |
| 150 - 200 | 注意 | 黄色 | 减少新开仓 |
| 120 - 150 | 预警 | 橙色 | 禁止开仓，考虑减仓 |
| 100 - 120 | 危险 | 红色 | 立即减仓 |
| < 100 | 爆仓风险 | 红色闪烁 | 紧急平仓 |

### 13.3 实时风险报告数据结构

```python
class AccountRiskReport:
    """
    实时账户风险报告
    """
    def __init__(self):
        self.timestamp = None
        self.account_status = None      # 极度安全/安全/注意/预警/危险
        self.margin_level = 0.0         # 风险率 (保证金率)
        self.total_equity = 0.0         # 总权益
        self.used_margin = 0.0          # 已用保证金
        self.available_margin = 0.0     # 可用保证金
        self.usdt_debt = 0.0            # USDT 负债
        self.unrealized_pnl = 0.0       # 未实现盈亏
        self.position_value = 0.0       # 持仓价值
        self.leverage_used = 0.0        # 实际使用杠杆

    def to_display(self) -> str:
        """
        生成终端显示格式
        """
        return f"""
[{self.timestamp}] --- 实时风险报告 ---
账户状态: ◉ {self.account_status}
风险率 (Margin Level): {self.margin_level:.2f}
USDT 负债: {self.usdt_debt:.2f} USDT
"""
```

### 13.4 风险率计算

```python
def calculate_margin_level(account: dict) -> float:
    """
    计算风险率 (保证金水平)

    公式: Margin Level = (总权益 / 已用保证金) * 100

    - 无持仓时返回 999 (或无穷大)
    - 数值越高越安全
    - 低于 100 面临强平
    """
    total_equity = account['balance'] + account['unrealized_pnl']
    used_margin = account['used_margin']

    if used_margin == 0:
        return 999.0  # 无持仓，极度安全

    margin_level = (total_equity / used_margin) * 100
    return round(margin_level, 2)


def calculate_leverage_used(account: dict) -> float:
    """
    计算实际使用杠杆

    公式: 实际杠杆 = 持仓价值 / 总权益
    """
    total_equity = account['balance'] + account['unrealized_pnl']
    position_value = account['position_value']

    if total_equity == 0:
        return 0

    leverage = position_value / total_equity
    return round(leverage, 2)
```

### 13.5 账户状态判定

```python
def evaluate_account_status(margin_level: float, debt: float, leverage: float) -> dict:
    """
    综合评估账户状态
    """
    # 基于风险率判定
    if margin_level >= ACCOUNT_RISK_CONFIG['margin_level_safe']:
        status = '极度安全'
        color = 'green'
        can_open = True
    elif margin_level >= ACCOUNT_RISK_CONFIG['margin_level_normal']:
        status = '安全'
        color = 'green'
        can_open = True
    elif margin_level >= ACCOUNT_RISK_CONFIG['margin_level_warning']:
        status = '注意'
        color = 'yellow'
        can_open = True  # 可开仓但减少仓位
    elif margin_level >= ACCOUNT_RISK_CONFIG['margin_level_danger']:
        status = '预警'
        color = 'orange'
        can_open = False
    elif margin_level >= ACCOUNT_RISK_CONFIG['margin_level_liquidation']:
        status = '危险'
        color = 'red'
        can_open = False
    else:
        status = '爆仓风险'
        color = 'red_blink'
        can_open = False

    # 杠杆检查
    if leverage > ACCOUNT_RISK_CONFIG['max_leverage']:
        status = '杠杆过高'
        can_open = False

    # 负债检查
    debt_ratio = debt / (debt + account['balance']) if debt > 0 else 0
    if debt_ratio > ACCOUNT_RISK_CONFIG['max_debt_ratio']:
        status = '负债过高'
        can_open = False

    return {
        'status': status,
        'color': color,
        'can_open_position': can_open,
        'margin_level': margin_level,
        'leverage': leverage,
        'debt_ratio': debt_ratio
    }
```

### 13.6 爆仓预警系统

```python
class LiquidationWarning:
    """
    爆仓预警系统
    """

    def __init__(self):
        self.warning_triggered = False
        self.last_warning_time = None
        self.warning_cooldown = 60  # 预警冷却时间（秒）

    def check_liquidation_risk(self, account: dict, current_price: float) -> dict:
        """
        检查爆仓风险并计算强平价格
        """
        position = account.get('position', {})

        if not position or position.get('size', 0) == 0:
            return {'at_risk': False, 'liquidation_price': None}

        # 计算强平价格
        entry_price = position['entry_price']
        size = position['size']
        leverage = position['leverage']
        direction = position['direction']  # 'LONG' or 'SHORT'
        maintenance_margin_rate = 0.005  # 维持保证金率 0.5%

        if direction == 'LONG':
            # 多仓强平价 = 入场价 * (1 - 1/杠杆 + 维持保证金率)
            liquidation_price = entry_price * (1 - 1/leverage + maintenance_margin_rate)
        else:
            # 空仓强平价 = 入场价 * (1 + 1/杠杆 - 维持保证金率)
            liquidation_price = entry_price * (1 + 1/leverage - maintenance_margin_rate)

        # 计算距离强平的百分比
        if direction == 'LONG':
            distance_to_liq = (current_price - liquidation_price) / current_price
        else:
            distance_to_liq = (liquidation_price - current_price) / current_price

        # 风险等级
        if distance_to_liq < 0.02:  # 距离强平 < 2%
            risk_level = 'CRITICAL'
        elif distance_to_liq < 0.05:  # 距离强平 < 5%
            risk_level = 'HIGH'
        elif distance_to_liq < 0.10:  # 距离强平 < 10%
            risk_level = 'MEDIUM'
        else:
            risk_level = 'LOW'

        return {
            'at_risk': distance_to_liq < 0.10,
            'liquidation_price': round(liquidation_price, 6),
            'current_price': current_price,
            'distance_to_liq_pct': round(distance_to_liq * 100, 2),
            'risk_level': risk_level,
            'direction': direction
        }

    def generate_warning_message(self, liq_info: dict) -> str:
        """
        生成预警消息
        """
        if liq_info['risk_level'] == 'CRITICAL':
            return f"""
╔══════════════════════════════════════════════════════════╗
║  ⚠️  紧急爆仓预警  ⚠️                                      ║
╠══════════════════════════════════════════════════════════╣
║  当前价格: {liq_info['current_price']}
║  强平价格: {liq_info['liquidation_price']}
║  距离强平: {liq_info['distance_to_liq_pct']}%
║  持仓方向: {liq_info['direction']}
║                                                          ║
║  >>> 建议立即减仓或追加保证金 <<<                          ║
╚══════════════════════════════════════════════════════════╝
"""
        elif liq_info['risk_level'] == 'HIGH':
            return f"[!WARNING] 爆仓风险较高 | 强平价: {liq_info['liquidation_price']} | 距离: {liq_info['distance_to_liq_pct']}%"
        else:
            return None
```

### 13.7 实时风险报告输出格式

```
┌────────────────────────────────────────────────────────────┐
│ [14:58:07] --- 实时风险报告 ---                            │
├────────────────────────────────────────────────────────────┤
│ 账户状态: ◉ 极度安全                                        │
│ 风险率 (Margin Level): 999.00                              │
│ USDT 负债: 0.00 USDT                                       │
├────────────────────────────────────────────────────────────┤
│ 总权益: 10,000.00 USDT                                     │
│ 已用保证金: 0.00 USDT                                      │
│ 可用保证金: 10,000.00 USDT                                 │
│ 未实现盈亏: 0.00 USDT                                      │
│ 实际杠杆: 0.00x                                            │
└────────────────────────────────────────────────────────────┘
```

### 13.8 与策略系统集成

```python
def should_allow_new_position(account_risk: dict, signal: dict) -> dict:
    """
    根据账户风险状态决定是否允许新开仓
    """
    # 账户状态检查
    if not account_risk['can_open_position']:
        return {
            'allowed': False,
            'reason': f"账户状态: {account_risk['status']}",
            'suggestion': '请先降低风险敞口'
        }

    # 检查新仓位是否会导致风险过高
    projected_leverage = account_risk['leverage'] + signal['position_size'] / account_risk['equity']

    if projected_leverage > ACCOUNT_RISK_CONFIG['max_leverage']:
        return {
            'allowed': False,
            'reason': '开仓后杠杆将超过限制',
            'current_leverage': account_risk['leverage'],
            'projected_leverage': projected_leverage,
            'max_allowed': ACCOUNT_RISK_CONFIG['max_leverage']
        }

    # 检查风险率是否会降至预警线以下
    projected_margin_level = estimate_margin_level_after_trade(account_risk, signal)

    if projected_margin_level < ACCOUNT_RISK_CONFIG['margin_level_warning']:
        return {
            'allowed': False,
            'reason': '开仓后风险率将过低',
            'projected_margin_level': projected_margin_level
        }

    return {
        'allowed': True,
        'projected_leverage': projected_leverage,
        'projected_margin_level': projected_margin_level
    }
```

---

## 十四、执行环境检查

### 14.1 借币可用性检查（做空前置条件）

```python
MARGIN_CONFIG = {
    "min_borrowable_amount": 100,     # 最小可借数量
    "borrow_rate_warning": 0.001,     # 借币利率预警线（0.1%/天）
    "auto_repay": True,               # 是否自动还币
    "check_interval": 30              # 检查间隔（秒）
}

class BorrowAvailabilityChecker:
    """
    借币可用性检查器
    做空前必须确认有币可借
    """

    def __init__(self, exchange_client):
        self.client = exchange_client
        self.cache = {}
        self.cache_ttl = 30  # 缓存有效期（秒）

    async def check_borrow_availability(self, symbol: str, amount: float) -> dict:
        """
        检查借币可用性

        返回:
        {
            'available': True/False,
            'borrowable_amount': 10000.0,
            'hourly_rate': 0.0001,
            'daily_rate': 0.0024,
            'status': 'OK' / 'LIMITED' / 'UNAVAILABLE'
        }
        """
        try:
            # 查询可借数量
            margin_info = await self.client.get_margin_info(symbol)

            borrowable = margin_info.get('borrowable', 0)
            hourly_rate = margin_info.get('hourlyInterestRate', 0)
            daily_rate = hourly_rate * 24

            # 判断状态
            if borrowable >= amount:
                status = 'OK'
                available = True
            elif borrowable > 0:
                status = 'LIMITED'
                available = borrowable >= MARGIN_CONFIG['min_borrowable_amount']
            else:
                status = 'UNAVAILABLE'
                available = False

            # 利率预警
            rate_warning = daily_rate > MARGIN_CONFIG['borrow_rate_warning']

            return {
                'available': available,
                'borrowable_amount': borrowable,
                'requested_amount': amount,
                'hourly_rate': hourly_rate,
                'daily_rate': daily_rate,
                'status': status,
                'rate_warning': rate_warning
            }

        except Exception as e:
            return {
                'available': False,
                'status': 'ERROR',
                'error': str(e)
            }

    def format_status(self, check_result: dict) -> str:
        """
        格式化借币状态显示
        """
        if check_result['status'] == 'OK':
            return f"借币可用性: ◉ OK | 可借: {check_result['borrowable_amount']:.2f}"
        elif check_result['status'] == 'LIMITED':
            return f"借币可用性: ⚠ 受限 | 可借: {check_result['borrowable_amount']:.2f}"
        elif check_result['status'] == 'UNAVAILABLE':
            return f"借币可用性: ✗ 不可用 | 无币可借"
        else:
            return f"借币可用性: ✗ 错误 | {check_result.get('error', '未知')}"
```

### 14.2 做空前置检查流程

```python
async def pre_short_check(symbol: str, size: float, account: dict) -> dict:
    """
    做空前完整检查流程
    """
    checks = {
        'borrow_available': False,
        'margin_sufficient': False,
        'rate_acceptable': False,
        'account_healthy': False,
        'all_passed': False
    }

    # 1. 检查借币可用性
    borrow_check = await borrow_checker.check_borrow_availability(symbol, size)
    checks['borrow_available'] = borrow_check['available']
    checks['borrow_details'] = borrow_check

    # 2. 检查保证金是否充足
    required_margin = calculate_required_margin(size, account['leverage'])
    checks['margin_sufficient'] = account['available_margin'] >= required_margin
    checks['required_margin'] = required_margin

    # 3. 检查借币利率是否可接受
    if borrow_check.get('daily_rate', 0) < MARGIN_CONFIG['borrow_rate_warning']:
        checks['rate_acceptable'] = True
    checks['daily_rate'] = borrow_check.get('daily_rate', 0)

    # 4. 检查账户健康度
    checks['account_healthy'] = account['margin_level'] > ACCOUNT_RISK_CONFIG['margin_level_warning']

    # 综合判定
    checks['all_passed'] = all([
        checks['borrow_available'],
        checks['margin_sufficient'],
        checks['rate_acceptable'],
        checks['account_healthy']
    ])

    return checks
```

### 14.3 数据源对齐状态

```python
class DataSourceAligner:
    """
    数据源对齐检查器
    确保 Market(M) + Iceberg(I) + Chain(A) 数据同步
    """

    def __init__(self):
        self.source_status = {
            'M': {'connected': False, 'last_update': None, 'latency_ms': 0},
            'I': {'connected': False, 'last_update': None, 'latency_ms': 0},
            'A': {'connected': False, 'last_update': None, 'latency_ms': 0}
        }
        self.max_allowed_lag = 5000  # 最大允许延迟（毫秒）
        self.max_time_diff = 10      # 数据源间最大时间差（秒）

    def update_source_status(self, source: str, timestamp: float, latency: float):
        """
        更新数据源状态
        """
        self.source_status[source] = {
            'connected': True,
            'last_update': timestamp,
            'latency_ms': latency
        }

    def check_alignment(self) -> dict:
        """
        检查数据源对齐状态

        返回:
        {
            'aligned': True/False,
            'status': '实时对齐' / '部分延迟' / '严重不同步',
            'details': {...}
        }
        """
        current_time = time.time()

        # 检查各数据源状态
        source_health = {}
        timestamps = []

        for source, status in self.source_status.items():
            if not status['connected']:
                source_health[source] = 'DISCONNECTED'
                continue

            age = current_time - status['last_update']

            if age < 5:
                source_health[source] = 'REALTIME'
                timestamps.append(status['last_update'])
            elif age < 30:
                source_health[source] = 'DELAYED'
                timestamps.append(status['last_update'])
            else:
                source_health[source] = 'STALE'

        # 计算数据源间的时间差
        if len(timestamps) >= 2:
            time_diff = max(timestamps) - min(timestamps)
        else:
            time_diff = float('inf')

        # 判定对齐状态
        all_realtime = all(h == 'REALTIME' for h in source_health.values())
        any_disconnected = any(h == 'DISCONNECTED' for h in source_health.values())
        any_stale = any(h == 'STALE' for h in source_health.values())

        if all_realtime and time_diff < self.max_time_diff:
            aligned = True
            status = '实时对齐'
        elif any_disconnected or any_stale:
            aligned = False
            status = '严重不同步'
        else:
            aligned = False
            status = '部分延迟'

        return {
            'aligned': aligned,
            'status': status,
            'source_health': source_health,
            'time_diff_seconds': time_diff,
            'display': self.format_alignment_status(source_health, status)
        }

    def format_alignment_status(self, health: dict, status: str) -> str:
        """
        格式化对齐状态显示
        """
        icons = {
            'REALTIME': '◉',
            'DELAYED': '○',
            'STALE': '✗',
            'DISCONNECTED': '✗'
        }

        source_display = ' + '.join([
            f"{icons.get(health.get(s, 'DISCONNECTED'), '?')}{s}"
            for s in ['M', 'I', 'A']
        ])

        if status == '实时对齐':
            return f"[数据源] {source_display} {status}"
        else:
            return f"[数据源] ⚠ {source_display} {status}"
```

### 14.4 执行环境综合报告

```python
class ExecutionEnvironment:
    """
    执行环境综合检查
    """

    def __init__(self):
        self.borrow_checker = BorrowAvailabilityChecker()
        self.data_aligner = DataSourceAligner()
        self.slippage_estimator = SlippageEstimator()

    async def get_environment_report(self, symbol: str, direction: str,
                                      size: float, orderbook: dict) -> dict:
        """
        获取完整的执行环境报告
        """
        report = {
            'timestamp': time.time(),
            'symbol': symbol,
            'direction': direction
        }

        # 1. 借币可用性（仅做空时检查）
        if direction == 'SHORT':
            borrow_status = await self.borrow_checker.check_borrow_availability(symbol, size)
            report['borrow'] = {
                'status': borrow_status['status'],
                'display': 'OK' if borrow_status['available'] else 'UNAVAILABLE'
            }
        else:
            report['borrow'] = {'status': 'N/A', 'display': 'N/A (做多)'}

        # 2. 预估滑点
        slippage = self.slippage_estimator.estimate(size, orderbook)
        report['slippage'] = {
            'estimated_pct': slippage,
            'display': f"{slippage * 100:.2f}%"
        }

        # 3. 数据源对齐
        alignment = self.data_aligner.check_alignment()
        report['data_alignment'] = {
            'aligned': alignment['aligned'],
            'status': alignment['status'],
            'display': alignment['display']
        }

        # 4. 综合可执行性判断
        can_execute = True
        blockers = []

        if direction == 'SHORT' and report['borrow']['status'] != 'OK':
            can_execute = False
            blockers.append('借币不可用')

        if slippage > 0.01:  # 滑点超过 1%
            can_execute = False
            blockers.append(f'滑点过高 ({report["slippage"]["display"]})')

        if not alignment['aligned']:
            can_execute = False
            blockers.append('数据源不同步')

        report['can_execute'] = can_execute
        report['blockers'] = blockers

        return report

    def format_report(self, report: dict) -> str:
        """
        格式化执行环境报告
        """
        lines = [
            f"[执行环境] 借币可用性: {report['borrow']['display']} | "
            f"预估滑点: {report['slippage']['display']}",
            report['data_alignment']['display']
        ]

        if not report['can_execute']:
            lines.append(f"[!BLOCKED] 原因: {', '.join(report['blockers'])}")

        return '\n'.join(lines)
```

### 14.5 战略地图完整输出格式

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ [战略地图] 1D: 中性 | 4H: 多 | 15M: 多                                        │
│   ◉ 【诱多陷阱】检测到高位派发, 请勿追涨!                                      │
│ [执行环境] 借币可用性: OK | 预估滑点: 0.22%                                    │
│ [数据源] ◉M + ◉I + ◉A 实时对齐                                                │
└──────────────────────────────────────────────────────────────────────────────┘

[扫描中] 11:38:24 | 1D:中性 4H:多 | 状态:静默 | [下次刷新] 4H: 499s | 1D: 1372s
```

### 14.6 执行环境状态枚举

| 检查项 | 状态值 | 含义 |
|-------|-------|------|
| 借币可用性 | `OK` | 可正常做空 |
| | `LIMITED` | 可借数量受限 |
| | `UNAVAILABLE` | 无币可借，禁止做空 |
| | `N/A` | 做多时不需要检查 |
| 预估滑点 | `< 0.1%` | 优秀 |
| | `0.1% - 0.5%` | 正常 |
| | `0.5% - 1%` | 注意 |
| | `> 1%` | 建议拆单或放弃 |
| 数据源对齐 | `实时对齐` | 三源同步，可信赖 |
| | `部分延迟` | 有延迟，降低置信度 |
| | `严重不同步` | 禁止交易 |

---

## 十五、数据源配置

### 15.1 交易所 API

```python
EXCHANGE_CONFIG = {
    "binance": {
        "ws_base": "wss://stream.binance.com:9443/ws",
        "rest_base": "https://api.binance.com",
        "streams": [
            "{symbol}@trade",           # 实时成交
            "{symbol}@depth20@100ms",   # 订单簿深度
            "{symbol}@kline_{interval}" # K线数据
        ],
        "rate_limits": {
            "requests_per_minute": 1200,
            "orders_per_second": 10
        }
    }
}
```

### 15.2 链上数据 API

```python
CHAIN_DATA_CONFIG = {
    "etherscan": {
        "api_base": "https://api.etherscan.io/api",
        "api_key": "YOUR_API_KEY"
    },
    "whale_alert": {
        "api_base": "https://api.whale-alert.io/v1",
        "min_value_usd": 100000
    }
}
```

### 15.3 本地存储

```python
STORAGE_CONFIG = {
    "database": "sqlite:///storage/trading.db",
    "signal_log": "storage/signals/",
    "backtest_results": "storage/backtest/",
    "data_retention_days": 90
}
```

---

## 十六、终端输出格式

### 16.1 实时监控输出

**基础监控格式**：
```
================================================================================
◉ Shell Market Watcher (System M) 启动
================================================================================
□□ 刷新频率: 5 秒
◉ 日志路径: storage/logs/market.log

[01:15:11] ◉ 监控中 | 价: 0.0450 | RSI: 61.3 | 量: 288
[01:15:19] ◉ 监控中 | 价: 0.0450 | RSI: 61.3 | 量: 288
[01:16:37] ◉ 监控中 | 价: 0.0450 | RSI: 61.3 | 量: 22,161
```

**完整指标监控格式**：
```
[15:34:26] ◉ 监控中 | 价: 0.0475 | 分数: 0 | OBI: 0.08 | 斜率: 0.04 | BTC: □-0.00% | 鲸鱼: 8.89% | VR: 2.5 | CVD: □- | VWAP: 0.0465 | 量: 212,040 | [链上: 中性]
```

**字段说明**：

| 字段 | 含义 | 示例 |
|-----|------|------|
| 价 | 当前价格 | 0.0475 |
| 分数 | 综合评分 (0-100) | 20 |
| OBI | 订单簿失衡 (-1 ~ 1) | 0.08 |
| 斜率 | 价格趋势斜率 | 0.04 |
| BTC | BTC 联动变化 | □-0.01% |
| 鲸鱼 | 鲸鱼成交占比 | 8.89% |
| VR | 量比 | 2.5 |
| CVD | 累积量差方向 | □- (负) / □+ (正) |
| VWAP | 成交量加权均价 | 0.0465 |
| 量 | 成交量 | 212,040 |
| [链上] | 链上状态标签 | 中性/流入/流出 |

**巨鲸信号格式**：
```
[2025-12-27 15:33:14] ◉ 【巨鲸吸筹】大额买入 | SHELL/USDT | 价: 0.0471 | 额: $9,267
[2025-12-27 15:32:44] ◉ 【巨鲸出货】大额卖出 | SHELL/USDT | 价: 0.047 | 额: $7,227
```

### 16.2 冰山单检测输出

```
[扫描中] 价格: 0.044500 | 最近成交: 0 笔 | 状态: 未发现水下大单
[扫描中] 价格: 0.044400 | 最近成交: 11 笔 | 状态: 未发现水下大单
[ICEBERG] ◉发现隐藏卖单 | 价格: 0.044400 | 累计成交: 4694.94U | 挂单深度: 647.76U | 强度: 7.25x
[ICEBERG] ◉发现隐藏卖单 | 价格: 0.044400 | 累计成交: 4694.94U | 挂单深度: 895.74U | 强度: 5.24x
[ICEBERG] ◉发现隐藏买单 | 价格: 0.044300 | 累计成交: 4071.39U | 挂单深度: 193.13U | 强度: 21.08x
```

### 16.3 战情指挥中心输出

```
[SYSTEM C] ◉ 战情指挥中心已上线 | 模式: 全域三维共振判定
[INFO] 正在监听: Market(M) + Iceberg(I) + Chain(A)

[扫描中] 15:57:40 | 状态: 全系统静默, 等待共振信号...
[扫描中] 15:58:10 | 状态: 检测到 Iceberg 信号, 等待确认...
[!ALERT] 15:58:45 | 双重共振触发 | M+I | 方向: SHORT | 置信度: 65%
[!SIGNAL] 15:59:00 | 三重共振确认 | M+I+A | 方向: SHORT | 置信度: 87% | 建议操作: 开空
```

### 16.4 战略地图输出

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ [战略地图] 1D: 中性 | 4H: 多 | 15M: 多                                        │
│   ◉ 【诱多陷阱】检测到高位派发, 请勿追涨!                                      │
│ [执行环境] 借币可用性: OK | 预估滑点: 0.22%                                    │
│ [数据源] A+M+I 实时对齐                                                        │
└──────────────────────────────────────────────────────────────────────────────┘

[扫描中] 11:38:24 | 1D:中性 4H:多 | 状态:静默 | [下次刷新] 4H: 499s | 1D: 1372s
```

---

## 十七、技术栈

### 17.1 核心依赖

```python
# requirements.txt
python>=3.10
asyncio
aiohttp>=3.8.0
websockets>=10.0
numpy>=1.21.0
pandas>=1.3.0
sqlalchemy>=1.4.0
rich>=12.0.0          # 终端美化
colorama>=0.4.4       # 跨平台颜色支持
python-dotenv>=0.19.0 # 环境变量管理
ccxt>=2.0.0           # 交易所统一接口
ta-lib>=0.4.24        # 技术指标库
```

### 17.2 项目结构

```
shell-market-watcher/
├── main.py                    # 主入口 (System M)
├── iceberg_detector.py        # 冰山单检测 (System I)
├── chain_analyzer.py          # 链上分析 (System A)
├── command_center.py          # 战情指挥 (System C)
├── core/
│   ├── __init__.py
│   ├── indicators.py          # 核心指标计算
│   ├── analyzer.py            # 信号分析器
│   ├── risk_manager.py        # 风险管理
│   └── executor.py            # 订单执行
├── scripts/
│   ├── backtest_reviewer.py   # 回测复盘 (System R)
│   └── parameter_optimizer.py # 参数优化
├── config/
│   ├── settings.py            # 全局配置
│   └── secrets.py             # API 密钥（不提交）
├── storage/
│   ├── logs/                  # 日志文件
│   ├── signals/               # 信号记录
│   └── backtest/              # 回测结果
└── tests/
    ├── test_indicators.py
    ├── test_signals.py
    └── test_risk_manager.py
```

---

## 十八、使用示例

### 18.1 启动市场监控

```bash
python main.py --symbol SHELL/USDT --interval 5
```

### 18.2 启动冰山检测

```bash
python iceberg_detector.py --symbol SHELL/USDT --threshold 5.0
```

### 18.3 启动战情指挥

```bash
python command_center.py --mode full_resonance
```

### 18.4 运行回测

```bash
python scripts/backtest_reviewer.py --start 2025-01-01 --end 2025-12-26
```

---

## 十九、核心术语表

| 中文术语 | 英文术语 | 定义 |
|---------|---------|------|
| 订单流毒性 | Order Flow Toxicity | 衡量知情交易者在订单流中占比的指标 |
| 流动性失衡 | Liquidity Imbalance | 买卖盘深度的不对称程度 |
| 冰山单 | Iceberg Order | 仅显示部分数量的大额隐藏订单 |
| 订单簿失衡 | Order Book Imbalance (OBI) | 买卖挂单量的差异比率 |
| 累积量差 | Cumulative Volume Delta (CVD) | 主动买入与主动卖出的累积差值 |
| 成交量加权均价 | VWAP | 按成交量加权的平均价格 |
| 微观结构 | Market Microstructure | 从订单层面分析市场的方法论 |
| 事件驱动 | Event-Driven | 基于特定事件触发交易决策的系统 |
| 三维共振 | Multi-dimensional Resonance | 多数据源信号同时确认的状态 |
| 诱多陷阱 | Bull Trap | 假突破后快速反转的做多陷阱 |
| 流动性猎杀 | Liquidity Hunt/Grab | 主力扫除止损单后反向运动 |

---

*文档版本: 1.0*
*最后更新: 2025-12-26*
