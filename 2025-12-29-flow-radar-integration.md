# Flow Radar 模块集成日志

**日期**: 2025-12-29
**项目**: Flow Radar (流动性雷达)
**任务**: 按照 GPT/Gemini/Claude 三方审查的执行计划，完成模块集成

---

## 背景

Flow Radar 是一个加密货币交易监控系统，用于检测市场操纵模式（洗盘、拉高出货等）。

之前已完成的模块：
- `core/state_machine.py` - 滞回状态机
- `core/event_logger.py` - 事件记录
- `core/dynamic_threshold.py` - 动态阈值
- `core/trade_deduplicator.py` - 成交去重
- `core/state_saver.py` - 状态持久化
- `core/divergence_detector.py` - 背离检测

本次任务是将这些模块按照执行计划 Step A-G 集成到 `alert_monitor.py` 主循环。

---

## 执行计划 (来自三方审查)

### Step A: 时间源统一
所有窗口/冷却/滞回都用 `event_ts`，不用 `time.time()`

### Step B: 去重放在最前面
成交去重作为数据卫生闸门，放在处理流程最前面

### Step C: 状态持久化
启动时恢复 CVD 等状态，定期保存，关闭时强制保存

### Step D: 动态阈值引擎
使用 min(P99, P95*3, median*50) 抗极端值

### Step E: 冰山分层输出
区分 ACTIVITY vs CONFIRMED 信号级别

### Step F: 背离检测
价格/CVD 背离时调整置信度

### Step G: 状态机决定告警
基于滞回状态机触发告警

---

## 实施变更

### 1. 新增导入
```python
from core.trade_deduplicator import TradeDeduplicator
from core.state_saver import StateSaver
from core.divergence_detector import DivergenceDetector, DivergenceType
```

### 2. 新增实例变量
```python
# Step B: 成交去重
self.deduplicator = TradeDeduplicator(max_size=10000, ttl_seconds=300)

# Step C: 状态持久化
self.state_saver = StateSaver(symbol=self.symbol)
self._restore_state()  # 启动时恢复状态

# Step F: 背离检测
self.divergence_detector = DivergenceDetector(window=20)
self.last_divergence = None

# CVD 累计
self.cvd_total = 0.0

# Step E: 确认冰山统计
self.confirmed_buy_count = 0
self.confirmed_sell_count = 0
self.confirmed_buy_volume = 0.0
self.confirmed_sell_volume = 0.0
```

### 3. 新增方法

#### `_restore_state()` - 启动恢复
```python
def _restore_state(self):
    saved = self.state_saver.load()
    if saved and not self.state_saver.is_stale(max_age_hours=24):
        self.cvd_total = saved.cvd_total
        self.total_whale_flow = saved.total_whale_flow
        # ... 恢复其他状态
```

#### `_save_state(event_ts)` - 定期保存
```python
def _save_state(self, event_ts: float):
    state = {
        'cvd_total': self.cvd_total,
        'total_whale_flow': self.total_whale_flow,
        # ...
    }
    self.state_saver.save(state, event_ts)
```

### 4. 修改 `analyze_and_alert()` 方法

#### Step A: 时间源统一
```python
def analyze_and_alert(self, data: Dict, event_ts: float = None):
    if event_ts is None:
        event_ts = time.time()
```

#### Step B: 成交去重
```python
raw_trades = data['trades']
trades = self.deduplicator.filter_trades(raw_trades, event_ts)
```

#### Step F: 背离检测
```python
self.last_divergence = self.divergence_detector.update(
    price=self.current_price,
    cvd=self.cvd_total,
    high=high_price,
    low=low_price,
    timestamp=event_ts
)

if self.last_divergence and self.last_divergence.detected:
    if self.last_divergence.type == DivergenceType.BEARISH:
        divergence_adjustment = -int(self.last_divergence.confidence * 30)
```

#### Step G: 状态机传入 event_ts
```python
self.current_signal = self.state_machine.update(
    score=score,
    iceberg_ratio=iceberg_ratio,
    ice_buy_vol=self.iceberg_buy_volume,
    ice_sell_vol=self.iceberg_sell_volume,
    event_ts=event_ts
)
```

### 5. 修改 `detect_icebergs()` 方法

#### Step E: 冰山分层
```python
ice_level = level.get_iceberg_level()
signal = IcebergSignal(
    # ...
    level=ice_level
)

# 仅统计确认冰山
confirmed_buy = [s for s in buy_signals if s.level == IcebergLevel.CONFIRMED]
self.confirmed_buy_count = len(confirmed_buy)
```

### 6. 修改 `shutdown()` 方法

#### 关闭时强制保存
```python
async def shutdown(self):
    # 强制保存状态
    self.state_saver.save(state, time.time(), force=True)
    console.print("[green]✓ 状态已保存[/green]")
```

### 7. 修改 `run()` 方法

#### 传入统一 event_ts
```python
event_ts = time.time()
data = await self.fetch_data()
if data:
    analysis = self.analyze_and_alert(data, event_ts)
```

### 8. 显示增强

冰山统计现在显示：
- 总数和确认数：`冰山买单: 5个 (确认:2)`
- 确认比：`确认比: 0.67 (买方主导)`
- 信号标记：`✓买` (确认) vs `?买` (待确认)

---

## 验收测试 (待执行)

### 测试 1: 确定性回放
同一份 events 回放 10 次，alerts hash 一致

### 测试 2: 去重正确性
同一批 trades 重复喂 2 次，CVD 不变

### 测试 3: 阈值抗污染
插入极端大单，阈值不会让系统"失明"

### 测试 4: 持久化连续性
kill 程序再启动，CVD 不归零

### 测试 5: 分层有效性
CONFIRMED 数量明显少于 ACTIVITY

### 测试 6: 背离触发一致性
价格新高但 CVD 下降时，confidence 下调

---

## 下一步

1. 运行 6 个验收测试
2. 72 小时连续运行验证
3. Phase 2: WebSocket 升级、断线重连、订单簿一致性校验

---

## 文件变更

- `alert_monitor.py` - 主要变更，集成所有模块

## 状态

✅ 集成完成，语法检查通过
