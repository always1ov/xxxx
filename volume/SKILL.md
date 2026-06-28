# SKILL: 成交量形态 + Z值财务预警计算

## 依赖数据
- `tdx_get_kline`（period="day", adjust="qfq", count=500）字段：CLOSE, OPEN, VOL
- `tdx_get_finance` 字段：见下方 FINANCE 映射表

## 计算步骤

### 第一步：换手率

```
换手     = VOL * 10000 / FINANCE(7)     // FINANCE(7) = 流通股本（股）
十日换手  = SUM(换手, 10)
二十日换手 = SUM(换手, 20)
```

### 第二步：量能形态判定

**优先级从高到低，只输出命中的最高优先级形态**

| 优先级 | 形态名称 | 触发条件 |
|--------|---------|---------|
| 1（最高）| 地量 | VOL < REF(LLV(VOL, 100), 1) |
| 2 | 低量 | VOL < REF(LLV(VOL, 13), 1) |
| 3 | 倍量 | VOL >= REF(VOL,1) * 1.90 AND CLOSE > REF(CLOSE,1) |
| 4 | 倍缩 | VOL <= REF(VOL,1) * 0.50 |
| 5 | 梯量 | COUNT(VOL > REF(VOL,1), 3) = 3 AND COUNT(CLOSE > OPEN, 3) = 3 |
| 6 | 缩量涨 | COUNT(CLOSE > REF(CLOSE,1), 2) = 2 AND COUNT(VOL < REF(VOL,1), 2) = 2 |
| 7 | 平量 | ABS(VOL - HHV(REF(VOL,1),5)) / HHV(REF(VOL,1),5) <= 0.03 OR ABS(VOL - REF(VOL,1)) / REF(VOL,1) <= 0.03 |
| 8（兜底）| 普通量 | 以上均不满足 |

> 注意：地量同时满足低量，低量满足时不再判断优先级更低的形态，以此类推。

### 第三步：Z值财务预警（Altman Z-Score 改良版）

**FINANCE 字段映射**

| 变量 | FINANCE编号 | 含义 |
|------|------------|------|
| FINANCE(7) | 7 | 流通股本（股） |
| FINANCE(10) | 10 | 总资产 |
| FINANCE(11) | 11 | 流动资产 |
| FINANCE(15) | 15 | 流动负债 |
| FINANCE(17) | 17 | 资本公积金 |
| FINANCE(19) | 19 | 股东权益（净资产） |
| FINANCE(20) | 20 | 营业收入 |
| FINANCE(23) | 23 | 营业利润 |
| FINANCE(31) | 31 | 未分配利润 |

**计算公式**

```
X1 = (FINANCE(11) - FINANCE(15)) / FINANCE(10) * 1.2
     // 营运资本 / 总资产

X2 = (FINANCE(31) + FINANCE(17)) / FINANCE(10) * 1.4
     // (未分配利润 + 资本公积金) / 总资产
     // 注：以资本公积金代替盈余公积

X3 = FINANCE(23) / FINANCE(10) * 3.3
     // 营业利润 / 总资产

X4 = FINANCE(19) / FINANCE(15) * 0.6
     // 净资产 / 流动负债
     // 注：原版分母为负债总额，此处用流动负债代替

X5 = FINANCE(20) / FINANCE(15) * 0.999
     // 营业收入 / 流动负债

Z值 = X1 + X2 + X3 + X4 + X5
```

**预警等级**

| Z值范围 | 预警等级 | 含义 |
|---------|---------|------|
| Z < 1.2 | ●重警● | 财务高风险，接近破产警戒线 |
| 1.2 ≤ Z ≤ 2.6 | ○轻警○ | 财务需关注，灰色地带 |
| Z > 2.6 | ◎无警◎ | 财务健康 |
