# RSI 70/30 Reversal Dashboard — Binance Futures (5m + 15m)

Two Pine Script v6 files plus this guide.

| File | Purpose |
|---|---|
| `rsi_reversal_dashboard.pine` | **Indicator**: mobile dashboard, confirmed LONG/SHORT signals, TradingView alerts, simulated stats (history vs live) |
| `rsi_reversal_strategy.pine` | **Strategy**: same signal logic, for Strategy Tester backtests with fees, slippage and risk-based sizing |

> **Status:** I could not compile Pine Script in my workspace, so neither file has been run in TradingView yet. Expect to possibly fix a small compile error on first paste (see Troubleshooting). Nothing here has been backtested, so **no profitability is claimed**.

---

## 1. How the logic works

**LONG** (SHORT is the mirror image)
1. **Arm:** 5m RSI(14) closes ≤ 30 → LONG WATCH.
2. **Turn:** RSI rises at least `turnMin` (default 1.0) points off its lowest value. A new lower RSI low resets the turn.
3. **Supertrend:** 5m Supertrend (ATR 12, mult 4) is bullish. Optionally it must have freshly flipped after the RSI low.
4. **EMA score (0–5):** EMA7>EMA20, close>EMA20, EMA7 crossed above EMA20 within N bars, close>EMA50, close>EMA200. Default minimum is 3.
5. **15m context:** last *fully closed* 15m bar only (Supertrend + ribbon). Reversals are labelled **TREND-ALIGNED** or **COUNTERTREND**. A countertrend setup needs extra EMA score (default +1). **Strict mode** blocks countertrend signals entirely.
6. **Stop check:** SL = 5m Supertrend level. It must sit on the correct side of entry and between the min and max stop distance (default 0.10%–2.50%), otherwise the signal is rejected.
7. **Fire:** only on the 5m candle close, and only once.

**No duplicate signals:** after a signal (or an expired or cancelled setup) the setup is *locked*. It unlocks only after RSI leaves the extreme zone by the re-arm buffer (default 10 points: LONG needs RSI ≥ 40, SHORT needs RSI ≤ 60) and then hits 30/70 again.

**Trade plan:** entry ref = signal-candle close. LONG: `TP = Entry + 2 × (Entry − SL)`. SHORT: `TP = Entry − 2 × (SL − Entry)`. The R:R is configurable.

**Non-repainting:** 15m data uses `[1]` with `lookahead_on`, so it is always the last closed 15m bar and identical on history and live. This is deliberately conservative, because the 15m view can lag by up to one 15m bar. Signals only evaluate when `barstate.isconfirmed`, and alerts use once-per-bar-close.

---

## 2. Installation

1. Open TradingView (web) → chart **BINANCE:BTCUSDT.P** (or any Binance perpetual) → set the timeframe to **5m**.
2. Open **Pine Editor** (bottom panel) → **Open → New indicator**, delete the template, paste all of `rsi_reversal_dashboard.pine`.
3. Click **Save** (name it), then **Add to chart**. Fix any compile error shown (see Troubleshooting).
4. For backtests: repeat with **New strategy** and `rsi_reversal_strategy.pine`.
5. **Phone:** saved scripts sync to the TradingView app. Open the chart there, tap **Indicators → My scripts** (or *Personal*), add it. Use **Settings → Dashboard → Text size / Position** to fit your screen. Turn off *Show simulated stats* and *Show last 5 signals* for a compact view.

---

## 3. Alert setup (push to your phone)

TradingView alerts are created **per symbol and per chart timeframe** (the chart must be on 5m).

**Option A — recommended (full message with SL/TP/RR)**
1. On the 5m chart with the indicator added, tap the **Alert (clock) icon → Create alert**.
2. **Condition:** `RSI 70/30 Reversal Dashboard` → **Any alert() function call**.
3. **Trigger:** leave as the script sets it (once per bar close). **Expiration:** open-ended if your plan allows.
4. **Notifications:** tick **Notify in app** (push) and optionally Email / Webhook.
5. Name it e.g. `RSI-REV BTCUSDT.P 5m` → **Create**.

Message format:
`CONFIRMED LONG | BTCUSDT.P | 5m | Entry ref 66000.1 | SL 65500.0 | TP 67000.2 | RR 1:2 | COUNTERTREND`

**Option B — built-in conditions**
Condition = `RSI-REV` → **CONFIRMED LONG**, **CONFIRMED SHORT**, or **ANY CONFIRMED SIGNAL**. Set *Trigger → Once Per Bar Close*. These messages use `{{plot()}}` placeholders for entry/SL/TP.

⚠ Use **either A or B** for a given symbol, not both, or you will get two notifications per signal. If you use Option B, turn off *Send dynamic alert() messages* in the script settings.

**Phone push:** in the TradingView app go to **Settings → Notifications** and allow push. Also make sure iOS/Android system notifications are on for TradingView. Send a test by creating a throwaway price alert first.

**Limits:** one alert per coin. The number of active alerts depends on your TradingView plan. Alerts only run while the alert exists on TradingView's servers, so your PC can be off.

---

## 4. Backtesting guidance

**Settings (strategy → Settings → Properties):**
- **Initial capital:** your real account size.
- **Commission:** percent, **0.05% per side** by default (typical Binance USDT-M taker fee). Confirm your own tier, since BNB discounts and VIP levels change it.
- **Slippage (ticks):** the default is a placeholder. Convert your assumption to ticks: `ticks = price × slip% ÷ mintick`. Example: BTC at 100,000, mintick 0.1, 0.02% → 20 ticks. Use more for illiquid alts.
- Keep **Recalculate on every tick** off. Orders fill at the **next bar open**.
- If you have **Bar Magnifier** (paid plans), enable it so TP/SL ordering inside a 5m bar is resolved on lower-timeframe data.
- Use **Deep Backtesting** (paid plans) to test further back than the chart's loaded bars.

**What the strategy does *not* model:** funding payments, liquidation, partial fills, order-book depth, exchange outages, and Binance minimum order sizes or quantity rounding.

**Reading the numbers:** the on-chart table and the Strategy Tester show trade count, win rate, net profit, max drawdown and expectancy (a rough R-multiple figure is included). A 1:2 target needs about 33.3% wins to break even *before* costs. After fees and slippage the required win rate is higher: `breakeven = (1 + c) ÷ (RR + 1)`, where `c` = round-trip cost ÷ stop distance. A tight 0.3% stop with 0.14% round-trip cost has c ≈ 0.47, so you need about 49% wins at 1:2. Tight stops are therefore expensive.

**Don't fool yourself:**
- Aim for a meaningful sample (a few hundred trades per setting set is a common rough floor). Fewer trades means the result is mostly noise.
- Pick settings on one date range, then test on a **later, untouched** range (out-of-sample).
- Don't tune many inputs to maximise one coin's equity curve. That is curve-fitting.
- Test several market regimes (trend, range, high-vol crashes) and several coins.
- Compare against simply not trading. A positive backtest does not guarantee future results.

**Per-coin and per-timeframe breakdown:** Pine cannot aggregate results across symbols, so do it manually:
1. Apply the strategy to each coin and timeframe you care about, with identical settings and date range.
2. Record the Strategy Tester summary into a table:

| Coin | TF | Range | Trades | Win % | Net profit | Max DD | Expectancy | Notes |
|---|---|---|---|---|---|---|---|---|
| BTCUSDT.P | 5m | | | | | | | |
| ETHUSDT.P | 5m | | | | | | | |

3. Or use **List of trades → Export** and pivot by coin in a spreadsheet.

**Backtest vs live (kept separate):**
- The **strategy** is backtest only.
- The **indicator** dashboard shows two columns of *simulated* results: **Backtest*** (trades on loaded history) and **Live**** (trades that closed on realtime bars since you loaded the chart). Live resets when the chart reloads, and it is still simulated from candle highs/lows, not your actual fills.
- For a real live record, log each alert and your actual fills (price, fees, slippage) in a sheet and compare against the backtest expectancy over time.

**Simulation rules in the indicator stats:** one simulated trade at a time (extra signals are counted as *Skipped*), SL is assumed first if SL and TP are touched in the same candle, and fee + slippage per side are subtracted in R.

---

## 5. Dashboard reference

Status line: `LONG WATCH`, `SHORT WATCH`, `▲ CONFIRMED LONG`, `▼ CONFIRMED SHORT`, `NO TRADE`. Below it you get the 15m trend and alignment, a rules checklist (RSI zone / Turn / ST / EMA / 15m / SL), a 5m vs 15m table (RSI, Supertrend, ribbon state, price vs each EMA), entry / SL / TP in large text, stop distance, R:R, risk $, size, notional and leverage, simulated stats, and the last 5 signals with TP/SL outcome.

`PREVIEW` entries are what the plan would be if the signal fired right now. `SIGNAL` is the actual confirmed level.

---

## 6. Honest limitations

- Reversal setups will miss some reversals, and will fire false signals (RSI can stay overbought/oversold in strong trends). Strict mode and a higher EMA score reduce false signals but also reduce trade count. Nothing eliminates them.
- Supertrend ATR 12 × 4 is a slow, wide stop. Many 5m setups will be rejected by the 2.5% max-stop rule on volatile alts. That is intended, but check how many signals you actually get.
- The 15m filter is lagged by design (closed bars only).
- Entry reference is the signal-candle close. A real market order fills later and at a different price. The strategy uses the next bar open.
- Leverage and position size shown are informational. Check Binance's actual margin and liquidation price before trading.

---

## 7. Troubleshooting

- **Compile error on a specific line:** copy the exact message. Common first-paste causes are a typo introduced when pasting or a TradingView behaviour change. Send me the line and message and I'll fix it.
- **Table not visible on mobile:** open the indicator settings, set Position to *top left* or *bottom center*, and reduce text size.
- **`Trend TF must be higher than chart TF` warning:** the trend timeframe input must be larger than the chart timeframe.
- **No signals at all:** lower *Min EMA score*, widen *max stop %*, set Strict mode off, or use *State* Supertrend mode instead of *Fresh flip*.
- **Alerts fire twice:** you have both an `alert()` alert and a built-in-condition alert on the same symbol.
