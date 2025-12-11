---
title: "Option那些事（一）：隐含波动率曲面初探"
date: 2025-12-05
collection: projects
permalink: /projects/2025/12/quant/iv-surface/
excerpt: 本文中尝试根据yfinance期权合约数据，构建隐含波动率曲面
tags:
  - Quant
  - Option
  - Implied Volatility
header:
  teaser: qqq-iv-surface-svi3d.png
---

本篇文章的灵感来源于看到 @[猫佬](https://www.zhihu.com/people/yan-guan-lin-20) 的知乎回答 https://www.zhihu.com/question/1921237127863202。

> 在构建波动曲面、选市场合约价格点的时候（你们不会“全市场”的合约都选吧？不会吧？不会吧？）近距离价外期权和深度价外期权的权重之比不妨设为 7:3，度价外的对数在值度约为 0.6~0.7。

我一想，还真不知道为什么不是用所有合约点。于是决定做一个实验，看看会发生什么？

# 数据

为简单验证，我选择 yfinance 作为数据源。yfinance 免费提供 option snapshot 数据，对于我们做波动率曲面已经够用了。

# 实验1：选用全市场数据 v.s. 只使用 OTM 数据

图 1 对比了使用全市场的合约和只使用 OTM 合约。有几个推翻我此前印象的发现：

1. 我以为同一 strike、dte 的 put 和 call，iv 相同。但实际上 call 和 put 的 iv 并不在一条曲线上。
2. 我以为波动率曲线是在 atm 处最小，而实际数据无论是 QQQ 期权还是 SPY 期权，argmin(iv) 都大于 spot price，也就是 iv curve 是左偏的（左侧价格点也比右侧多），而且左侧 iv 曲线比右侧陡峭。
3. jay 提到不用 ibkr 提供的 greeks，得自己手写一个 bs solver，我还纳闷为什么。但自己解出来和 yf 提供的确实有些差别。

![图1： QQQ期权在20251204上午11:45时的隐含波动率曲线](/images/qqq_iv_curve.png)

首先，从图中可以看到 call 和 put 的 iv 曲线其实是两条，两条曲线大致在 atm 处相交（相切）。不过 QQQ 和 SPY 的曲线位置则有所不同。QQQ 是相交穿过，而 SPY 是相切，被包着。

其次，即使是我们使用 OTM 合约点，我们也发现左边（Put OTM）很高，右边（Call OTM）很低，同时 iv 最低点并非 atm。

Gemini 解释说，这是著名的 Volatility Smirk 现象。来源于（或者反映了）投资者非常害怕市场暴跌，所以疯狂买入虚值看跌期权（OTM Put）做保险。这推高了左侧的 IV。这种现象在美股中常见，如果画外汇的 IV，就是对称的 smile。

另一个有趣的现象是，到期日较近的合约 iv 曲面是右侧能钩上去的，而到期日较远的合约 iv 曲面右侧则彻底躺平成标准的 smirk。

第三，关于供应商提供的 iv。为什么要自己算，我觉得主要有两个原因：

1. 不知道供应商提供的 greeks 是否存在延迟。
2. 第二个则更核心，不知道供应商的 iv 是怎么算的，比如 price 使用 mid price 还是 last price，利率 r 和 dividend q 是怎么估计，T 是用交易日还是日历日。这些都是问题。

# 实验2：更精细的 data recipe

在进行下一个实验前，我们需要说明一下整个流程。上面我们只 visualize 了 iv 的散点模式。光有 (T, moneyness, iv) 的 data point 还不够：原始的散点图有空洞（某些 Strike 没交易），且有噪音（Bid-Ask 跳动）。我们不能直接用散点图定价，必须用数学模型把它“平滑化”。

## 第二步：SVI 校准 (Stochastic Volatility Inspired)

对于每一个固定的到期日 $T$，我们假设 IV 符合 Raw SVI 公式：
$$\sigma_{BS}^2(k) = a + b (\\rho(k - m) + \\sqrt{(k - m)^2 + \\sigma^2})$$
- k: Log-Moneyness (ln(K/S))
- $\{a, b, \\rho, m, \\sigma\}$: 5 个待定参数。

- 流程：切片 (Slicing)：把数据按到期日分组。比如 SPY 有 10 个到期日，就分成 10 组。
- 最优化 (Optimization)：对于每一组，使用最小二乘法（scipy.optimize.minimize）寻找最优的 5 个参数，使得模型曲线穿过所有的散点。

结果：你不再需要成百上千个散点，只需要 10 组参数就能完整描述整个市场。

## 第三阶段：时间维度的插值 (Temporal Interpolation)

在 2D 线条变成 3D 曲面做完 SVI 校准后，你在特定的日期（如 12/0X 或 1/7）有了完整的微笑曲线。但如果要跨日到期的期权定价怎么办？

**全方差插值 (Total Variance Interpolation)**：通常在“总方差”($\\sigma^2 T$) 空间进行线性插值。
$$v(t) = \\frac{t - t_1}{t_2 - t_1} v(t_2) + \\frac{t_2 - t}{t_2 - t_1} v(t_1)$$

这保证了随着时间推移，不确定性是线性增长的，避免套利。

通常最后还需要进行第四阶段：无套利检查 (Arbitrage Check)

- 日历价差 (Calendar Arbitrage)：远期总方差必须大于近期总方差（$\\partial (T\\sigma^2) / \\partial T > 0$）。不满足意味着可以“买近期、卖远期”无风险获利。
- 蝶式价差 (Butterfly Arbitrage)：概率密度函数必须非负。SVI 某些参数组合可能导致负概率，必须剔除或加惩罚项重算。

![图中展示了我们通过这两步画出的3Div曲面](/images/qqq-iv-surface-svi3d.png)

下面两张图对比了我们用全部的 OTM 期权和调整 deep OTM 与 near OTM 比例 3:7 得到的结果：

![alt text](/images/qqq-iv-surface-pure_otm.png)

![alt text](/images/qqq-iv-surface-prop_otm.png)

按理说应该加一个评估的。不过我还没搞懂怎么评估。

# 总结

总结一下，我利用 yfinance 的 option chain snapshot 数据，完成了构建 Implied volatility 曲面所需的数据清洗、曲面构建全流程。从中学习到了美股期权 iv 的 skew 性质，对 iv 近陡远平的 term structure 有了直观的认识。
