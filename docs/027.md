# 使用 Python 开发均值回归交易策略

> 原文：<https://www.trality.com/blog/mean-reversion-trading-strategy/>

你小时候玩过乐高积木吗？那些五颜六色、形状和大小各异的塑料联锁块？如果你有一个更小的套装，那么你也许可以在没有说明书的情况下组装这些部件，最终得到类似盒子上图片的东西。但是对于更大更复杂的集合，如果你想把所有的部分正确地组合在一起，你真的必须非常仔细地按照说明去做。否则，你将只剩下一些随意组装的五颜六色的塑料碎片——这可不是什么好东西。

这听起来有点牵强，但是设计交易策略是非常相似的。当然，一个简单的交易策略可以很快形成，但不会给人留下特别深刻的印象。对于其他任何事情，如果你想有机会盈利，你需要清楚地了解所有的战略构建模块是如何依次相互联系的。

那么，把这篇文章当作你的指导手册或模板，当你设计和组合你自己的策略时，你可以参考它作为指导或想法。跟随我们向您介绍一个完整的**均值回归交易策略**，并解释使用该策略的基本原理。到本文结束时，你将能够用状态过滤器识别市场状态，选择和使用正确的指标，并将信号转化为交易头寸。虽然需要一些组装，但设计良好、有利可图的交易策略是一件美妙的事情。

现在是时候将所有这些理论和知识付诸实践了。我们将涉及 **1)用状态过滤器识别市场状态，2)使用指标预测交易头寸的价值，以及 3)转换这个信号**

## 均值回归交易策略描述

我们的案例研究侧重于均值回归策略，其操作原则是，当价格处于局部极值时，它更有可能恢复到长期平均水平，而不是继续保持相同的方向。这些策略通常以几个更大的损失为代价获得高比例的盈利交易，当市场没有趋势时，通常会导致合理的可预测的盈亏轨迹。值得注意的是，均值回归策略依赖于合理的波动率估计，以避免波动率快速变化时的风险过大。

## 均值回归交易策略设计

完整的交易策略需要几个核心要素。

*   **信号**
    信号与价格相对于预期的未来平均价格的高低有关。该信号的一个重要特点是，它与潜在利润成比例，通过不对不确定的交易机会承担过多风险，帮助平滑利润和损失。
*   **头寸管理**
    头寸管理将信号转化为交易头寸，即它计算在每个时间步买入和卖出多少，以实现期望的风险。重要的是，风险与策略的利润预期成比例。它还应该在时间上保持平衡，当波动性较低时承担更多风险，当波动性较高时承担更少风险。
*   **制度过滤**
    简单的交易策略很可能不会在所有市场条件下都奏效。制度过滤器的工作是将交易活动限制在策略预期具有最佳风险调整回报的时间。

### 均值回复信号

均值回归策略的信号是由它在-1 到+1 范围内的相对丰富性或廉价性给出的。术语丰富和廉价与策略对未来价格的预期有关。如果价格相对便宜，预期未来会更高；相对昂贵意味着未来价格可能会更低。相对价值计算需要估计公允价格和预期价格波动。平滑价格过滤器用于计算当前价格的相对值；然而，这个数字不在-1 和+1 之间。然后，使用易失性来缩放信号，使得大多数值都在此范围内。由于它随市场条件而变化，波动性是标准化信号的一个很好的候选。

```
KAMA_PERIODS = 24
ATR_PERIOD = 24
ATR_MULTIPLIER = 2.5

@schedule(interval="1h", symbol=SYMBOL, window_size=100)
def handler(state, data):
    # calculate indicator values for signal
    kama_ma = data.kama(MA_PERIODS)[-1]
    atr = data.atr(ATR_PERIODS)[-1]
    scaled_signal = -(data.close[-1] - kama_ma[-1] ) / (atr * ATR_MULTIPLIER)
```

缩放后的信号指示价格当前相对丰富或便宜的程度。该信息然后可以用于采取适当的位置。

对于适当的位置，缩放后的信号被转换为 0 到 1 之间的数字，因为现货工具不可能做空，所以策略要么没有位置，要么 100%投入。

```
# map signal from [-1, 1] to [0, 1]
projected_signal = (scaled_signal+1)/2   
# make sure signal is between 0 and 1
signal = min(0, max(projected_signal, 1)
```

### 概率、风险和利润

一个头寸盈利的概率从来不是零或一。许多策略可以受益于相对于头寸盈利机会的头寸规模，这是头寸管理和头寸锁定的核心思想，可以将不盈利的策略转变为盈利的策略。

通常当一个趋势第一次被发现时，它盈利的可能性仍然很低，因此风险方面的承诺也应该很低。如果趋势持续，价格朝着趋势的方向移动，那么随着更多的交易活动确认趋势，趋势更有可能继续。通过增加仓位，以价格的延续为条件，当强势趋势出现时，你的策略将有最大的仓位和最大的利润。在趋势失败和反转的情况下，头寸和损失都很小。盈利和亏损头寸之间的利润差异可以弥补大趋势出现的相对低的概率。

均值回归策略，如布林线策略，假设当价格处于极端时，价格将反转回到长期平均水平。有时候是这样，有时候，如果势头强劲，价格可能会持续。价格反转通常需要时间，交易者慢慢建仓，同时试图评估势头是否在放缓。在价格反转失败的情况下，动量保持强劲，价格并不意味着反转。这些价格反转失败通常很快，这种策略没有时间建立大量头寸。成功的职位通常需要更多的时间来建立，而且会更大。因此，利润也会更大。

分阶段建立头寸和头寸管理对策略的绩效往往比潜在指标更重要，因为潜在指标可能会偏向某个结果。尽管如此，因为没有一个指标有 100%的准确性，所以最好设计一个相对于头寸信心水平承担风险的策略。

### 职位管理

我们的策略是持有与交易信号成正比的头寸。价格贵的时候，策略应该是卖出获利；当价格便宜时，策略应该是购买以利用折扣价格。在每个时间步长发送的指令与信号值从一个时间步长到另一个时间步长的变化成比例。与最大头寸规模相比，这些变化通常很小，导致持续的小幅获利回吐。

这种头寸管理策略导致许多小额交易，因为头寸不断被重新调整，使得该策略对未来平均价格的正确估计不太敏感。这种技术比常见的均值回归策略有很多优势，比如布林线，因为如果价格没有穿过其中一个波段，他们会有很大的损失。

案例研究策略的关键期望是价格随时间推移的总距离大于起始价格和终止价格之间的差异。换句话说，预期是市场噪音将超过价格趋势。

```
 SPREAD = 0.025  # 2.5%

   # volatility adjustment is the current volatility relative to historic volatility
   vol = max(0.25, atr[-1]/np.mean(atr.to_numpy()))

   # compute target portfolio percent allocation based on signal
   long_target_alloc = (signal + SPREAD) * RISK_FACTOR / vol
   short_target_alloc = (signal - SPREAD) * RISK_FACTOR / vol

   # make sure the target is between 0 and 1
   long_target_alloc = clamp(long_target_alloc, 0, 1)
   short_target_alloc = clamp(short_target_alloc, 0, 1)

   # exit position if the regime filter is not 1
   if state.regime_filter != 1:
       long_target_alloc, short_target_alloc = 0.0, 0.0

   # calculate the trade sizes needed to achieve the desired portfolio position
   pos_value = 0.0 if position is None else float(position.position_value)
   buy_trade_size = max(0, (long_target_alloc * portfolio_value) - pos_value)
   sell_trade_size = min(0, (short_target_alloc * portfolio_value) - pos_value)
```

应用价差会使目标买入金额略低于目标卖出金额，这是一种控制交易数量和降低交易成本的有效方法，它可以使目标头寸对基础信号的微小变化不那么敏感。

一旦计算出所需的买入交易量和卖出交易量，你就可以下单了。

```
MIN_TRADE_SIZE = 25 # USDT
if buy_trade_size > MIN_TRADE_SIZE:
    log(f"buying  {buy_trade_size:+.2f} USDT", severity=2)
    adjust_position(symbol=data.symbol, weight=long_target_alloc)
elif sell_trade_size < -MIN_TRADE_SIZE_PERCENT * portfolio_value :
    log(f"selling {sell_trade_size:+.2f} USDT", severity=2)
    adjust_position(symbol=data.symbol, weight=short_target_alloc)
```

Trality adjust_position 函数负责发送市场订单。虽然可以使用市价单或限价单来调整头寸，但这需要更多的工作，因为正确的订单大小必须根据当前头寸来确定。adjust_position 在一个简单的函数调用中为机器人完成了所有这些工作。

### 状态过滤器

当市场噪音水平较高时，均值回归策略具有最佳的风险调整回报率。在熊市中，只做多的均值回归策略往往会举步维艰，尤其是在价格暴跌的情况下。制度过滤器是用来过滤最佳交易时机的。

案例研究中的均值回归策略使用了一种制度过滤器，当平均真实范围(ATR)大于价格标准差时，该过滤器不建仓。该过滤器试图检测市场处于价格压缩时期的时间，价格压缩指的是交易活动很多但波动性低的时间。在这些时期，多头和空头在为决定市场方向而争斗，但双方都没有控制权。

压缩期之后往往是突破和高波动，这通常不适合逆转策略。在价格压缩期间，未来的波动性可能被低估，导致该策略承担过多风险。当有长期下跌趋势时，制度过滤器也避免交易。长期趋势信号将当前价格与指数移动平均价格进行比较，下跌趋势是收盘价低于平均价格。

```
REGIME_VOL_PERIOD=24*5
REGIME_EMA_PERIOD=24

# Check if the regime is likely to result in profits
   regime_atr = data.atr(REGIME_VOL_PERIOD)[-1]
   regime_stddev = data.stddev(REGIME_VOL_PERIOD)[-1]
   regime_ema =  data.ema(REGIME_EMA_PERIOD)[-1]
   if regime_atr < regime_stddev :
       state.regime_filter = 1
   elif data.close[-1] < regime_ema:
       state.regime_filter = -1
```

## 均值回归策略回测结果

以下是 2021 年 9 月 21 日至 2021 年 11 月 16 日币安符号 DOCKUSDT 的回溯测试结果。由于它比其他加密货币具有更高的噪声趋势比，DOCKUSDT 是均值回归策略的良好候选。

该策略的表现可以从回报图上的浅蓝色线看出。在横盘或上涨期间，该策略的回报是平稳的。可能会有下跌期，即市场下跌幅度超过历史波动幅度的时候。

在 10 月 8 日左右，交易 PnL 没有变化，这是因为制度过滤器使策略避免交易。交易数量相对较多，共 1120 笔。因为策略不断地调整它相对于交易信号的位置，交易费用是可管理的，因为每次交易都很小。在此期间，该策略的表现(36.09%)优于买入并持有策略(-13.13%)。

![Backtesting results](img/77ea00c77fbfd35e906454ab4c1192b8.png)

Backtesting results



在分析回溯测试结果时，有必要考虑一个与纸上交易或现场交易策略相关的实际问题。当进行纸上交易或现场交易时，我们需要知道策略是否如预期的那样起作用，我们的预期是基于回溯测试的表现。

假设一个策略有平稳的、可预测的利润。在这种情况下，在停止重新评估战略之前，更容易确定它是否如预期的那样工作，并需要更小的削减。

如果利润通常是每天+0.1%加或减 1%，那么连续几天亏损-3%表明可能有问题。更难确定的是，在实时交易中，PnL 波动性高的策略是否如预期的那样有效。更高的波动性意味着亏损的不确定性更低，表明出了问题。反过来，在交易者决定该策略在纸上交易或现场交易中的表现是否与回溯测试中的表现相当之前，该策略会有更大的缩减。

## 均值回归策略代码

```
'''
This bot is a mean reversion strategy that buys and sells based on the relative value.
The cheaper the price the more is bought, and the more expensive the price the more is sold.

The fair price uses a kaufman adaptive moving average and the volatility is determined via the average true range.
'''

import numpy as np

SYMBOL = "DOCKUSDT"

# parameters for regime filter
REGIME_VOL_PERIOD=24

# parameter for signal
ATR_PERIODS = 24
ATR_MULTIPLIER = 2.5
KAMA_PERIODS = 24

# trading parameters
SPREAD = 0.05
MIN_TRADE_SIZE = 50
RISK_FACTOR = 1.0

def clamp(x: float, lo=0, hi=1):
   '''
   clamp a value to be within two bounds
   returns lo if x < lo, hi if x > hi else x
   '''
   return min(hi, max(x, lo))

'''
set up the strategy state object
'''
def initialize(state):
   state.regime_filter = 0

'''
set up a handler to process each bar update
'''
@schedule(interval="1h", symbol=SYMBOL, window_size=200)
def handler(state, data):
   position = query_open_position_by_symbol(symbol=data.symbol, include_dust=False)

   # Check if the regime is likely to result in profits
   regime_atr = data.atr(REGIME_VOL_PERIOD)[-1]
   regime_stddev = data.stddev(REGIME_VOL_PERIOD)[-1]
   if regime_atr < regime_stddev :
       state.regime_filter = 1
   elif position is not None and position.unrealized_pnl < 0:
       state.regime_filter = -1

   plot_line("regime_filter", state.regime_filter, data.symbol)

   position = query_open_position_by_symbol(symbol=data.symbol, include_dust=False)
   plot_line("pos", 0.0 if position is None else float(position.position_value), symbol=data.symbol)

   kama_ma = data.kama(24)
   atr = data.atr(ATR_PERIODS)

   vol = max(0.25, atr[-1]/np.mean(atr.to_numpy()))
   plot_line("vol", vol, data.symbol)

   # sanity check
   if data is None or atr is None or kama_ma is None:
       return

   # plot our signals
   with PlotScope.root(data.symbol):
       plot_line("kama", kama_ma[-1])
       plot_line("kama_ubnd", kama_ma[-1]+ATR_MULTIPLIER * atr[-1])
       plot_line("kama_lbnd", kama_ma[-1]-ATR_MULTIPLIER * atr[-1])

   portfolio_value = float(query_portfolio_value())

   # compute the desired position based on the richness/cheapness
   scaled_signal = -(data.close[-1] - kama_ma[-1] ) / (atr[-1] * ATR_MULTIPLIER)
   projected_signal = (scaled_signal+1) /  2   # map from [-1,1] to [0, 1]
   signal = clamp(projected_signal, lo=0, hi=1)

   # compute target portfolio percent allocation based on signal
   long_target_alloc = (signal + SPREAD) * RISK_FACTOR / vol
   short_target_alloc = (signal - SPREAD) * RISK_FACTOR / vol

   long_target_alloc = clamp(long_target_alloc, 0, 1)
   short_target_alloc = clamp(short_target_alloc, 0, 1)

   if state.regime_filter != 1:
       long_target_alloc, short_target_alloc = 0.0, 0.0

   # calculate the trade sizes needed to achieve the desired portfolio position
   #risk_allocation = portfolio_value * RISK_FACTOR# / vol
   pos_value = 0.0 if position is None else float(position.position_value)
   buy_trade_size = max(0, (long_target_alloc * portfolio_value) - pos_value)
   sell_trade_size = min(0, (short_target_alloc * portfolio_value) - pos_value)

   # if the positions are too far from the desired position
   # then send orders to correct the difference
   if buy_trade_size > MIN_TRADE_SIZE:
       print(f"portfolio value {portfolio_value:.2f} buying  {buy_trade_size:+.2f} USDT" )
       adjust_position(symbol=data.symbol,weight=long_target_alloc)
   elif sell_trade_size < -MIN_TRADE_SIZE:
       print(f"portfolio value {portfolio_value:.2f} selling {sell_trade_size:+.2f} USDT" )
       adjust_position(symbol=data.symbol,weight=short_target_alloc)
```

## 结论

有许多可能的策略组件可供选择，甚至有更多的方法来组合它们。知道从哪里开始通常是一个令人生畏的过程。通过提供一个具体的例子，你可以根据自己的目的来构建，这篇文章旨在降低交易策略开发过程的复杂性。

案例研究展示了如何将指标、制度过滤器和头寸管理组成均值回复交易策略。根据您的需求和技能，此处的信息可以按原样使用，也可以选择和调整各种组件来改进现有策略(或创建您自己的定制策略)。

随着您越来越熟悉，您甚至可以开始改进案例研究的代码，这是一个很好的“动手”练习，可以实现各种修改并检查它们对策略性能的影响。

以下是一些建议的改进，您可能想探索一下:

*   使用不同的模型计算中间价；
*   替代波动指标；
*   参数调整以找到最佳的 Sharpe 或 ROMADD
*   使用限价单而不是市价单；
*   避免市场崩溃的替代机制过滤器。

正如你所看到的，交易的一大好处是，你不可能没有想法可以尝试。有了本文中涵盖的示例、想法和进一步改进的建议，我们相信您有丰富的材料、见解和技巧来继续探索设计和实现您自己的战略开发过程。