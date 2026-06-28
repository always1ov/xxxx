# SKILL: MACD-OBV 量价共振计算

## 依赖数据
- `tdx_get_kline`（period="day", adjust="qfq", count=500）
- 字段：CLOSE, VOL

## 计算步骤

### 第一步：MACD 三线

```
DIFF = EMA(CLOSE, 12) - EMA(CLOSE, 26)
DEA  = EMA(DIFF, 9)
MACD = 2 * (DIFF - DEA)
```

### 第二步：OBV 量能共振

```
VA   = IF(CLOSE > REF(CLOSE,1), VOL, -VOL)
OBV1 = SUM(IF(CLOSE = REF(CLOSE,1), 0, VA), 0)   // 全历史累积求和
OBV2 = EMA(OBV1, 3) - MA(OBV1, 9)
OBV3 = EMA(IF(OBV2 > 0, OBV2, 0), 3)
MAC3 = MA(CLOSE, 3)
```

### 第三步：信号判定

```
金叉     = 前根 DIFF < DEA 且 当根 DIFF > DEA
死叉     = 前根 DIFF > DEA 且 当根 DIFF < DEA
黄柱共振  = OBV3 > REF(OBV3,1) AND MAC3 > REF(MAC3,1)
黄柱减弱  = OBV3 与 MAC3 任一方向背离（不满足共振但曾共振）
无黄柱   = 不满足共振条件
最强买点  = 金叉与黄柱共振同日出现
共振消失  = 前根黄柱共振，当根不满足
```

### 第四步：状态描述

每根 K 线输出三个维度：

**零轴位置**
- `多头区间` — DIFF > 0
- `空头区间` — DIFF < 0
- `零轴上穿` — DIFF 由负转正（当根首次 > 0）

**柱方向**
- `红柱放大` — MACD > 0 且 > REF(MACD,1)
- `红柱缩短` — MACD > 0 且 < REF(MACD,1)
- `绿柱放大` — MACD < 0 且 ABS(MACD) > ABS(REF(MACD,1))
- `绿柱缩短` — MACD < 0 且 ABS(MACD) < ABS(REF(MACD,1))
- `零轴穿越` — MACD 由正转负或由负转正

**黄柱状态**
- `黄柱共振` / `黄柱减弱` / `无黄柱`（见上）

## 近期关键信号标注

扫描最近 10 个交易日，命中则记录，无信号日期跳过：

| 标注 | 触发条件 |
|------|---------|
| ⚡ 最强买点 | 金叉与黄柱共振同日 |
| 🔴 金叉↑↑ | DIFF 上穿 DEA |
| 🟢 死叉↓↓ | DEA 上穿 DIFF |
| 🟡 黄柱持续 | OBV3 与 MAC3 同向放大 |
| ⚫ 共振消失 | 黄柱状态结束 |
