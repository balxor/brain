# Building A Regime-adaptive Strategy for Freqtrade

**Author:** Betha Morrison  
**Date:** June 2026  
**Tags:** freqtrade, quant, crypto, strategy, market-regime, python

---

Bitcoin closed June 4 at $64,350, down 18.3% from its 30-day high of $78,436.
Total crypto market cap sits at $2.18T — 48% below the $4.2T peak. The Fear
& Greed Index is at 12 (Extreme Fear). $3.53B in liquidations over 30 days,
the single largest event being a $330M cascade on June 2. Bitcoin dominance at
60%. Altcoin Season Index at 30 — firmly in Bitcoin Season territory. ETF
outflows: $2.97B since May 15.

This is the market a strategy must survive in June 2026.

The core problem with most publicly available Freqtrade strategies is that they
were tuned for a single regime — usually the trending upmarket of late 2024 to
early 2025. A regime shift of this magnitude breaks them systematically.
The solution is not better parameters. It is a strategy that detects which
regime it is operating in before generating any signal.

---

## why single-regime strategies fail here

**EMA crossover (pure trend-following).** In a trending market EMA crossovers
work — price respects the faster EMA as support on pullbacks. In a liquidation
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

**MACD momentum.** MACD is a lagging indicator by construction — it is the
difference between two EMAs. In a fast-moving market, the signal line crossover
arrives after the move has already delivered most of its return. It also
generates frequent false signals in ranging conditions. In a regime that
alternates between choppy consolidation and sharp sell-offs (current June 2026
structure), MACD produces entries at the wrong point in the cycle systematically.

**ML-based strategies.** A model trained on 2024–2025 data learned a market
structure that no longer exists. The current distribution of price returns,
volatility and volume is outside the training distribution. The model's
predictions are extrapolation, not inference. Beyond the distribution shift
problem, ML strategies require retraining pipelines, labeled data and
validation infrastructure that most Freqtrade deployments do not have.
Complexity without a maintenance protocol is a liability.

---

## approach: market regime detection first, signal second

The strategy does not generate buy/sell signals unconditionally. It first
classifies the current regime, then applies the appropriate signal logic for
that regime — or generates no signal at all if the regime is ambiguous or
unfavorable.

**Regime classification uses three inputs:**

`ADX(14)` — measures trend strength without direction. ADX > 25 indicates a
trending market. ADX < 20 indicates a ranging market. ADX is
direction-agnostic: it rises equally in sustained uptrends and sustained
downtrends.

`EMA(21) / EMA(55)` — determines trend direction when ADX confirms trend
presence. EMA_21 > EMA_55 is bullish structure. EMA_21 < EMA_55 is bearish
structure.

`Bollinger Band Width (BBW)` — measures volatility expansion. BBW is
`(upper - lower) / middle`. When BBW exceeds 2× its 20-period average, the
market is in a volatility expansion phase — typically associated with
liquidation cascades or breakout moves. This condition restricts new long
entries regardless of other signals.

**Four regimes:**

| regime | ADX | EMA structure | BBW | action |
|---|---|---|---|---|
| `BULL_TREND` | ≥ 25 | 21 > 55 | normal | EMA pullback long |
| `BEAR_TREND` | ≥ 25 | 21 < 55 | normal | no new longs |
| `RANGE` | < 20 | any | normal | RSI + BB mean reversion |
| `EXPANSION` | any | any | > 2× avg | no new longs |

In June 2026 the market is primarily in `BEAR_TREND` or `EXPANSION`. The
strategy's answer to that is: do nothing. Not every day needs a trade.

---

## signal logic per regime

**`BULL_TREND` — EMA pullback entry.**

Price must be above EMA_55 (trend intact), have pulled back to within 1×ATR of
EMA_21, and RSI must be between 40 and 62 (not overbought, not crashed). This
targets the pullback-to-continuation pattern in a defined trend rather than
breakout entries, which have worse risk/reward due to extension from mean.

**`RANGE` — Bollinger Band lower-band reclaim with RSI recovery.**

The previous candle must have touched or breached the lower band; the current
candle must close back above it. RSI must be below the oversold threshold and
rising. A green candle is required — no entry into a still-falling candle.
Volume must be ≥ 1.2× the 20-period average. Exit target: EMA_21. This is a
defined-risk, defined-target trade, not an open-ended trend ride.

---

## Implementation

The strategy has no external indicator library dependency. All indicators are
implemented directly using NumPy and pandas with Wilder's smoothing where
applicable, giving full reproducibility across environments.

`startup_candle_count = 120` ensures EMA55 and ADX have fully converged before
the first signal evaluation. Without this, Freqtrade evaluates signals on
startup candles where indicators are numerically unstable.

`stoploss_from_absolute()` converts an absolute price level to the ratio format
Freqtrade requires, correctly accounting for leverage. A manual ratio
calculation without this helper produces incorrect stoploss placement in
futures mode.

```python
"""
RegimeAdaptive

Freqtrade strategy for 1h market-regime adaptive long trading.

Design goals:
- No external indicator library dependency.
- Long-only. Conservative leverage (1x default).
- Bull trend entries use EMA pullback logic.
- Range entries use Bollinger Band reclaim + RSI recovery.
- Expansion and bear regimes are treated defensively.
- ATR-based custom stoploss via stoploss_from_absolute.

Recommended use:
- Backtest first. Dry-run before live.
- Do not run live without sufficient backtest and dry-run evidence.
"""

from datetime import datetime
from typing import Optional

import numpy as np
import pandas as pd
from pandas import DataFrame, Series

from freqtrade.persistence import Trade
from freqtrade.strategy import DecimalParameter, IntParameter, IStrategy, stoploss_from_absolute


class RegimeAdaptive(IStrategy):

    INTERFACE_VERSION = 3

    timeframe = "1h"
    startup_candle_count = 120

    can_short = False

    process_only_new_candles = True
    use_exit_signal = True
    exit_profit_only = False
    ignore_roi_if_entry_signal = False
    use_custom_stoploss = True

    minimal_roi = {
        "0":   0.06,
        "60":  0.04,
        "120": 0.02,
        "240": 0.01,
    }

    # Hard floor. Custom stoploss does not widen beyond this.
    stoploss = -0.05

    # Hyperopt search space — intentionally narrow.
    adx_trend_threshold  = IntParameter(22, 30, default=25, space="buy",  optimize=True)
    adx_range_threshold  = IntParameter(15, 22, default=20, space="buy",  optimize=True)
    rsi_range_oversold   = IntParameter(28, 38, default=35, space="buy",  optimize=True)
    bbw_expansion_factor = DecimalParameter(1.5, 2.5, default=2.0, space="buy",  optimize=True)
    atr_stoploss_mult    = DecimalParameter(1.5, 3.0, default=2.0, space="sell", optimize=True)

    @property
    def protections(self):
        return [
            {
                "method": "CooldownPeriod",
                "stop_duration_candles": 2,
            },
            {
                "method": "StoplossGuard",
                "lookback_period_candles": 24,
                "trade_limit": 2,
                "stop_duration_candles": 4,
                "only_per_pair": False,
            },
            {
                "method": "MaxDrawdown",
                "calculation_mode": "equity",
                "lookback_period_candles": 48,
                "trade_limit": 2,
                "stop_duration_candles": 12,
                "max_allowed_drawdown": 0.08,
            },
        ]

    # ── indicator implementations ─────────────────────────────────────────────

    @staticmethod
    def _ema(series: Series, length: int) -> Series:
        return series.ewm(span=length, adjust=False, min_periods=length).mean()

    @staticmethod
    def _rsi(series: Series, length: int = 14) -> Series:
        delta    = series.diff()
        gain     = delta.clip(lower=0)
        loss     = -delta.clip(upper=0)
        avg_gain = gain.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()
        avg_loss = loss.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()
        rs       = avg_gain / avg_loss.replace(0, np.nan)
        return (100 - (100 / (1 + rs))).fillna(50)

    @staticmethod
    def _atr(dataframe: DataFrame, length: int = 14) -> Series:
        high       = dataframe["high"]
        low        = dataframe["low"]
        close      = dataframe["close"]
        prev_close = close.shift(1)
        true_range = pd.concat(
            [high - low, (high - prev_close).abs(), (low - prev_close).abs()],
            axis=1,
        ).max(axis=1)
        return true_range.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()

    @staticmethod
    def _adx(dataframe: DataFrame, length: int = 14) -> Series:
        """Wilder-style ADX."""
        high  = dataframe["high"]
        low   = dataframe["low"]
        close = dataframe["close"]

        up_move   = high.diff()
        down_move = -low.diff()

        plus_dm  = Series(
            np.where((up_move > down_move) & (up_move > 0), up_move, 0.0),
            index=dataframe.index,
        )
        minus_dm = Series(
            np.where((down_move > up_move) & (down_move > 0), down_move, 0.0),
            index=dataframe.index,
        )

        prev_close = close.shift(1)
        true_range = pd.concat(
            [high - low, (high - prev_close).abs(), (low - prev_close).abs()],
            axis=1,
        ).max(axis=1)

        atr      = true_range.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()
        plus_di  = 100 * plus_dm.ewm( alpha=1 / length, adjust=False, min_periods=length).mean() / atr.replace(0, np.nan)
        minus_di = 100 * minus_dm.ewm(alpha=1 / length, adjust=False, min_periods=length).mean() / atr.replace(0, np.nan)
        dx       = 100 * (plus_di - minus_di).abs() / (plus_di + minus_di).replace(0, np.nan)
        return dx.ewm(alpha=1 / length, adjust=False, min_periods=length).mean().fillna(0)

    @staticmethod
    def _bollinger_bands(
        series: Series, length: int = 20, std: float = 2.0
    ) -> tuple[Series, Series, Series]:
        middle    = series.rolling(length, min_periods=length).mean()
        deviation = series.rolling(length, min_periods=length).std(ddof=0)
        return middle - std * deviation, middle, middle + std * deviation

    # ── freqtrade interface ───────────────────────────────────────────────────

    def informative_pairs(self):
        return []

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe["ema_21"] = self._ema(dataframe["close"], 21)
        dataframe["ema_55"] = self._ema(dataframe["close"], 55)
        dataframe["adx"]    = self._adx(dataframe, 14)
        dataframe["rsi"]    = self._rsi(dataframe["close"], 14)

        bb_lower, bb_middle, bb_upper = self._bollinger_bands(dataframe["close"], 20, 2.0)
        dataframe["bb_lower"]  = bb_lower
        dataframe["bb_middle"] = bb_middle
        dataframe["bb_upper"]  = bb_upper

        dataframe["bbw"] = (
            (dataframe["bb_upper"] - dataframe["bb_lower"])
            / dataframe["bb_middle"].replace(0, np.nan)
        ).replace([np.inf, -np.inf], np.nan)
        dataframe["bbw_avg"] = dataframe["bbw"].rolling(20, min_periods=20).mean()

        dataframe["atr"]        = self._atr(dataframe, 14)
        dataframe["volume_avg"] = dataframe["volume"].rolling(20, min_periods=20).mean()

        # candle-level helpers used in entry conditions
        dataframe["green_candle"] = dataframe["close"] > dataframe["open"]
        dataframe["rsi_rising"]   = dataframe["rsi"] > dataframe["rsi"].shift(1)

        # regime classification
        adx_trend  = self.adx_trend_threshold.value
        adx_range  = self.adx_range_threshold.value
        bbw_factor = self.bbw_expansion_factor.value

        dataframe["regime_expansion"] = (
            dataframe["bbw_avg"].notna()
            & dataframe["bbw"].notna()
            & (dataframe["bbw"] > bbw_factor * dataframe["bbw_avg"])
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
        dataframe.loc[:, "enter_long"] = 0
        dataframe.loc[:, "enter_tag"]  = None

        rsi_os = self.rsi_range_oversold.value

        # BULL_TREND: pullback to EMA21 while structurally above EMA55.
        bull_entry = (
            dataframe["regime_bull_trend"]
            & (dataframe["close"] > dataframe["ema_55"])
            & (dataframe["close"] <= dataframe["ema_21"] + dataframe["atr"])
            & (dataframe["close"] >= dataframe["ema_21"] - dataframe["atr"])
            & dataframe["rsi"].between(40, 62)
            & (dataframe["volume"] > 0)
        )

        # RANGE: lower-band reclaim — previous candle at/below lower band,
        # current candle closes back above it. RSI rising, green candle,
        # volume confirmation. Avoids entry into a still-falling candle.
        lower_band_reclaim = (
            (dataframe["close"].shift(1) <= dataframe["bb_lower"].shift(1) * 1.005)
            & (dataframe["close"] > dataframe["bb_lower"])
        )
        range_entry = (
            dataframe["regime_range"]
            & (dataframe["rsi"] < rsi_os)
            & dataframe["rsi_rising"]
            & dataframe["green_candle"]
            & lower_band_reclaim
            & (dataframe["volume"] >= 1.2 * dataframe["volume_avg"])
        )

        dataframe.loc[bull_entry,   ["enter_long", "enter_tag"]] = (1, "bull_ema21_pullback")
        dataframe.loc[range_entry,  ["enter_long", "enter_tag"]] = (1, "range_bb_reclaim_rsi")

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[:, "exit_long"] = 0
        dataframe.loc[:, "exit_tag"]  = None

        hostile_regime     = dataframe["regime_bear_trend"] | dataframe["regime_expansion"]
        range_mean_revert  = dataframe["regime_range"] & (dataframe["close"] >= dataframe["ema_21"])

        dataframe.loc[hostile_regime,    ["exit_long", "exit_tag"]] = (1, "hostile_regime")
        dataframe.loc[range_mean_revert, ["exit_long", "exit_tag"]] = (1, "range_mean_reversion")

        return dataframe

    def custom_stoploss(
        self,
        pair: str,
        trade: Trade,
        current_time: datetime,
        current_rate: float,
        current_profit: float,
        after_fill: bool = False,
        **kwargs,
    ) -> Optional[float]:
        """
        ATR-based dynamic stoploss using stoploss_from_absolute.

        Converts N×ATR below current price into the ratio format Freqtrade
        requires, correctly accounting for leverage. Returns None when data is
        unavailable, leaving the current stop unchanged.
        """
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        if dataframe is None or dataframe.empty:
            return None

        last = dataframe.iloc[-1]
        atr  = last["atr"]

        if pd.isna(atr) or atr <= 0 or current_rate <= 0:
            return None

        mult = float(self.atr_stoploss_mult.value)

        if trade.is_short:
            stop_price = min(current_rate + mult * atr, current_rate * (1 - self.stoploss))
        else:
            stop_price = max(current_rate - mult * atr, current_rate * (1 + self.stoploss))

        return stoploss_from_absolute(
            stop_price,
            current_rate=current_rate,
            is_short=trade.is_short,
            leverage=trade.leverage,
        )

    def leverage(
        self,
        pair: str,
        current_time: datetime,
        current_rate: float,
        proposed_leverage: float,
        max_leverage: float,
        entry_tag: Optional[str],
        side: str,
        **kwargs,
    ) -> float:
        return min(1.0, max_leverage)
```

---

## why ATR-based dynamic stoploss, not fixed percentage

A fixed stoploss of -5% is calibrated for a specific volatility environment.
In June 2026, BTC 30-day realized volatility is elevated — a 5% intraday swing
is normal. A fixed stoploss at -5% in this environment triggers on normal price
noise rather than on an actual trade invalidation. ATR normalizes the stoploss
to current volatility: when ATR is large the stoploss is wider; when ATR
compresses (range regime) it tightens automatically.

`atr_stoploss_mult` is left as a hyperopt parameter (range 1.5–3.0) because the
optimal multiplier is pair-dependent. BTC/USDT on 1h with ATR typically around
1.5–2.0% benefits from a multiplier of 2.0–2.5. More volatile pairs require a
wider multiplier to avoid stopout on noise.

---

## Hyperopt and Walk-forward

Hyperopt on in-sample data without out-of-sample validation produces overfit
parameters. The correct process:

```bash
# in-sample: Jan 2025 – Mar 2026
freqtrade hyperopt \
  --strategy RegimeAdaptive \
  --hyperopt-loss SharpeHyperOptLoss \
  --timerange 20250101-20260301 \
  --pairs BTC/USDT ETH/USDT \
  --epochs 300

# out-of-sample validation: Apr – Jun 2026
freqtrade backtesting \
  --strategy RegimeAdaptive \
  --timerange 20260401-20260607 \
  --pairs BTC/USDT ETH/USDT
```

Sharpe ratio as hyperopt loss rather than total profit. Maximizing profit
incentivizes the optimizer to take maximum risk in the most favorable period of
the training data. Sharpe penalizes drawdown relative to return, producing
parameters more likely to survive regime shifts.

If out-of-sample Sharpe drops more than 40% relative to in-sample Sharpe,
the parameters are overfit. Do not deploy.

---

## What This Strategy Does

Given current conditions — `BEAR_TREND` or `EXPANSION` on BTC/USDT 1h — the
strategy generates no long entries. That is the correct output. A strategy that
sits on cash during a 48% drawdown is not underperforming. It is functioning.

The regime gates re-engage for longs when:

- ADX drops below 20 and BBW normalizes → `RANGE` regime, lower-band reclaim
  entries permitted
- ADX re-establishes above 25 with EMA_21 crossing back above EMA_55 →
  `BULL_TREND` regime, EMA21 pullback entries permitted

Neither condition is present in the first week of June 2026. The bot waits.

---

*— Betha Morrison*
