---
name: volume-z
description: |
  使用自定义通达信公式识别成交量形态分类（倍量/低量/地量/平量/倍缩/梯量/缩量涨）
  并计算 Altman Z-Score 变体进行财务健康预警（重警/轻警/无警）。
  触发条件：用户要求量能分析、成交量形态、财务预警、Z值分析时自动调用。
  依赖 tdx-mcp 提供的 K 线数据（volume、close、open）及财务数据（tdx_get_finance）。
version: 1.0.0
---

# 量能分类 + Z值财务预警 Skill

基于用户自定义通达信公式，识别当日成交量形态，并通过 Altman Z-Score 变体评估财务健康度。

## 公式原文（不得修改任何逻辑与字符）

```
VVOL:=IF(CURRBARSCOUNT=1 AND PERIOD=5,VOL*240/FROMOPEN,DRAWNULL),NODRAW;
STICKLINE(CURRBARSCOUNT=1 AND PERIOD=5,VVOL,0,-1,-1),COLOR00C0C0;
VOLUME:VOL,VOLSTICK;
换手:VOL*10000/FINANCE(7);
十日换手:SUM(换手,10);
二十日换手:SUM(换手,20);
倍量:VOL>=REF(V,1)*1.90 AND C>REF(C,1),NODRAW,COLORYELLOW;
低量:VOL<REF(LLV(VOL,13),1),NODRAW,COLORGREEN;
地量:VOL<REF(LLV(VOL,100),1),NODRAW,COLORMAGENTA;
平量:ABS(VOL-HHV(REF(VOL,1),5))/HHV(REF(VOL,1),5)<=0.03 OR ABS(VOL-REF(VOL,1))/REF(VOL,1)<=0.03,NODRAW,COLORWHITE;
倍缩:VOL<=REF(V,1)*0.5,NODRAW,COLORRED;
梯量:COUNT(V>REF(V,1),3)=3 AND COUNT(C>O,3)=3,NODRAW,COLOR824173;
缩量涨:COUNT(C>REF(C,1),2)=2 AND COUNT(V<REF(V,1),2)=2,NODRAW,COLORBLUE;
STICKLINE(倍量,0,V,0.5,0),COLORYELLOW;
STICKLINE(低量,0,V,0.5,0),COLORGREEN;
STICKLINE(地量,0,V,0.5,0),COLORMAGENTA;
STICKLINE(平量,0,V,0.5,0),COLORWHITE;
STICKLINE(倍缩,0,V,0.5,0),COLORRED;
STICKLINE(梯量,0,V,0.5,0),COLOR824173;
STICKLINE(缩量涨,0,V,0.5,0),COLORBLUE;
X1:=(FINANCE(11)-FINANCE(15))/FINANCE(10)*1.2;
X2:=(FINANCE(31)+FINANCE(17))/FINANCE(10)*1.4;
X3:=FINANCE(23)/FINANCE(10)*3.3;
X4:=FINANCE(19)/FINANCE(15)*0.6;
X5:=FINANCE(20)/FINANCE(15)*0.999;
Z值:=X1+X2+X3+X4+X5;
DRAWTEXT_FIX(1,0.75,0.01,1,'财务预警：'),COLORRED;
DRAWTEXT_FIX(Z值<1.2,0.851,0.01,1,' ●重 警●'),COLORFFFFFF;
DRAWTEXT_FIX(BETWEEN(Z值,1.2,2.6),0.851,0.01,1,' ○轻 警○'),COLOR0099FF;
DRAWTEXT_FIX(Z值>2.6,0.851,0.01,1,' ◎无 警◎'),COLORYELLOW;
```

---

## 数据来源

**K 线数据**（量能分类）：
```
tdx_get_kline(code="sh600519", period="day", adjust=None, count=120)
```
所需字段：`volume`（成交量）、`close`（收盘价）、`open`（开盘价）、`high`（最高价）、`low`（最低价）

> 注意：量能分类使用不复权数据（adjust=None），成交量不能复权

**财务数据**（Z值计算）：
```
tdx_get_finance(code="sh600519")
```
所需字段（FINANCE序号对应）：
| 序号 | 字段含义 | tdx_get_finance 字段 |
|------|---------|---------------------|
| FINANCE(7) | 流通股本（股） | `float_shares` |
| FINANCE(10) | 总资产 | `total_assets` |
| FINANCE(11) | 流动资产 | 需从 `total_assets - net_assets` 估算，或标注不可得 |
| FINANCE(15) | 流动负债 | 需从财务数据推算，或标注不可得 |
| FINANCE(17) | 未分配利润 | 需从财务数据推算，或标注不可得 |
| FINANCE(19) | 净资产（股东权益） | `net_assets` |
| FINANCE(20) | 营业收入 | `main_revenue` |
| FINANCE(23) | 营业利润 | `net_profit`（近似替代） |
| FINANCE(31) | 资本公积金 | 不可得，标注缺失 |

> ⚠️ tdx_get_finance 返回字段有限，部分 FINANCE 序号无法精确对应。Z值计算时：
> - 可得字段正常计算
> - 不可得字段标注"字段不可得，Z值为估算"
> - Z值结论仍可输出，但需注明为估算值

---

## 计算步骤（严格按公式逻辑，不得改动）

### 第一部分：量能分类

取最近 100 根 K 线，对**当前最后一根**判断以下条件（互不排斥，可同时成立）：

```
倍量 = VOL >= REF(VOL,1)*1.90  AND  CLOSE > REF(CLOSE,1)
低量 = VOL < REF(LLV(VOL,13),1)          # 成交量 < 前13日最低量
地量 = VOL < REF(LLV(VOL,100),1)         # 成交量 < 前100日最低量
平量 = ABS(VOL-HHV(REF(VOL,1),5))/HHV(REF(VOL,1),5)<=0.03
       OR ABS(VOL-REF(VOL,1))/REF(VOL,1)<=0.03
倍缩 = VOL <= REF(VOL,1)*0.5
梯量 = COUNT(VOL>REF(VOL,1),3)=3  AND  COUNT(CLOSE>OPEN,3)=3  # 连续3日量增阳线
缩量涨 = COUNT(CLOSE>REF(CLOSE,1),2)=2  AND  COUNT(VOL<REF(VOL,1),2)=2  # 连续2日缩量上涨
```

辅助函数：
- `LLV(series, n)` = 最近 n 日最低值
- `HHV(series, n)` = 最近 n 日最高值
- `REF(series, n)` = n 日前的值
- `COUNT(condition, n)` = 最近 n 日满足条件的天数

### 第二部分：换手率

```
换手 = VOL * 10000 / float_shares      # 单日换手率（%）
十日换手 = SUM(换手, 10)               # 10日累计换手
二十日换手 = SUM(换手, 20)             # 20日累计换手
```

### 第三部分：Z值财务预警

```
X1 = (流动资产 - 流动负债) / 总资产 * 1.2
X2 = (资本公积金 + 未分配利润) / 总资产 * 1.4
X3 = 营业利润 / 总资产 * 3.3
X4 = 净资产 / 流动负债 * 0.6
X5 = 营业收入 / 流动负债 * 0.999
Z值 = X1 + X2 + X3 + X4 + X5
```

预警判断：
```
Z值 < 1.2          → ●重 警● （财务风险高）
1.2 ≤ Z值 ≤ 2.6   → ○轻 警○ （财务存在隐患）
Z值 > 2.6          → ◎无 警◎ （财务健康）
```

---

## 信号解读规则

### 量能形态

| 形态 | 条件 | 含义 |
|------|------|------|
| 倍量 🟡 | 量≥前日1.9倍且收涨 | 放量上攻，主力介入信号 |
| 低量 🟢 | 量<前13日最低量 | 近期缩量，观望情绪浓 |
| 地量 🟣 | 量<前100日最低量 | 历史地量，筑底信号，关注反转 |
| 平量 ⬜ | 量变化≤3% | 成交量持平，方向待定 |
| 倍缩 🔴 | 量≤前日0.5倍 | 急剧缩量，做多意愿骤降 |
| 梯量 🟤 | 连续3日量增+收阳 | 温和放量上攻，健康趋势信号 |
| 缩量涨 🔵 | 连续2日缩量上涨 | 惜售特征，筹码锁定，看涨 |

> 可同时出现多个形态，需组合判断

### Z值预警

| 预警 | Z值 | 含义 |
|------|-----|------|
| ●重 警● | < 1.2 | 财务困境风险高，谨慎持有 |
| ○轻 警○ | 1.2 ~ 2.6 | 财务存在隐患，关注后续报告 |
| ◎无 警◎ | > 2.6 | 财务健康，无明显风险 |

---

## 输出格式

```
【量能 + 财务预警】
当日量能：{形态emoji} {形态名称}（可多个）
换手率：{今日换手}% · 十日累计 {值}% · 二十日累计 {值}%
财务预警：{预警等级}（Z值 {值}，{正常计算/估算-注明缺失字段}）
```

---

## 注意事项

- 量能分类使用不复权 K 线（adjust=None），成交量不可复权
- 地量判断需要至少 101 根 K 线，建议取 120 根
- 多个量能形态可同时成立，输出时全部列出
- Z值部分字段 tdx_get_finance 可能无法精确提供，输出时注明估算
- 公式逻辑与字符不得修改
