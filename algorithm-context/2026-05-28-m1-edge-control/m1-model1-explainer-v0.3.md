# M1 Model 1 v0.3 解说版

日期：2026-05-28

用途：这不是正式算法 spec，而是给你开会时用的解释稿。别人问“这是什么东西”的时候，可以按这里的人话回答。

---

## 1. 这版到底在做什么

一句话：

> Model 1 先解决 import-only 场景下，BESS 什么时候主动充电、充多少，以及 EV pool 的总功率上限；安全边界由 L1 管，参数由云端发，本地 L2 只在安全边界内算目标。

再白话一点：

```text
云端告诉本地：今天 SOC 应该怎么看、现在电价算便宜还是贵、参数怎么调。
安全层告诉本地：现在能不能动、最多能充多少。
本地算法每秒看一次现场，然后输出 EV 最多能拿多少电、BESS 主动充多少电。
Model 1 里完全不做 BESS 放电。
```

---

## 2. 这版怎么来的

你可以这样解释：

```text
我们不是直接写一个大优化器。
我们先把 Model 1 范围切小，只做 BESS + EV + Import Only。

第一版只有 MIC / EV gap / BESS support 的规则。
后来 Ning 提醒需要把电价放进来。
再参考他 draft 02 里面的思路：先算物理上限，再算使用比例。

所以 v0.3 的主公式变成：
物理上限 × SOC/价格决定的使用比例 = BESS 主动充电目标功率。
```

重点：

```text
价格只影响“可用余量用多少比例”。
价格不会让功率突破 MIC、BMS、PCS 的硬限制。
```

---

## 3. L1 / L2 / L3 怎么解释

不要说成一条流程线。要说成三个 layer。

| 名字 | 人话 | 它做什么 |
|---|---|---|
| L3 | 云端参数层 | 每天/每半小时下发 model、SOC band、price rank、参数 |
| L2 | 本地算法层 | 每秒根据现场状态算输出 |
| L1 | 安全保护层 | 最快频率判断安全、设备能力、数据新鲜度 |

你可以这样讲：

```text
L1、L2、L3 不是串行跑的。
它们是三层同时存在。

L3 给 L2 目标和参数。
L1 给 L2 安全边界。
L2 只能在 L1 给出的边界里做经济调度。
```

如果 Ning 问“为什么不要串行画”：

```text
因为安全保护不应该等经济算法算完。
L1 是更快的硬保护，随时限制 L2。
而且 L2 算完之后，L1 还应该再做最终限幅或拒绝。
```

---

## 4. 每个核心词怎么解释

### 4.1 Model 1

人话：

```text
这是当前第一种业务场景。
只考虑 BESS + EV，电只能从电网进来，不考虑光伏、不卖电、不做 V2G。
```

### 4.2 MIC

MIC = Maximum Import Capacity。

人话：

```text
站点最多允许从电网拿多少功率。
比如 MIC = 500 kW，就表示这个站点从电网取电最好不要超过 500 kW。
```

如果问 MIC 从哪来：

```text
可以来自本地配置，也可以云端下发更新。
如果电网容量或合约变了，云端/配置需要能更新这个值。
```

### 4.3 margin

人话：

```text
margin 是安全余量，不是利润率。
比如 MIC 是 500 kW，margin 是 5%，算法只按大约 475 kW 去规划，避免顶到上限。
```

### 4.4 grid headroom

人话：

```text
headroom 就是电网还剩多少余量。
公式就是：MIC 扣掉站点当前负荷，再扣掉安全余量。
```

你可以说：

```text
如果 IT 能直接给“当前电网可用余量 kW”，我们可以直接用，不一定非要自己从 MIC 和 site_load 算。
```

### 4.5 site_base_load

人话：

```text
site_base_load 是站点当前基础负荷。
注意这里的 AC 是交流侧，不是空调。
```

需要和 IT 确认的点：

```text
site_base_load 不能已经包含 EV_request，否则公式里会把 EV 扣两次。
如果口径容易混，最好让 IT 直接给 site_import_headroom_kw。
```

你可以这样说：

```text
我们真正需要的是“当前电网还剩多少 kW 可以用”。
如果 IT 能直接给这个值，最稳。
```

### 4.6 EV_request

人话：

```text
EV_request 是当前所有车加起来想要的功率。
```

例子：

```text
两辆车分别想要 100 kW 和 80 kW，那 EV_request = 180 kW。
```

### 4.7 p_ev_limit_kw

人话：

```text
p_ev_limit_kw 是系统允许 EV pool 最多拿多少功率。
它就是站级的 P_gun_pool_max。
```

注意：

```text
我们只输出总上限。
每把枪怎么分，不在 Model 1 里做。
```

### 4.8 SOC

人话：

```text
SOC 是当前电池还有多少电。
它来自 BMS，是当前确定值。
```

### 4.9 SOC band

人话：

```text
SOC band 是云端给本地的一组 SOC 区间线。
本地拿当前 SOC 去判断电池现在属于偏低、中间还是偏高。
```

不要说复杂：

```text
不用解释成“概率 SOC”。
就说云端给区间阈值，本地用当前 SOC 落在哪个区间。
```

例子：

```text
soc_p25 = 40%
soc_p75 = 70%

当前 SOC = 35% -> 偏低，更倾向充电
当前 SOC = 55% -> 正常
当前 SOC = 80% -> 偏高，不主动充电
```

### 4.10 price rank / buy rank

人话：

```text
本地不直接处理原始电价。
云端先把电价变成一个 rank。
0 表示便宜，1 表示贵。
```

如果问为什么不直接给电价：

```text
因为本地只需要知道“现在相对便宜还是贵”，不需要解析复杂 tariff。
原始价格、充电价格、收益复盘都放云端更合适。
```

### 4.11 base_charge_ratio 和 cheap_price_bonus_ratio

人话：

```text
这两个都是调参表。
base_charge_ratio_by_soc_band 表示不看价格时，这个 SOC band 本来就应该用多少余量充电。
cheap_price_bonus_ratio_by_soc_band 表示如果现在买电便宜，可以额外多用多少余量。
```

例子：

```text
当前可充上限是 100 kW。
base = 0.3。
price_bonus = 0.4。
那么 charge_ratio = 0.7，BESS 目标充电 70 kW。
```

注意：

```text
这个比例最多是 1。
也就是最多用完可用余量，不会超出物理上限。
```

### 4.12 p_bess_target_kw

人话：

```text
p_bess_target_kw 是我们给 BESS/PCS 的目标功率。
正数表示充电。
0 表示不动。
Model 1 不输出负数。
```

Model 1 当前口径：

```text
充电侧公式先讲清楚。
放电不在 Model 1 里做。
```

### 4.13 SAFE_PROTECT

人话：

```text
只要安全状态不通过，经济算法不继续算。
BESS 目标功率设为 0，EV 用安全上限。
```

---

## 5. 公式怎么口头解释

不要一上来讲变量。按这四步讲：

### Step 1：先算电网还有多少余量

```text
site_import_headroom = MIC 扣掉 site_base_load，再扣掉 margin
```

意思：

```text
先知道站点现在还能安全用多少电网功率。
如果 IT 直接给 site_import_headroom，就直接用，不需要自己算。
```

### Step 2：EV 优先

```text
BESS 可主动充电上限 = site_import_headroom - EV_request
```

意思：

```text
车当前想要的先满足。
剩下的余量才给 BESS 主动充电。
```

### Step 3：看 SOC 和电价，决定用多少比例

```text
SOC 低 -> 更愿意充
SOC 高 -> 不主动充
电价便宜 -> 多充一点
电价贵 -> 少充一点
```

### Step 4：输出目标

```text
BESS 目标功率 = 可充物理上限 × 使用比例
EV limit = EV pool 允许的总功率上限
```

### 这套公式怎么来的

你可以这样讲：

```text
这套公式不是从优化器直接跳出来的。
它是从三个现实约束推出来的。
```

第一，电网有上限：

```text
站点最多不能超过 MIC。
所以先算 MIC 扣掉基础负荷和 margin 后，还剩多少电网余量。
```

第二，Model 1 里 EV 优先：

```text
EV 当前想要的功率先占用余量。
剩下的余量才给 BESS 主动充电。
```

第三，充多少不是拍脑袋：

```text
云端给 SOC band 和买电 price rank。
SOC 越低，越愿意充。
电价越便宜，越愿意用更多余量充。
所以最后变成：
可充物理上限 × 使用比例 = BESS 目标充电功率。
```

最关键的一句话：

```text
价格只影响比例，不直接加 kW。
所以它不会突破 MIC、BMS、PCS 的硬限制。
```

---

## 6. Ning / IT 可能会问什么

### Q1：L1/L2/L3 是不是串行流程？

答：

```text
不是。L1、L2、L3 是三层。
L1 是安全保护，优先级最高。
L3 是云端参数。
L2 是本地每秒算目标。
```

### Q2：site_base_load 包不包括 EV 和 BESS？

答：

```text
这需要和 IT 统一计量口径。
模型真正需要的是当前电网剩余可用功率。
如果 site_base_load 口径不稳定，可以让 IT 直接给 site_import_headroom_kw。
最重要的是不能把 EV_request 扣两次。
```

### Q3：为什么 EV 优先？

答：

```text
Model 1 当前默认先服务 EV 当前请求。
BESS 主动充电只使用 EV 之后剩下的 headroom。
如果要在 SOC 很低时让 BESS 抢一部分 headroom，需要单独确认。
```

### Q4：电价在哪里体现？

答：

```text
电价不作为原始价格给本地。
云端把电价转成 grid_buy_price_rank。
本地用这个 rank 调整 BESS 充电比例。
```

### Q5：为什么价格不是直接加 kW？

答：

```text
因为直接加 kW 可能突破 MIC 或 PCS 限制。
现在用的是“物理上限 × 使用比例”。
价格只能改变比例，不能创造额外功率。
```

### Q6：放电为什么完全不写？

答：

```text
因为这次 Model 1 的范围就是本地充电侧调度。
放电不是 Model 1 的交付内容。
所以文档里不再放 discharge 公式，也不要求 IT 实现任何放电模式。
```

### Q7：每把枪怎么分？

答：

```text
不在这版。
我们只输出 EV pool 总功率上限，也就是 P_gun_pool_max。
枪之间怎么分由 charger/IT/EMS 那层处理。
```

### Q8：L1 不通过怎么办？

答：

```text
直接 SAFE_PROTECT。
BESS 目标功率为 0。
EV limit 使用 safe_ev_limit_kw。
```

### Q9：这个是不是已经是完整本地算法？

答：

```text
这是 Model 1 的本地协调算法说明。
它不是完整 EMS，也不是底层 PCS command。
它定义的是 L2 怎么根据 L3 参数和 L1 安全边界输出站级目标。
```

### Q10：L2 算出来以后谁保证不越界？

答：

```text
L1 不只是前置 gate。
L2 算完以后，L1 / EMS 还要对最终 target 做 clamp 或 reject。
所以公式本身不应该被当成唯一保护。
```

---

## 7. 最容易讲错的地方

| 容易讲错 | 正确说法 |
|---|---|
| “本地算电价” | 本地不算原始电价，云端给 price rank |
| “SOC band 是当前 SOC 的概率” | 当前 SOC 是确定值；band 是云端给的一组区间阈值 |
| “L1 通过以后才运行 L2” | L1/L2 并行，L1 持续限制 L2 |
| “放电也按同样价格公式做” | Model 1 完全不做放电 |
| “p_ev_limit_kw 是每把枪功率” | 这是 EV pool 总上限，不是单枪分配 |
| “margin 是收益 margin” | 这是 MIC 安全余量 |
| “site_base_load 已经包含 EV 也没关系” | 不行，这会让 EV 被扣两次 |

---

## 8. 你可以直接念的版本

> 这版 v0.3 是把 Model 1 收窄成 BESS + EV、Import Only 的本地充电侧调度。云端 L3 给 model、SOC band、价格 rank 和参数；本地 L1 给安全状态和功率上限，并在最终执行前限幅；L2 每秒在这些边界里算两个核心输出：EV pool 的总功率上限 `p_ev_limit_kw`，以及 BESS 的目标功率 `p_bess_target_kw`。公式上先算 MIC 扣掉 site base load 和 margin 后的电网余量，也可以直接使用 IT 给的 `site_import_headroom_kw`；再按 EV 优先算 BESS 可主动充电上限；最后用 SOC band 和买电 price rank 决定这个上限使用多少比例。这样价格只影响比例，不会突破 MIC、BMS、PCS 的硬限制。Model 1 完全不做 BESS 放电，`p_bess_target_kw` 只会是正数或 0。
