# Building a Regime-adaptive Strategy for Freqtrade

**Author:** Betha Morrison  
**Date:** June 2026  
**Tags:** freqtrade, quant, crypto, strategy, market-regime, python

---

Bitcoin closed June 4 at $64,350, down 18.3% from its 30-day high of $78,436.
Total crypto market cap sits at $2.18T - 48% below the $4.2T peak. The Fear
& Greed Index is at 12 (Extreme Fear). $3.53B in liquidations over 30 days,
the single largest event being a $330M cascade on June 2. Bitcoin dominance at
60%. Altcoin Season Index at 30 - firmly in Bitcoin Season territory. ETF
outflows: $2.97B since May 15.

This is the market a strategy must survive in June 2026.

The core problem with most publicly available Freqtrade strategies is that they
were tuned for a single regime - usually the trending upmarket of late 2024 to
early 2025. A regime shift of this magnitude breaks them systematically.
The solution is not better parameters. It is a strategy that detects which
regime it is operating in before generating any signal.

---

## Why Single-regime Strategies Fail Here

**EMA crossover (pure trend-following).** In a trending market EMA crossovers
work - price respects the faster EMA as support on pullbacks. In a liquidation
cascade, price gaps through multiple EMA levels in a single candle. The signal
fires after the move, the entry is at the bottom of the cascade and the next
leg down produces an immediate stoploss hit. The whipsaw rate in a
high-volatility downtrend makes this approach structurally unprofitable.

**Pure RSI mean-reversion.** RSI < 30 in a ranging market correctly identifies
oversold conditions. RSI < 30 in a sustained downtrend identifies the first
leg of a multi-leg decline. The instrument can remain oversold for weeks.
Without a regime gate, mean-reversion signals in the current June 2026 tape
are knife-catching.

**Grid bots.** Grid strategies require price to oscillate within a defined
range. A 48% drawdown from peak exits the grid range entirely. The bot keeps
buying at predefined levels while price continues falling, resulting in a fully
deployed position at a deep loss with no exit mechanism. Grid bots are
structurally unsuitable for any market with directional trend risk above a
certain magnitude.

**MACD momentum.** MACD is a lagging indicator by construction - it is the
difference between two EMAs. In a fast-moving market, the signal line crossover
arrives after the move has already delivered most of its return. It also
generates frequent false signals in ranging conditions. In a regime that
alternates between choppy consolidation and sharp sell-offs (current June 2026
structure), MACD produces entries at the wrong point in the cycle
systematically.

**ML-based strategies.** A model trained on 2024–2025 data learned a market
structure that no longer exists. The current distribution of price returns,
volatility and volume is outside the training distribution. The model's
predictions are extrapolation, not inference. Beyond the distribution shift
problem, ML strategies require retraining pipelines, labeled data and
validation infrastructure that most Freqtrade deployments do not have.
Complexity without a maintenance protocol is a liability.

---

## Approach: Market Regime Detection First, Signal Second

The strategy does not generate buy/sell signals unconditionally. It first
classifies the current regime, then applies the appropriate signal logic for
that regime - or generates no signal at all if the regime is ambiguous or
unfavorable.

**Regime classification uses three inputs:**

`ADX(14)` - measures trend strength without direction. ADX > 25 indicates a
trending market. ADX < 20 indicates a ranging market. ADX is direction-agnostic:
it rises equally in sustained uptrends and sustained downtrends.

`EMA(21) / EMA(55)` - determines trend direction when ADX confirms trend
presence. EMA_21 > EMA_55 is bullish structure. EMA_21 < EMA_55 is bearish
structure.

`Bollinger Band Width (BBW)` - measures volatility expansion. BBW is
`(upper - lower) / middle`. When BBW exceeds 2x its 20-period average, the
market is in a volatility expansion phase - typically associated with
liquidation cascades or breakout moves. This condition restricts new long
entries regardless of other signals.

**Four regimes:**

| regime | ADX | EMA structure | BBW | action |
|---|---|---|---|---|
| `BULL_TREND` | ≥ 25 | 21 > 55 | normal | EMA pullback long |
| `BEAR_TREND` | ≥ 25 | 21 < 55 | normal | no new longs |
| `RANGE` | < 20 | any | normal | RSI + BB mean reversion |
| `EXPANSION` | any | any | > 2x avg | no new longs |

In June 2026 the market is primarily in `BEAR_TREND` or `EXPANSION`. The
strategy's answer to that is: do nothing. Not every day needs a trade.

---

## Signal Logic per Regime

**`BULL_TREND` - EMA pullback entry.**

Price must be above EMA_55 (trend intact), have pulled back to within 1×ATR of
EMA_21 and RSI must be between 40 and 60 (not overbought, not crashed). This
targets the pullback-to-continuation pattern in a defined trend rather than
breakout entries, which have worse risk/reward due to extension from mean.

**`RANGE` - RSI + Bollinger Band lower-band mean reversion.**

RSI < 35 and price within 0.5% of the lower Bollinger Band (period 20,
stddev 2.0). Volume on the entry candle must be ≥ 1.5× the 20-period average
volume - low-volume bounces off the lower band are unreliable. Exit target:
EMA_21 or upper band, whichever is closer. This is a defined-risk, defined-target
trade, not an open-ended trend ride.

---

## Implementation

```python
# regime_adaptive.py
# freqtrade strategy - market regime adaptive
# requires: pandas-ta

from freqtrade.strategy import IStrategy, DecimalParameter, IntParameter
from pandas import DataFrame
import pandas_ta as ta

class RegimeAdaptive(IStrategy):

    INTERFACE_VERSION = 3
    timeframe = "1h"
    use_custom_stoploss = True

    # ROI: exit faster in range regime, hold longer in trend
    minimal_roi = {
        "0":   0.06,
        "60":  0.04,
        "120": 0.02,
        "240": 0.01,
    }

    stoploss = -0.05   # hard floor; custom stoploss takes over dynamically

    # hyperopt search space - kept narrow intentionally
    adx_trend_threshold   = IntParameter(22, 30, default=25, space="buy")
    adx_range_threshold   = IntParameter(15, 22, default=20, space="buy")
    rsi_range_oversold    = IntParameter(28, 38, default=35, space="buy")
    bbw_expansion_factor  = DecimalParameter(1.5, 2.5, default=2.0, space="buy")
    atr_stoploss_mult     = DecimalParameter(1.5, 3.0, default=2.0, space="sell")

    def informative_pairs(self):
        return []

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        # trend direction
        dataframe["ema_21"] = ta.ema(dataframe["close"], length=21)
        dataframe["ema_55"] = ta.ema(dataframe["close"], length=55)

        # trend strength
        adx_data = ta.adx(dataframe["high"], dataframe["low"],
                          dataframe["close"], length=14)
        dataframe["adx"] = adx_data["ADX_14"]

        # mean reversion
        dataframe["rsi"] = ta.rsi(dataframe["close"], length=14)
        bb = ta.bbands(dataframe["close"], length=20, std=2.0)
        dataframe["bb_lower"]  = bb["BBL_20_2.0"]
        dataframe["bb_upper"]  = bb["BBU_20_2.0"]
        dataframe["bb_middle"] = bb["BBM_20_2.0"]

        # bollinger band width - volatility expansion detector
        dataframe["bbw"] = (
            (dataframe["bb_upper"] - dataframe["bb_lower"])
            / dataframe["bb_middle"]
        )
        dataframe["bbw_avg"] = dataframe["bbw"].rolling(20).mean()

        # atr - for dynamic stoploss and pullback proximity
        dataframe["atr"] = ta.atr(dataframe["high"], dataframe["low"],
                                   dataframe["close"], length=14)

        # volume filter
        dataframe["volume_avg"] = dataframe["volume"].rolling(20).mean()

        # regime classification
        adx_trend = self.adx_trend_threshold.value
        adx_range = self.adx_range_threshold.value
        bbw_factor = self.bbw_expansion_factor.value

        dataframe["regime_expansion"] = (
            dataframe["bbw"] > bbw_factor * dataframe["bbw_avg"]
        )
        dataframe["regime_bull_trend"] = (
            (dataframe["adx"] >= adx_trend)
            & (dataframe["ema_21"] > dataframe["ema_55"])
            & ~dataframe["regime_expansion"]
        )
        dataframe["regime_bear_trend"] = (
            (dataframe["adx"] >= adx_trend)
            & (dataframe["ema_21"] < dataframe["ema_55"])
        )
        dataframe["regime_range"] = (
            (dataframe["adx"] < adx_range)
            & ~dataframe["regime_expansion"]
        )

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        rsi_os = self.rsi_range_oversold.value

        # BULL_TREND: pullback to EMA_21 entry
        bull_entry = (
            dataframe["regime_bull_trend"]
            & (dataframe["close"] > dataframe["ema_55"])
            & (dataframe["close"] <= dataframe["ema_21"]
               + dataframe["atr"])
            & (dataframe["close"] >= dataframe["ema_21"]
               - dataframe["atr"])
            & dataframe["rsi"].between(40, 60)
        )

        # RANGE: lower-band RSI mean reversion
        range_entry = (
            dataframe["regime_range"]
            & (dataframe["rsi"] < rsi_os)
            & (dataframe["close"] <= dataframe["bb_lower"] * 1.005)
            & (dataframe["volume"] >= 1.5 * dataframe["volume_avg"])
        )

        dataframe.loc[bull_entry | range_entry, "enter_long"] = 1
        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        # exit long when trend flips or regime becomes hostile
        dataframe.loc[
            dataframe["regime_bear_trend"] | dataframe["regime_expansion"],
            "exit_long"
        ] = 1

        # range exit: price reaches EMA_21
        dataframe.loc[
            dataframe["regime_range"]
            & (dataframe["close"] >= dataframe["ema_21"]),
            "exit_long"
        ] = 1

        return dataframe

    def custom_stoploss(self, current_time, current_rate, current_profit,
                        min_profit, trade, **kwargs) -> float:
        dataframe, _ = self.dp.get_analyzed_dataframe(
            trade.pair, self.timeframe
        )
        if dataframe.empty:
            return self.stoploss

        last = dataframe.iloc[-1]
        atr  = last["atr"]

        if atr > 0 and current_rate > 0:
            # dynamic stoploss: N × ATR below current price
            mult = self.atr_stoploss_mult.value
            sl   = -(mult * atr / current_rate)
            # never widen beyond the hard floor
            return max(sl, self.stoploss)

        return self.stoploss
```

---

## Why ATR-based Dynamic Stoploss, Not Fixed Percentage

A fixed stoploss of -5% is calibrated for a specific volatility environment.
In June 2026, BTC 30-day realized volatility is elevated - a 5% intraday swing
is normal. A fixed stoploss at -5% in this environment triggers on normal price
noise rather than on an actual trade invalidation. ATR normalizes the stoploss
to the current volatility: when ATR is large, the stoploss is wider; when ATR
compresses (range regime), the stoploss tightens automatically. The stoploss
moves with the market structure, not against it.

`atr_stoploss_mult` is left as a hyperopt parameter (range 1.5–3.0) because the
optimal multiplier is pair-dependent. BTC/USDT on 1h with ATR typically around
1.5–2.0% benefits from a multiplier of 2.0–2.5. More volatile altcoin pairs
require a wider multiplier to avoid constant stopout on noise.

---

## Hyperopt and Walk-forward

Hyperopt on in-sample data without out-of-sample validation produces
overfit parameters. The correct process:

```bash
# backtest period: Jan 2025 – Mar 2026 (in-sample)
freqtrade hyperopt \
  --strategy RegimeAdaptive \
  --hyperopt-loss SharpeHyperOptLoss \
  --timerange 20250101-20260301 \
  --pairs BTC/USDT ETH/USDT \
  --epochs 300

# walk-forward validation: Apr – Jun 2026 (out-of-sample)
freqtrade backtesting \
  --strategy RegimeAdaptive \
  --timerange 20260401-20260607 \
  --pairs BTC/USDT ETH/USDT
```

Sharpe ratio as hyperopt loss function rather than total profit. Maximizing
profit in hyperopt incentivizes the optimizer to find parameters that take
maximum risk in the most favorable period of the in-sample data. Sharpe penalizes
drawdown relative to return, producing parameters that are more likely to
survive regime shifts.

If out-of-sample Sharpe drops more than 40% relative to in-sample Sharpe,
the strategy is overfit to the training period. Do not deploy.

---

## What this Strategy Does

Given current conditions - `BEAR_TREND` or `EXPANSION` on BTC/USDT 1h - the
strategy generates no long entries. That is the correct output. A strategy that
sits on cash during a 48% drawdown is not underperforming. It is functioning.

The regime gates will re-engage for longs when:

- ADX drops below 20 and BBW normalizes → `RANGE` regime, mean-reversion entries
  permitted on oversold bounces
- Or ADX re-establishes above 25 with EMA_21 crossing back above EMA_55 →
  `BULL_TREND` regime, pullback entries permitted

Neither condition is present in the first week of June 2026. The bot waits.

---

*- Betha Morrison*
