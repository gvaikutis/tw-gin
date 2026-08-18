---
name: pinescript-indicator-creator
description: "Use when creating or modifying this workspace's Pine Script v6 liquidity-map indicator, including pivot liquidity zones, equal highs/lows, volume strength, sweep mitigation, target forecasts, target invalidation, reversal flips, and the candle-volume dashboard."
argument-hint: "Describe the liquidity-map or dashboard change"
---

# Pine Liquidity Map Indicator

Use this skill for changes to `PineScripts/Last Candle Volume Info.pine`.

## Core Model

- Use Pine Script v6 and `indicator(..., overlay=true, max_lines_count=500)`.
- Identify BSL from confirmed `ta.pivothigh()` values and SSL from confirmed `ta.pivotlow()` values.
- Store every level in the `LiquidityZone` UDT: price, BSL/SSL side, touch count, high-volume flag, active status, and line reference.
- Treat nearby same-side pivots within `tolerancePercent` as EQH/EQL: increase `touches` rather than creating a second zone.
- Use pivot-bar volume relative to the volume SMA to assign high-volume strength. EQH/EQL has highest priority, followed by high-volume pivots, then normal pivots.

## Map And Mitigation

- BSL is mint/green; SSL is crimson/red. Use thicker, more opaque lines for stronger zones.
- Keep active levels extending right. Respect the user-selected `Wick Sweep` or `Close Breach` mitigation rule.
- A mitigated zone becomes inactive. Keep it as a gray dashed historical line when `keepMitigated` is enabled; otherwise delete it.
- Cap stored zones using `maxZones`, deleting the oldest line with its array entry.

## Targeting Rules

- A swept SSL creates a bull target at active BSL above price. A swept BSL creates a bear target at active SSL below price.
- Select by zone priority first and distance second. Do not use BOS, CHoCH, or MSB as a prerequisite.
- Draw only one active forecast target line. It is yellow and extends right; an achieved target becomes gray and dashed.
- Apply both target-distance controls before displaying a target: percentage distance and optional ATR distance.
- If filtered, show `TGT DISTANT` in the Signal column and do not draw an active target line.

## Invalidation And Reversals

- Record the originating sweep candle low for bull targets and high for bear targets.
- Cancel a bull target when price breaks below its originating low; cancel a bear target when price breaks above its originating high.
- On cancellation, delete the yellow target line immediately and retain `REVERSAL / TGT CANCELLED` on the originating dashboard row.
- Immediately flip direction after cancellation: an invalidated bull target scans active SSL below price and reports `REVERSAL -> BEAR TGT: [price]`; apply the inverse behavior to invalidated bear targets.

## Volume Confirmation

- Dashboard Buy/Sell volume is an estimate from close location in the candle range; TradingView does not provide bid/ask volume here.
- Maintain a stack of active bullish order-block boxes from confirmed pivot lows and one active bearish order-block box from the latest confirmed pivot high. Draw bullish blocks green and bearish blocks violet; gray them when invalidated.
- Maintain a stack of bearish order blocks as resistance. While price closes below the highest active bearish-block top, lock only the live `C1` dashboard row to `SELL-SIDE BIAS / BEAR OB CAP` and suppress its whale buys or other counter-trend signals. Release the lock only on a close above that highest top; never let this live condition overwrite C2-C10 history.
- On bearish BOS or bearish CHoCH, select a bullish block strictly below price by power, touches, then distance. Set the yellow bear target to its top and report `BOS CONFIRMED -> BEAR TGT: OB [price]` or the equivalent CHoCH message.
- If price reaches the targeted bullish block with positive Delta and CMF above zero, clear bear mode and report `OB REACHED -> WAITING FOR CONFIRMATION` instead of locking a bearish target hit.
- A `WHALE BUY` requires candle overlap with the active bullish block, positive Delta, above-average total volume, and the existing high buy-volume condition. A bullish-block test with non-positive Delta shows `OB TESTING / WAITING FOR DELTA` instead.
- Apply the symmetrical active bearish-block, negative-Delta, and above-average-volume requirements to `WHALE SELL`.
- When an uncleared bearish block sits above a confirmed whale buy, cap its message at `WHALE BUY -> TGT: BEAR OB [price]`.
- A bull sweep requires positive Delta, positive CMF, or CMF recovery from below zero when `requireBullConfirmation` is enabled.
- A bull sweep with Delta below `negativeDeltaThreshold` and CMF below zero is failed: report `SWEEP FAILED -> CONTINUATION DOWN` and seek an SSL bear target.
- Highlight reversal signals with solid orange only when their Delta and CMF agree with the reversal direction.
- Reversal probability labels show probability, the strength word, and the directional volume: `55% STRONG\nBuy: 140.76M` for bullish and `43% HIGH VOL\nSell: 229.53M` for bearish. Directional volume reuses the live `C1` dashboard Buy/Sell estimate and `compactVolume()` formatting.
- Classify labels with the timeframe-scaled `High-vol threshold` inputs (three buckets: `≤ 15m`, `16m–1H`, `2H+`), each default `10M`. The active bucket is auto-selected from the chart timeframe. Probability below 50% is `WEAK` when total volume is below the bucket threshold, `HIGH VOL` when at or above it, and `STRONG` when probability is at least 50%.
- Keep weak bullish labels orange and weak bearish labels gray. Low-probability `HIGH VOL` labels are blue for bullish reversals and purple for bearish reversals. Strong bullish labels are teal, strong bearish labels orange.

## Dashboard Contract

- Preserve the seven columns: `Candle`, `Buy`, `Sell`, `Total`, `Delta`, `CMF`, and `Signal`.
- Render the fixed 10 recent-candle rows only under `barstate.islast`.
- The Signal column owns sweep, target, target-hit, distance-filter, failure, and reversal messages. Do not replace the candle-volume dashboard with a separate table.
- Keep target-hit rows blue and cancelled-target origin rows orange.
- Persist superseded target outcomes by origin bar. A target superseded during bearish expansion must remain `SUPERCEDED -> BEARISH MOMENTUM` with a solid dark-charcoal background, rather than being recomputed by later candles.

## Flow & Trend Confirmation

- Maintain a running Cumulative Volume Delta (`cvd += nz(deltaVol)`) from the estimated per-candle delta. Its absolute value is meaningless; only its slope versus price matters.
- Detect CVD divergence at confirmed pivots using the CVD value at the pivot bar (`cvd[rightBars]`) versus the prior same-side pivot: a higher price high with a lower CVD high is bearish divergence; a lower price low with a higher CVD low is bullish divergence.
- Mark divergences on chart with a small triangle offset back to the pivot bar (green up below price for bullish, red down above price for bearish) and add `+15` to the reversal-probability score when a divergence inside `cvdDivergenceWindow` agrees with the reversal-label direction.
- Tag the matching dashboard `Signal` row with `| BULL DIV` or `| BEAR DIV` (or the tag alone when the row had no other signal). Do not let the tag change the row background or WHALE alert parsing.
- Compute an HTF trend bias from an EMA on `trendBiasTimeframe` (default 4H): `UP` above, `DOWN` below, `RANGE` otherwise. Use it as context only; reversals against the bias are counter-trend.
- Compute RVOL as current volume divided by the `whaleLength` volume average; green at or above 1.5x, gray below 1.0x.
- Show the HTF trend bias, CVD divergence state (`BULL DIV` / `BEAR DIV` / `-`), and RVOL as extra rows in the top-right H4 panel; never add them as columns to the bottom-right seven-column dashboard.

## Graphic Elements Contract

- Preserve the complete chart rendering contract documented in [Last Candle Volume graphic items.txt](../../../PineScripts/Last%20Candle%20Volume%20graphic%20items.txt).
- Active BSL/SSL levels extend right in mint/crimson; mitigated levels are gray dashed history or are deleted according to `keepMitigated`.
- Active bullish order-block boxes are green, active bearish order-block boxes are violet, and invalidated blocks are gray and stop extending. Bullish order-block boxes have no Buy label; bearish labels remain at the bearish block top.
- Each bullish CVD-divergence triangle has a `Buy: $...` label directly below its pivot candle, using that candle's estimated buy value.
- Each bearish CVD-divergence triangle has a `Sell: $...` label directly above its pivot candle, using that candle's estimated sell value.
- Keep the locked H4 levels, orange statistical corridors/fills, optional lower-timeframe H4 projections, yellow active target line/tag, reversal labels, CVD/OI markers, and absorption ceilings visually distinct as documented.
- Keep the top-right H4 panel, bottom-right 10-candle dashboard, and optional bottom-left failure table in their established positions. Do not turn Data Window export plots into chart plots.
- Estimated Buy/Sell figures derive from candle close location in the high-low range. Never present them as exchange bid/ask data.

## Change Checklist

1. Keep zone, target, and table state aligned through the same series events.
2. Guard array loops when the array is empty and delete line objects when pruning or removing state.
3. Run VS Code diagnostics on the Pine file after edits. Pine runtime behavior must be compiled in TradingView.
