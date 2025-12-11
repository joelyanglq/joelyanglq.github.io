---
title: '回测实现之路（一）'
date: 2025-10-02
permalink: /posts/2025/10/quant/backtest-function/
tags:
  - Quant
  - Backtest
---

前两天学长刚劝我们手写一个回测框架来加深对细节的理解，结果今天MATH 507的第二次作业需要写一个回测框架。虽然不是第一次写，但之前写的时候要么是借用AI写了一部分，要么是考虑的例子比较平凡，没有自顶向下系统的思考过回测流程。

这次作业特别好的是给了一些不常见的例子，让我思考如何抽象整个流程。

# 1. 最简单的回测函数

我们这里先考虑最简单的情况，在这里先不考虑手续费、滑点、停牌、企业财年不同等因素，且使用最平凡的one-period optimization. 即t时刻决定从t到t+1的持仓，而不是确定t到t+1, t+1到t+2, ..., t+N-1到t+N共N期的持仓。

整体的思路是：
for rebalance_dates:
1. 计算从t到t+1的持仓w_t
2. 计算收益，并保存

为了计算1，我们通常需要**获取过去一个时间窗口（lookback）的数据**，比如过去一年、过去一个月、过去一周的收益率动量, ROIC（from 财报）, 自由现金流/市值 ...

同时我们的选股池可能也并非总是全市场，我们可能会根据一些原则从初始选股池中筛选出一个子集，在子集上分配权重

于是，流程可以写作
for rebalance_dates:
0. 获取lookback窗口中所需的数据
1. 计算从t到t+1的持仓w_t
2. 计算收益，并保存

# 2. 从一期到多期

之前我们是在t时刻只考虑从t到t+1的持仓，然后等到t+1时再决定t+1到t+2的持仓 ... 这叫one-period optimization。有时候，我们需要一下决定多期的持仓（multi-period optimization），比如在t时刻就=决定从t到t+1, t+1到t+2, ..., t+4到t+5共N期的持仓。

为什么要这么操作呢？我们稍后再说。

这个时候我们上述的回测流程就需要发生一些改变。

for rebalance_dates:
1. if horizon起始日：
    1. 获取lookback窗口中所需的数据
    2. 计算N期持仓
2. 根据预先计算的持仓调整
3. 计算收益

# 3. 从多期到多期
上述的多期是一个简化的情况：我们在t时刻就能确定接下来N期的持仓。即
$$w_{t:t+N-1} = f(R_{\leq t}),$$
其中$R_{\leq t}$是t时刻以前的数据。

但有时候我们可能需要更复杂的策略，

# 4.抽象

现在我们有两种回测流程，我们如何将这两种流程放到一个框架中呢？

一种做法是，我们将计算持仓和计算收益分开来，通过**两次loop**的方式，先计算得到持仓表，然后根据这个持仓表计算收益。这某种程度上避免了原来 **1-loop** 方法中计算当期权重，还是搜索权重的纠结。

沿着这个思路，我们还能注意到每期调用api获取数据是很麻烦的。获取数据也可以这一步提前储存好。

```python
import numpy as np
import pandas as pd

def backtest_on_the_fly(prices, start_date, end_date, lookback=900, horizon=100, mode="dp"):
    """
    prices: pd.DataFrame, index = DatetimeIndex (完整的历史数据, 含扩展历史)
    start_date, end_date: 回测区间（用户感知的起止）
    lookback: 用于 DP / 计算的历史窗口（天数）
    horizon: DP 的期长（天） —— 在 mode="dp" 下使用
    mode: "dp" (dynamic programming) or "rolling" (每step重算权重)
    返回: pd.Series(net_value) 和 可选的 weights_history (若需要轻量记录)
    """

    # --- 1. 保证时间范围（内部自动补历史） ---
    start_date = pd.to_datetime(start_date)
    end_date = pd.to_datetime(end_date)

    # 为保证能取到 lookback 的历史，内部扩展起始日期
    extended_start = start_date - pd.Timedelta(days=lookback * 2)
    prices = prices.loc[(prices.index >= extended_start) & (prices.index <= end_date)]
    dates = prices.index  # 全量已加载日期
    trading_dates = dates[(dates >= start_date) & (dates <= end_date)]

    # --- 2. 准备返回值 ---
    nav = [1.0]  # 初始净值
    nav_dates = []  # 对应 nav（除了起始1.0）所对应的日期（通常从第一个可交易日开始）
    last_weights = None  # 可选：保存上一次权重
    weights_log = {}     # 可选：只记录每次重建 DP 时的权重或少量样本

    # DP 相关缓存
    dp_cache = None
    horizon_start_idx = None  # trading_dates 的下标，标记 DP 起点

    # --- 3. 单 loop 回测（当天算权重，立刻算当日收益） ---
    for i, today in enumerate(trading_dates):
        # 索引位置（在全量 dates 中的位置）
        loc = dates.get_loc(today)

        # 确保有足够历史（若不够，跳过直到有足够历史）
        if loc < lookback:
            # 如果没有足够历史数据来初始化 DP / 计算权重，则跳过（或你也可以抛错误）
            continue

        # --- DP 模式：若到达新的 horizon 起点，则重建 DP_cache ---
        if mode == "dp":
            if (horizon_start_idx is None) or (i - horizon_start_idx >= horizon):
                horizon_start_idx = i
                hist_prices = prices.iloc[loc - lookback: loc]  # 使用 <= today 之前的历史
                dp_cache = run_dynamic_programming(hist_prices, horizon)
                # optional: 记录 dp 起点时的一些信息
                weights_log[today] = {"note": "dp_rebuild"}

            # 基于 dp_cache 和当日位置，计算当天权重
            t_in_horizon = i - horizon_start_idx
            weights = get_daily_weights_from_dp(dp_cache, t_in_horizon, today, prices)

        # --- rolling 模式（按 step 或每 trading day 重算） ---
        elif mode == "rolling":
            # 这里示例为每日重算；如果你想按 step 频率重算，判断 i%step==0 即可
            hist_prices = prices.iloc[loc - lookback: loc]
            weights = compute_weights(hist_prices, horizon=1)[0]  # 返回长度为 n_assets 的权重

        else:
            raise ValueError("Unsupported mode")

        # 保存最近权重（可选）
        last_weights = weights

        # --- 立即用当天计算得到的权重去计算今天->明天的组合收益 ---
        if i + 1 < len(trading_dates):
            tomorrow = trading_dates[i + 1]
            tomorrow_loc = dates.get_loc(tomorrow)

            # 简化假设：使用收盘价作为交易/估值（若有开盘价并想模拟 next-open 执行，请改为使用 Open 列）
            daily_ret = (prices.iloc[tomorrow_loc].values / prices.iloc[loc].values) - 1.0
            port_ret = float(np.dot(weights, daily_ret))  # 组合当日收益

            nav.append(nav[-1] * (1.0 + port_ret))
            nav_dates.append(tomorrow)  # nav 对应到 next day 的时间点（因为收益是从今天到明天）
        else:
            # 没有明日价格了（回测到末尾），停止
            break

    # 构造结果 Series（nav[0] = 1.0 没有日期，nav_dates 对应 nav[1:])
    nav_series = pd.Series([nav[0]] + nav[1:], index=[start_date] + nav_dates)  # 把起始点也放入
    return nav_series, weights_log


# --------------------------
# 占位函数：请把这些替换为你的真实实现
# --------------------------
def run_dynamic_programming(hist_prices, horizon):
    """
    在 horizon 起点运行 DP，返回 dp_cache（可以是 dict、函数、array 等）
    此处示例返回一个简单的序列：每个 t 返回等权 weight。
    真实场景你会返回状态价值表或策略映射表。
    """
    n_assets = hist_prices.shape[1]
    # dp_cache 示例： dict t->weights
    return {t: np.repeat(1.0 / n_assets, n_assets) for t in range(horizon)}

def get_daily_weights_from_dp(dp_cache, t_in_horizon, today, prices):
    """
    从 dp_cache 中根据 t_in_horizon 取出当日权重。
    真实实现可能还会结合当天的额外特征来微调 dp 决策。
    """
    if t_in_horizon in dp_cache:
        return dp_cache[t_in_horizon]
    else:
        # 超出 horizon 的保护：退回等权
        n_assets = prices.shape[1]
        return np.repeat(1.0 / n_assets, n_assets)

def compute_weights(hist_prices, horizon=1):
    """
    供 rolling 模式使用的权重计算器（示例）。
    返回 (horizon, n_assets) 的矩阵/ndarray。
    """
    n_assets = hist_prices.shape[1]
    return np.tile(np.repeat(1.0 / n_assets, n_assets), (horizon, 1))

```