# SKILL: 趋势K线（发财线）计算

## 依赖数据
- `tdx_get_kline`（period="day", adjust="qfq", count=500）字段：CLOSE, OPEN, HIGH, LOW
- `tdx_get_quote` 字段：现价、涨跌幅、成交额
- `tdx_get_turnover` 字段：换手率

## 计算步骤

### 第一步：HHJSJDB 加权趋势线

```
HHJSJDA = (3*CLOSE + OPEN + LOW + HIGH) / 6

HHJSJDB = (20*HHJSJDA + 19*REF(HHJSJDA,1) + 18*REF(HHJSJDA,2) + 17*REF(HHJSJDA,3)
         + 16*REF(HHJSJDA,4) + 15*REF(HHJSJDA,5) + 14*REF(HHJSJDA,6) + 13*REF(HHJSJDA,7)
         + 12*REF(HHJSJDA,8) + 11*REF(HHJSJDA,9) + 10*REF(HHJSJDA,10) + 9*REF(HHJSJDA,11)
         + 8*REF(HHJSJDA,12) + 7*REF(HHJSJDA,13) + 6*REF(HHJSJDA,14) + 5*REF(HHJSJDA,15)
         + 4*REF(HHJSJDA,16) + 3*REF(HHJSJDA,17) + 2*REF(HHJSJDA,18)
         + REF(HHJSJDA,20)) / 210

HHJSJDC = MA(HHJSJDB, 5)
```

### 第二步：决策线与分界线

```
决策线（MA60）   = MA(CLOSE, 60)
分界线（EMA108） = EMA(CLOSE, 108)
```

### 第三步：乖离率

```
乖离率 = (CLOSE - HHJSJDB) / HHJSJDB * 100
```

### 第四步：HHJSJDB 方向与金死叉

```
上升↑ = HHJSJDB > REF(HHJSJDB, 1)
下降↓ = HHJSJDB < REF(HHJSJDB, 1)

金叉             = 前根 HHJSJDB <= HHJSJDC 且 当根 HHJSJDB > HHJSJDC
死叉             = 前根 HHJSJDB >= HHJSJDC 且 当根 HHJSJDB < HHJSJDC
HHJSJDB>HHJSJDC = 持续在信号线上方（非金叉当日）
HHJSJDB<HHJSJDC = 持续在信号线下方（非死叉当日）
```

### 第五步：趋势位置

| 状态 | 条件 | 含义 |
|------|------|------|
| 强势区 | CLOSE > 决策线 > 分界线 | 多头强势，趋势向上 |
| 蓄势区 | CLOSE > 分界线，CLOSE < 决策线 | 中期偏多，短期承压 |
| 警戒区 | CLOSE < 分界线，CLOSE > 决策线 | 中期偏空，短期反弹 |
| 弱势区 | CLOSE < 分界线 < 决策线 | 空头弱势，趋势向下 |

### 第六步：K线形态判定

**优先级从高到低，只输出命中的最高优先级形态，无特殊形态则不输出**

```
RC1    = REF(CLOSE, 1)
CS     = IF(CLOSE >= 1, 10000, 100000)
C涨停5 = 1.05 * RC1 - 49 / CS

VER1   = IF(DATE <= 1200823, 1, 0)
BK     = IF(科创板, 0.2, IF(创业板 AND DATE > VER1, 0.2, IF(ST板块, 0.05, 0.1)))
涨停价  = ZTPRICE(RC1, BK)
跌停价  = DTPRICE(RC1, BK)

涨停   = CLOSE >= 涨停价 AND CLOSE = HIGH
跌停   = CLOSE <= 跌停价 AND CLOSE = LOW
假阳线  = CLOSE > OPEN AND CLOSE < REF(CLOSE, 1)
假阴线  = CLOSE < OPEN AND CLOSE > REF(CLOSE, 1)
大阳线  = CLOSE > OPEN AND (CLOSE >= C涨停5 OR CLOSE > (1.05*OPEN - 51/CS))
          OR (CLOSE > 1000 AND CLOSE > RC1 * 1.024)
```

| 优先级 | 形态 | 触发条件 |
|--------|------|---------|
| 1（最高）| 涨停 | CLOSE >= 涨停价 AND CLOSE = HIGH |
| 2 | 跌停 | CLOSE <= 跌停价 AND CLOSE = LOW |
| 3 | 假阳线 | CLOSE > OPEN AND CLOSE < REF(CLOSE,1) |
| 4 | 假阴线 | CLOSE < OPEN AND CLOSE > REF(CLOSE,1) |
| 5 | 大阳线 | CLOSE > OPEN AND (CLOSE >= C涨停5 OR CLOSE > (1.05*OPEN - 51/CS)) OR (CLOSE > 1000 AND CLOSE > RC1*1.024) |

### 第七步：均线系统（上升轨道判断）

```
MA5   = MA(CLOSE, 5)
MA10  = MA(CLOSE, 10)
MA20  = MA(CLOSE, 20)
MA60  = MA(CLOSE, 60)      // 即决策线，已计算
MA250 = MA(CLOSE, 250)
EMA108 = EMA(CLOSE, 108)   // 即分界线，已计算
```

**多头排列评分（满分6分）**

```
得分 = 0
若 CLOSE > MA20        → +1
若 MA5 > MA10          → +1
若 MA10 > MA20         → +1
若 MA20 > MA60         → +1
若 MACD > 0 且 MACD > REF(MACD,1)  → +1（红柱放大）
若 CLOSE > MA60        → +1
```

**评分解读**

| 分数 | 含义 |
|------|------|
| 6分 | 强势上升轨道，均线完全多头排列 |
| 4-5分 | 偏多，趋势向上但需确认 |
| 2-3分 | 震荡，上升轨道未形成 |
| 0-1分 | 弱势，空头排列 |

**MA250 辅助判断**
- CLOSE > MA250 → 处于牛市轨道
- CLOSE < MA250 → 处于熊市轨道
