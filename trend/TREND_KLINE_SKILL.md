---
name: trend-kline
description: |
  使用自定义通达信公式计算加权趋势线（HHJSJDB）、决策线（MA60）、分界线（EMA108）、
  乖离率，并识别K线形态（涨停/跌停/大阳线/假阳线/假阴线）。
  触发条件：用户要求趋势分析、趋势线、乖离率、K线形态识别时自动调用。
  依赖 tdx-mcp 提供的 K 线数据（close、open、high、low）。
version: 1.0.0
---

# 趋势判断 + K线形态 Skill

基于用户自定义通达信公式，通过加权趋势线与多条均线判断趋势方向，同时识别当日K线特殊形态。

## 公式原文（不得修改任何逻辑与字符）

```
HHJSJDA:=(3*CLOSE+OPEN+LOW+HIGH)/6;
HHJSJDB:(20*HHJSJDA+19*REF(HHJSJDA,1)+18*REF(HHJSJDA,2)+17*REF(HHJSJDA,3)+16*REF(HHJSJDA,4)+15*REF(HHJSJDA,5)+14*REF(HHJSJDA,6)
+13*REF(HHJSJDA,7)+12*REF(HHJSJDA,8)+11*REF(HHJSJDA,9)+10*REF(HHJSJDA,10)+9*REF(HHJSJDA,11)+8*REF(HHJSJDA,12)
+7*REF(HHJSJDA,13)+6*REF(HHJSJDA,14)+5*REF(HHJSJDA,15)+4*REF(HHJSJDA,16)+3*REF(HHJSJDA,17)+2*REF(HHJSJDA,18)+
REF(HHJSJDA,20))/210,COLORYELLOW;
HHJSJDC:MA(HHJSJDB,5),COLORRED;
VAR1:HHJSJDB;
上升:IF(VAR1>REF(VAR1,1),VAR1,DRAWNULL),COLOR00FFFF;
下降:IF(VAR1<REF(VAR1,1),VAR1,DRAWNULL),COLORGREEN;
决策线:MA(CLOSE,60),POINTDOT,COLORGREEN;
分界线:EMA(CLOSE,108),POINTDOT,COLOR246BFF;
乖离率:(C-HHJSJDB)/HHJSJDB*100,NODRAW,COLORWHITE;
VER1:=IF(DATE<=1200823,1,0);
BK:=IF(INBLOCK('科创板'),0.2,IF(INBLOCK('创业板') AND DATE>VER1,0.2,IF(INBLOCK('ST板块'),0.05,0.1)));
涨停:=C>=ZTPRICE(REF(C,1),BK) AND C=H;
跌停:=C<=DTPRICE(REF(C,1),BK) AND C=L;
STICKLINE((C=ZTPRICE(REF(CLOSE,1),0.1) AND C=H),C,O,0.2,0),COLORYELLOW;
STICKLINE((C=ZTPRICE(REF(CLOSE,1),0.1) AND C=L),C,O,0.2,0),COLORGREEN;
RC1:=REF(C,1);
CS:=IF(C>=1,10000,100000);
C涨停10:=1.10*RC1-49/CS;
C涨停5:=1.05*RC1-49/CS;
C跌停10:=0.90*RC1+51/CS;
C跌停5:=0.95*RC1+51/CS;
大阳线:=C>O AND (C>=C涨停5 OR C>(1.05*O-51/CS)) OR (C>1000 AND C>RC1*1.024);
STICKLINE(涨停,O,C,0.2,0),COLORYELLOW;
STICKLINE(跌停,O,C,0.2,0),COLORGREEN;
BK1:=IF(INBLOCK('科创板'),0.2,IF(INBLOCK('ST板块'),0.05,0.1));
STICKLINE((C=ZTPRICE(REF(CLOSE,1),0.1) AND C=H),C,O,0.2,0),COLORYELLOW;
STICKLINE((C=ZTPRICE(REF(CLOSE,1),0.1) AND C=L),C,O,0.2,0),COLORGREEN;
STICKLINE((C=ZTPRICE(REF(CLOSE,1),0.1) AND C<H AND C<O),C,O,0.2,0),COLORGREEN;
STICKLINE((C=ZTPRICE(REF(CLOSE,1),0.1) AND C<H AND C>O),C,O,0.2,0),COLORRED;
假阳线:=C>O AND C<REF(C,1);
STICKLINE(假阳线,C,O,0.2,0),COLORFFFF00;
假阴线:=C<O AND C>REF(C,1);
STICKLINE(假阴线,C,O,0.2,0),COLOR0000FF;
```

---

## 数据来源

```
tdx_get_kline(code="sh600519", period="day", adjust="qfq", count=250)
```

所需字段：`close`、`open`、`high`、`low`

---

## 计算步骤（严格按公式逻辑，不得改动）

### EMA / MA 计算函数

```
MA(series, n)  = 最近 n 根收盘价简单平均
EMA[0] = series[0]
EMA[i] = EMA[i-1] * (n-1)/(n+1) + series[i] * 2/(n+1)
```

### 第一步：HHJSJDA（典型价格加权）

```
HHJSJDA = (3*CLOSE + OPEN + LOW + HIGH) / 6
```

### 第二步：HHJSJDB（线性加权趋势线，权重递减）

```
HHJSJDB = (
  20*HHJSJDA[0] + 19*HHJSJDA[1] + 18*HHJSJDA[2] + 17*HHJSJDA[3] +
  16*HHJSJDA[4] + 15*HHJSJDA[5] + 14*HHJSJDA[6] + 13*HHJSJDA[7] +
  12*HHJSJDA[8] + 11*HHJSJDA[9] + 10*HHJSJDA[10] + 9*HHJSJDA[11] +
  8*HHJSJDA[12] + 7*HHJSJDA[13] + 6*HHJSJDA[14] + 5*HHJSJDA[15] +
  4*HHJSJDA[16] + 3*HHJSJDA[17] + 2*HHJSJDA[18] + HHJSJDA[20]
) / 210
```

> 注：权重总和 = 20+19+...+1 = 210，下标 0 为当前，1 为昨日，以此类推

### 第三步：HHJSJDC（信号线）

```
HHJSJDC = MA(HHJSJDB, 5)
```

### 第四步：趋势方向

```
上升 = HHJSJDB[0] > HHJSJDB[1]
下降 = HHJSJDB[0] < HHJSJDB[1]
```

### 第五步：决策线与分界线

```
决策线 = MA(CLOSE, 60)
分界线 = EMA(CLOSE, 108)
```

### 第六步：乖离率

```
乖离率 = (CLOSE - HHJSJDB) / HHJSJDB * 100
```

### 第七步：K线形态识别（当日最后一根）

**涨跌停判断**（按板块区分幅度）：
```
科创板/创业板：幅度 20%
ST板块：幅度 5%
其他：幅度 10%

涨停 = CLOSE >= 涨停价 AND CLOSE = HIGH
跌停 = CLOSE <= 跌停价 AND CLOSE = LOW
```

**大阳线**：
```
大阳线 = (CLOSE > OPEN AND (CLOSE >= 1.05*REF(CLOSE,1) OR CLOSE > 1.05*OPEN))
          OR (CLOSE > 1000 AND CLOSE > REF(CLOSE,1)*1.024)
```

**假阳线**（收阳但低于昨收，向上的阴线）：
```
假阳线 = CLOSE > OPEN AND CLOSE < REF(CLOSE,1)
```

**假阴线**（收阴但高于昨收，向下的阳线）：
```
假阴线 = CLOSE < OPEN AND CLOSE > REF(CLOSE,1)
```

---

## 信号解读规则

### 趋势位置

| 状态 | 条件 | 含义 |
|------|------|------|
| 强势区 | 收盘 > 决策线 > 分界线 | 多头强势，趋势向上 |
| 蓄势区 | 收盘 > 分界线，收盘 < 决策线 | 中期偏多，短期承压 |
| 警戒区 | 收盘 < 分界线，收盘 > 决策线 | 中期偏空，短期反弹 |
| 弱势区 | 收盘 < 分界线 < 决策线 | 空头弱势，趋势向下 |

### HHJSJDB 方向

| 状态 | 含义 |
|------|------|
| 上升（HHJSJDB↑）| 趋势线向上，多头动能 |
| 下降（HHJSJDB↓）| 趋势线向下，空头动能 |

### HHJSJDB 与 HHJSJDC 关系

| 状态 | 含义 |
|------|------|
| HHJSJDB > HHJSJDC | 趋势线在信号线上方，偏多 |
| HHJSJDB < HHJSJDC | 趋势线在信号线下方，偏空 |
| HHJSJDB 上穿 HHJSJDC | 金叉，买入信号 |
| HHJSJDB 下穿 HHJSJDC | 死叉，卖出信号 |

### 乖离率

| 乖离率 | 含义 |
|--------|------|
| > +10% | 价格严重偏高，回调风险大 |
| +5% ~ +10% | 偏高，注意减仓 |
| -5% ~ +5% | 正常区间 |
| -5% ~ -10% | 偏低，关注支撑 |
| < -10% | 严重超跌，关注反弹 |

### K线形态

| 形态 | emoji | 含义 |
|------|-------|------|
| 涨停 | 🟡 | 封板涨停，强势信号 |
| 跌停 | 🟣 | 封板跌停，弱势信号 |
| 大阳线 | 🔴 | 强势上攻，多头主导 |
| 假阳线 | ⚠️ | 收阳但低于昨收，上涨陷阱，警惕 |
| 假阴线 | 💙 | 收阴但高于昨收，下跌陷阱，关注反转 |
| 普通K线 | — | 无特殊形态 |

---

## 注意事项

- HHJSJDB 需要至少 21 根 K 线预热，建议取 250 根
- 决策线 MA60 需 60 根，分界线 EMA108 需 108 根预热
- 涨跌停幅度按板块区分，tdx_get_kline 不含板块信息，默认按 10% 计算，科创板/创业板需用户说明
- 公式逻辑与字符不得修改
