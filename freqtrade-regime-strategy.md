# Building a Regime-Adaptive Strategy for Freqtrade

**Author:** Kenshin Himura  
**Date:** June 2026  

> A long-only Freqtrade strategy that classifies the current market regime before generating any signal, and stays in cash when the regime is unfavorable. This article covers the design rationale, the full implementation, and the correct hyperopt and validation workflow.

---

## Table of Contents

1. [Market Context](#1-market-context)
2. [Why Single-Regime Strategies Fail](#2-why-single-regime-strategies-fail)
3. [Design: Regime Detection First](#3-design-regime-detection-first)
4. [Signal Logic per Regime](#4-signal-logic-per-regime)
5. [Risk Management](#5-risk-management)
6. [Implementation Notes](#6-implementation-notes)
7. [Full Strategy Code](#7-full-strategy-code)
8. [Hyperopt and Walk-Forward Validation](#8-hyperopt-and-walk-forward-validation)
9. [Expected Behavior in Current Conditions](#9-expected-behavior-in-current-conditions)
10. [Limitations](#10-limitations)

---

## 1. Market Context

The following snapshot was recorded in early June 2026 and is used here as the operating context for the strategy. Treat the figures as a point-in-time reference; verify against current data before relying on them.

* BTC: ~$64,350, down ~18% from the 30-day high near $78,400
* Total crypto market cap: ~$2.18T, roughly 48% below the prior peak near $4.2T
* Fear & Greed Index: 12 (Extreme Fear)
* 30-day liquidations: ~$3.53B, largest single cascade ~$330M
* BTC dominance: ~60%; Altcoin Season Index: 30 (Bitcoin Season)
* Spot ETF flows: net outflow since mid-May

The relevant point for strategy design is the structure, not the exact numbers: a sharp, high-volatility drawdown that alternates between choppy consolidation and liquidation-driven sell-offs.

Most publicly available Freqtrade strategies are tuned for a single regime, typically the trending up-market of late 2024 to early 2025. A regime shift of this magnitude breaks them systematically. The fix is not parameter retuning; it is a strategy that detects the regime it is operating in before generating a signal.

---

## 2. Why Single-Regime Strategies Fail

**EMA crossover (pure trend-following).** In a trending market, price respects the faster EMA as support on pullbacks and crossovers work. In a liquidation cascade, price gaps through multiple EMA levels in a single candle. The signal fires after the move, the entry lands near the bottom of the cascade, and the next leg down triggers an immediate stoploss. The whipsaw rate in a high-volatility downtrend makes this approach unprofitable.

**Pure RSI mean-reversion.** RSI below 30 correctly identifies oversold conditions in a ranging market. In a sustained downtrend, the same reading identifies the first leg of a multi-leg decline, and the instrument can remain oversold for weeks. Without a regime gate, mean-reversion entries in the current tape repeatedly buy into continued declines.

**Grid bots.** Grid strategies require price to oscillate within a defined range. A drawdown that exits the grid range leaves the bot buying at predefined levels while price keeps falling, ending in a fully deployed position at a deep loss with no exit mechanism. Grid strategies are unsuitable for markets with directional trend risk above the grid's range.

**MACD momentum.** MACD is a lagging indicator by construction, computed as the difference between two EMAs. In a fast-moving market, the signal-line crossover arrives after most of the move has completed, and it generates frequent false signals in ranging conditions. In a regime alternating between consolidation and sharp sell-offs, MACD systematically enters at the wrong point in the cycle.

**ML-based strategies.** A model trained on 2024–2025 data learned a market structure that no longer holds. The current distribution of returns, volatility, and volume falls outside the training distribution, so the model's outputs are extrapolation rather than inference. ML strategies also require retraining pipelines, labeled data, and validation infrastructure that most Freqtrade deployments lack. Added complexity without a maintenance protocol increases operational risk.

---

## 3. Design: Regime Detection First

The strategy classifies the current regime, then applies the signal logic appropriate to that regime. If the regime is ambiguous or unfavorable, it generates no signal.

### Classification Inputs

**ADX(14)** measures trend strength independent of direction. ADX ≥ 25 indicates a trending market; ADX < 20 indicates a ranging market. ADX rises equally in sustained uptrends and downtrends.

**EMA(21) / EMA(55)** determines trend direction once ADX confirms a trend is present. EMA_21 above EMA_55 is bullish structure; EMA_21 below EMA_55 is bearish structure.

**Bollinger Band Width (BBW)** measures volatility expansion, defined as `(upper - lower) / middle`. When BBW exceeds a multiple of its 20-period average, the market is in a volatility-expansion phase, typically associated with liquidation cascades or breakout moves. This condition blocks new long entries regardless of other signals.

### Regime Definitions

| Regime       | ADX   | EMA structure | BBW        | Action                       |
| ------------ | ----- | ------------- | ---------- | ---------------------------- |
| `BULL_TREND` | ≥ 25  | 21 > 55       | normal     | EMA pullback long            |
| `BEAR_TREND` | ≥ 25  | 21 < 55       | normal     | No new longs                 |
| `RANGE`      | < 20  | any           | normal     | RSI + Bollinger mean reversion |
| `EXPANSION`  | any   | any           | > N× avg   | No new longs                 |

`EXPANSION` is evaluated first and overrides the trend and range labels for entry purposes, so `BULL_TREND` and `RANGE` are gated by `not EXPANSION`. `BEAR_TREND` and `EXPANSION` can co-occur; this is harmless because both block new longs and both trigger long exits. An ADX reading between 20 and 25 belongs to no regime by design, which produces no entry. Not every candle requires a trade.

---

## 4. Signal Logic per Regime

**`BULL_TREND` - EMA pullback entry.** Price must be above EMA_55 (trend intact), within 1×ATR of EMA_21 (a pullback rather than an extension), and RSI between 40 and 62 (neither overbought nor crashed). This targets pullback-to-continuation within a defined trend, which has better risk/reward than breakout entries that are already extended from the mean.

**`RANGE` - Bollinger lower-band reclaim with RSI recovery.** The previous candle must have touched or breached the lower band; the current candle must close back above it. RSI must be below the oversold threshold and rising, the current candle must be green, and volume must be at least 1.2× the 20-period average. The exit target is EMA_21. This is a defined-risk, defined-target trade rather than an open-ended trend ride.

Both entries require the regime gate to be satisfied first, so neither fires in `BEAR_TREND` or `EXPANSION`.

---

## 5. Risk Management

### ATR-Based Dynamic Stoploss

A fixed −5% stoploss is calibrated for one volatility environment. With elevated 30-day realized volatility, a 5% intraday swing is normal, so a fixed −5% stop triggers on noise rather than on trade invalidation. An ATR-based stop scales with current volatility: wider when ATR is large, tighter when ATR compresses in a range.

The static `stoploss = -0.05` is retained as an absolute backstop. The custom stoploss computes an N×ATR level and clamps it against this backstop in-function, so the realized loss per trade never exceeds 5% regardless of ATR.

`atr_stoploss_mult` is exposed as a hyperopt parameter (range 1.5–3.0) because the optimal multiplier is pair-dependent. BTC/USDT on 1h, with ATR typically around 1.5–2.0% of price, tends to suit a multiplier of 2.0–2.5; more volatile pairs require a wider multiplier to avoid stopping out on noise.

`stoploss_from_absolute()` converts an absolute price level to the ratio Freqtrade expects, accounting for leverage. A manual ratio calculation without this helper places the stop incorrectly in futures mode.

### Protections

Three protections limit consecutive losses and drawdown: a cooldown after each trade, a stoploss guard that pauses trading after repeated stoploss hits, and an equity-based maximum-drawdown guard. These are defined in the `protections` property.

---

## 6. Implementation Notes

The strategy has no external indicator-library dependency. ADX, RSI, ATR, EMA, and Bollinger Bands are implemented directly with NumPy and pandas, using Wilder's smoothing where applicable, which keeps results reproducible across environments.

`startup_candle_count = 120` ensures EMA_55 and the Wilder-smoothed ADX have converged before the first signal evaluation. Without enough warmup, Freqtrade would evaluate signals on candles where these indicators are still numerically unstable.

**Parameter placement and hyperopt.** This is the most important implementation detail. By default, Freqtrade computes `populate_indicators` once and caches the result; it re-runs only `populate_entry_trend` and `populate_exit_trend` for each hyperopt epoch. Any parameter used inside `populate_indicators` is therefore **not** optimized unless `--analyze-per-epoch` is passed. The regime thresholds (`adx_trend_threshold`, `adx_range_threshold`, `bbw_expansion_factor`) directly affect entries and exits, so the regime classification is computed in a helper called from `populate_entry_trend` and `populate_exit_trend`, not in `populate_indicators`. Only the raw, parameter-free indicators are computed and cached in `populate_indicators`. This makes the regime thresholds optimizable through the normal hyperopt buy-space without `--analyze-per-epoch`.

---

## 7. Full Strategy Code

```python
"""
RegimeAdaptive

Freqtrade strategy for 1h market-regime-adaptive long trading.

Design:
- No external indicator library dependency.
- Long-only. Conservative leverage (1x default).
- Bull-trend entries use EMA pullback logic.
- Range entries use Bollinger lower-band reclaim + RSI recovery.
- Bear and expansion regimes are treated defensively (no new longs, exit longs).
- ATR-based custom stoploss via stoploss_from_absolute.
- Regime thresholds are applied in populate_entry/exit_trend so they are
  optimized correctly by hyperopt without --analyze-per-epoch.

Usage:
- Backtest first, then dry-run. Do not run live without sufficient evidence.
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
        "0": 0.06,
        "60": 0.04,
        "120": 0.02,
        "240": 0.01,
    }

    # Hard floor. The custom stoploss never exceeds this loss.
    stoploss = -0.05

    # Hyperopt search space - intentionally narrow.
    adx_trend_threshold = IntParameter(22, 30, default=25, space="buy", optimize=True)
    adx_range_threshold = IntParameter(15, 22, default=20, space="buy", optimize=True)
    rsi_range_oversold = IntParameter(28, 38, default=35, space="buy", optimize=True)
    bbw_expansion_factor = DecimalParameter(1.5, 2.5, default=2.0, space="buy", optimize=True)
    atr_stoploss_mult = DecimalParameter(1.5, 3.0, default=2.0, space="sell", optimize=True)

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
        delta = series.diff()
        gain = delta.clip(lower=0)
        loss = -delta.clip(upper=0)
        avg_gain = gain.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()
        avg_loss = loss.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()

        rs = avg_gain / avg_loss.replace(0, np.nan)
        rsi = 100 - (100 / (1 + rs))
        # All-gain windows (avg_loss == 0) are RSI 100, not 50
        rsi = rsi.where(avg_loss != 0, 100.0)
        # Startup window where averages are undefined
        return rsi.fillna(50)

    @staticmethod
    def _atr(dataframe: DataFrame, length: int = 14) -> Series:
        high = dataframe["high"]
        low = dataframe["low"]
        close = dataframe["close"]
        prev_close = close.shift(1)
        true_range = pd.concat(
            [high - low, (high - prev_close).abs(), (low - prev_close).abs()],
            axis=1,
        ).max(axis=1)
        return true_range.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()

    @staticmethod
    def _adx(dataframe: DataFrame, length: int = 14) -> Series:
        """Wilder-style ADX."""
        high = dataframe["high"]
        low = dataframe["low"]
        close = dataframe["close"]

        up_move = high.diff()
        down_move = -low.diff()

        plus_dm = Series(
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

        atr = true_range.ewm(alpha=1 / length, adjust=False, min_periods=length).mean()
        plus_di = 100 * plus_dm.ewm(alpha=1 / length, adjust=False, min_periods=length).mean() / atr.replace(0, np.nan)
        minus_di = 100 * minus_dm.ewm(alpha=1 / length, adjust=False, min_periods=length).mean() / atr.replace(0, np.nan)
        dx = 100 * (plus_di - minus_di).abs() / (plus_di + minus_di).replace(0, np.nan)
        return dx.ewm(alpha=1 / length, adjust=False, min_periods=length).mean().fillna(0)

    @staticmethod
    def _bollinger_bands(
        series: Series, length: int = 20, std: float = 2.0
    ) -> tuple[Series, Series, Series]:
        middle = series.rolling(length, min_periods=length).mean()
        deviation = series.rolling(length, min_periods=length).std(ddof=0)
        return middle - std * deviation, middle, middle + std * deviation

    # ── freqtrade interface ───────────────────────────────────────────────────

    def informative_pairs(self):
        return []

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # Only parameter-free indicators are computed here so they can be cached.
        dataframe["ema_21"] = self._ema(dataframe["close"], 21)
        dataframe["ema_55"] = self._ema(dataframe["close"], 55)
        dataframe["adx"] = self._adx(dataframe, 14)
        dataframe["rsi"] = self._rsi(dataframe["close"], 14)

        bb_lower, bb_middle, bb_upper = self._bollinger_bands(dataframe["close"], 20, 2.0)
        dataframe["bb_lower"] = bb_lower
        dataframe["bb_middle"] = bb_middle
        dataframe["bb_upper"] = bb_upper

        dataframe["bbw"] = (
            (dataframe["bb_upper"] - dataframe["bb_lower"])
            / dataframe["bb_middle"].replace(0, np.nan)
        ).replace([np.inf, -np.inf], np.nan)
        dataframe["bbw_avg"] = dataframe["bbw"].rolling(20, min_periods=20).mean()

        dataframe["atr"] = self._atr(dataframe, 14)
        dataframe["volume_avg"] = dataframe["volume"].rolling(20, min_periods=20).mean()

        dataframe["green_candle"] = dataframe["close"] > dataframe["open"]
        dataframe["rsi_rising"] = dataframe["rsi"] > dataframe["rsi"].shift(1)

        return dataframe

    def _regimes(self, dataframe: DataFrame) -> dict:
        """Classify regimes from cached indicators using current parameter values.

        Called from populate_entry_trend and populate_exit_trend so that the
        threshold parameters are re-evaluated per hyperopt epoch.
        """
        adx_trend = self.adx_trend_threshold.value
        adx_range = self.adx_range_threshold.value
        bbw_factor = self.bbw_expansion_factor.value

        expansion = (
            dataframe["bbw_avg"].notna()
            & dataframe["bbw"].notna()
            & (dataframe["bbw"] > bbw_factor * dataframe["bbw_avg"])
        )
        bull_trend = (
            (dataframe["adx"] >= adx_trend)
            & (dataframe["ema_21"] > dataframe["ema_55"])
            & ~expansion
        )
        bear_trend = (
            (dataframe["adx"] >= adx_trend)
            & (dataframe["ema_21"] < dataframe["ema_55"])
        )
        range_regime = (
            (dataframe["adx"] < adx_range)
            & ~expansion
        )
        return {
            "expansion": expansion,
            "bull_trend": bull_trend,
            "bear_trend": bear_trend,
            "range": range_regime,
        }

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[:, "enter_long"] = 0
        dataframe.loc[:, "enter_tag"] = None

        regimes = self._regimes(dataframe)
        rsi_os = self.rsi_range_oversold.value

        # BULL_TREND: pullback to EMA21 while structurally above EMA55.
        bull_entry = (
            regimes["bull_trend"]
            & (dataframe["close"] > dataframe["ema_55"])
            & (dataframe["close"] <= dataframe["ema_21"] + dataframe["atr"])
            & (dataframe["close"] >= dataframe["ema_21"] - dataframe["atr"])
            & dataframe["rsi"].between(40, 62)
            & (dataframe["volume"] > 0)
        )

        # RANGE: lower-band reclaim - previous candle at/below the lower band,
        # current candle closes back above it, RSI rising, green candle, volume.
        lower_band_reclaim = (
            (dataframe["close"].shift(1) <= dataframe["bb_lower"].shift(1) * 1.005)
            & (dataframe["close"] > dataframe["bb_lower"])
        )
        range_entry = (
            regimes["range"]
            & (dataframe["rsi"] < rsi_os)
            & dataframe["rsi_rising"]
            & dataframe["green_candle"]
            & lower_band_reclaim
            & (dataframe["volume"] >= 1.2 * dataframe["volume_avg"])
        )

        dataframe.loc[bull_entry, ["enter_long", "enter_tag"]] = (1, "bull_ema21_pullback")
        dataframe.loc[range_entry, ["enter_long", "enter_tag"]] = (1, "range_bb_reclaim_rsi")

        return dataframe

    def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe.loc[:, "exit_long"] = 0
        dataframe.loc[:, "exit_tag"] = None

        regimes = self._regimes(dataframe)

        hostile_regime = regimes["bear_trend"] | regimes["expansion"]
        range_mean_revert = regimes["range"] & (dataframe["close"] >= dataframe["ema_21"])

        dataframe.loc[hostile_regime, ["exit_long", "exit_tag"]] = (1, "hostile_regime")
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
        """ATR-based stoploss, clamped to the static stoploss as a hard floor.

        Returns a ratio relative to current_rate via stoploss_from_absolute,
        accounting for leverage. Returns None when ATR is unavailable, leaving
        the existing stop unchanged.
        """
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
        if dataframe is None or dataframe.empty:
            return None

        atr = dataframe.iloc[-1]["atr"]
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

## 8. Hyperopt and Walk-Forward Validation

Hyperopt on in-sample data without out-of-sample validation produces overfit parameters. Split the data and validate on a period the optimizer never saw.

```bash
# In-sample optimization: Jan 2025 – Mar 2026
freqtrade hyperopt \
  --strategy RegimeAdaptive \
  --hyperopt-loss SharpeHyperOptLoss \
  --timerange 20250101-20260301 \
  --pairs BTC/USDT ETH/USDT \
  --epochs 300

# Out-of-sample validation: Apr – Jun 2026
freqtrade backtesting \
  --strategy RegimeAdaptive \
  --timerange 20260401-20260607 \
  --pairs BTC/USDT ETH/USDT
```

Sharpe ratio is used as the hyperopt loss rather than total profit. Maximizing profit drives the optimizer toward maximum risk in the most favorable part of the training period. Sharpe penalizes drawdown relative to return, producing parameters more likely to survive a regime shift.

Because the regime thresholds are applied in `populate_entry_trend` and `populate_exit_trend` (Section 6), the buy-space parameters are optimized correctly here without `--analyze-per-epoch`. If the thresholds had been left in `populate_indicators`, this command would silently optimize only `rsi_range_oversold` and `atr_stoploss_mult`, leaving the regime thresholds fixed at their defaults.

Acceptance criterion: if out-of-sample Sharpe drops more than 40% relative to in-sample Sharpe, treat the parameters as overfit and do not deploy.

---

## 9. Expected Behavior in Current Conditions

Under the early-June 2026 context in Section 1 (`BEAR_TREND` or `EXPANSION` on BTC/USDT 1h), the strategy generates no long entries. That is the intended output: a strategy that holds cash through a deep drawdown is functioning as designed, not underperforming.

The long side re-engages when either condition is met:

* ADX falls below the range threshold and BBW normalizes, entering the `RANGE` regime, which permits lower-band reclaim entries.
* ADX rises above the trend threshold with EMA_21 crossing back above EMA_55, entering the `BULL_TREND` regime, which permits EMA_21 pullback entries.

Neither condition holds in the recorded snapshot, so the strategy stays flat.

---

## 10. Limitations

* **Long-only.** `can_short = False`. The strategy avoids losses in bear regimes by staying in cash but does not profit from declines. The `custom_stoploss` short branch and the `leverage` method are present for completeness but are inactive in spot mode.
* **Regime detection is reactive.** ADX, EMA, and BBW are all derived from past prices and confirm a regime only after it has begun. The strategy does not anticipate regime changes; it classifies the current one.
* **Whipsaw at regime boundaries.** Near the ADX 20/25 thresholds, classification can oscillate between candles. The cooldown and stoploss-guard protections limit the cost of this, but they do not eliminate it.
* **Parameter ranges are pair-calibrated.** The defaults and search ranges are set with BTC/USDT and ETH/USDT 1h in mind. Other pairs and timeframes require re-running the hyperopt and validation workflow in Section 8.
* **Backtest realism.** Spot backtests do not fully capture slippage during expansion regimes, which is exactly when fills are worst. Dry-run before any live deployment, and size positions conservatively.

This strategy is provided for research and educational purposes. It is not financial advice, and it requires independent backtesting and dry-run validation before any use with real capital.

---

*- Kenshin Himura*
