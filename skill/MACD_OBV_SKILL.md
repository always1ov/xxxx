---
name: macd-obv
description: |
  使用自定义通达信公式计算改良版 MACD + OBV 量能共振指标。
  触发条件：用户要求技术分析、MACD 分析、量能分析、金叉死叉、OBV 共振时自动调用。
  依赖 tdx-mcp 提供的 K 线数据（close、volume）。
version: 1.0.0
---

# MACD-OBV 量能共振指标 Skill

基于用户自定义通达信公式，结合改良 MACD 与 OBV 量能信号，识别金叉/死叉及量价共振买点。

## 公式原文（不得修改任何逻辑与字符）

```
DIFF:=( EMA(CLOSE,12) - EMA(CLOSE,26));
DEA:=EMA(DIFF,9);
MACD:=2*(DIFF-DEA),STICK;
STICKLINE(DIFF< 0,0,DIFF,2,0),COLORGREEN;
STICKLINE(DIFF>=0,0,DIFF,2,0),COLORRED;
STICKLINE(DEA>=0,0,DEA,2,-1),COLOR0000CC;
STICKLINE(DEA< 0,0,DEA,2,-1),COLORGREEN;
柱1:=IF(DIFF>DEA,DIFF,0),COLORRED;
柱2:=IF(DEA< DIFF,DEA,0),COLORMAGENTA;
DRAWICON(CROSS(DIFF,DEA),柱2,1);
DRAWICON(CROSS(DEA,DIFF),DEA*1.1,2);
VA:=IF(CLOSE>REF(CLOSE,1),VOL,-VOL);
OBV1:=SUM(IF(CLOSE=REF(CLOSE,1),0,VA),0);
OBV2:=EMA(OBV1,3)-MA(OBV1,9);
OBV3:=EMA(IF(OBV2>0,OBV2,0),3);
MAC3:=MA(C,3);
STICKLINE(OBV3>REF(OBV3,1) AND MAC3>REF(MAC3,1),0,DEA/4,2,0),COLORYELLOW;
```

---

## 数据来源

使用 `tdx_get_kline` 获取日 K 线数据：

```
tdx_get_kline(code="sh600519", period="day", adjust="qfq", count=250)
```

所需字段：`close`（收盘价）、`volume`（成交量，单位：手）

---

## 计算步骤（严格按公式逻辑，不得改动）

### 第一步：EMA 计算函数

EMA(series, n) 递推公式：
```
EMA[0] = series[0]
EMA[i] = EMA[i-1] * (n-1)/(n+1) + series[i] * 2/(n+1)
```

### 第二步：MACD 三线

```
DIFF = EMA(CLOSE, 12) - EMA(CLOSE, 26)
DEA  = EMA(DIFF, 9)
MACD = 2 * (DIFF - DEA)
```

### 第三步：柱1 / 柱2

```
柱1 = DIFF  （当 DIFF > DEA 时）；否则 = 0
柱2 = DEA   （当 DEA < DIFF 时）；否则 = 0
```

### 第四步：金叉 / 死叉信号

CROSS(A, B) 定义：前一根 A < B，当前 A >= B（A 上穿 B）

```
金叉信号：CROSS(DIFF, DEA)  → 在柱2位置画图标1（↑）
死叉信号：CROSS(DEA, DIFF)  → 在 DEA*1.1 位置画图标2（↓）
```

### 第五步：OBV 量能

```
VA   = VOL   （当 CLOSE > REF(CLOSE,1) 时）；否则 = -VOL
OBV1 = 累计求和：若 CLOSE = REF(CLOSE,1) 则取 0，否则取 VA（从第1根累加到当前）
OBV2 = EMA(OBV1, 3) - MA(OBV1, 9)
OBV3 = EMA( IF(OBV2>0, OBV2, 0) , 3 )
MAC3 = MA(CLOSE, 3)
```

### 第六步：量价共振信号

```
共振条件：OBV3 > REF(OBV3,1)  AND  MAC3 > REF(MAC3,1)
满足时：在 0 到 DEA/4 之间画黄色柱（量能托底信号）
```

---

## 信号解读规则

| 信号 | 条件 | 含义 |
|------|------|------|
| 金叉 | CROSS(DIFF, DEA) | DIFF 上穿 DEA，潜在买点 |
| 死叉 | CROSS(DEA, DIFF) | DEA 上穿 DIFF，潜在卖点 |
| 黄柱共振 | OBV3↑ AND MAC3↑ | 量能与价格同步上升，买点信号增强 |
| DIFF > 0 且金叉 | DIFF>0 + 金叉 | 多头区域金叉，信号强 |
| DIFF < 0 且金叉 | DIFF<0 + 金叉 | 空头区域金叉，信号弱，需配合黄柱确认 |
| 黄柱 + 金叉共振 | 两者同一根K线 | 最强买点信号 |

---

## 输出格式

计算完成后，输出以下内容：

```
【MACD-OBV 指标】股票名称 / 代码 · 数据日期
DIFF（当前）：xxx
DEA（当前） ：xxx
MACD柱     ：xxx（正/负）
OBV3趋势   ：上升 / 下降
MAC3趋势   ：上升 / 下降

最近信号：
- 金叉：第 N 根K线（日期）
- 死叉：第 N 根K线（日期）
- 黄柱共振：最近出现日期

综合研判：（多头 / 空头 / 震荡，附信号组合说明）
```

---

## 注意事项

- EMA 需要足够的历史数据预热，建议取 250 根以上 K 线
- OBV1 为历史全量累计，250 根内计算结果为近似值，趋势判断有效，绝对值仅供参考
- 公式逻辑与字符不得修改，计算结果如与通达信软件存在微小浮点差异属正常
