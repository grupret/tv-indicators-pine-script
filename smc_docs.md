# Documentation: SMC Overlay [Intraday] (smc.pine)

## 1. Overview of All Points

The `smc.pine` script is an advanced main-chart overlay designed to automatically chart Smart Money Concepts (SMC) in real-time. It maps out institutional footprints, liquidity behavior, and structural trend shifts.

Here are the key components and features:

*   **Variables and Inputs**: Full customization for Moving Averages (EMA 9/20/50, SMA 200), VWAP bands, specific constraints for Order Blocks (lookback periods and capacity limits), ATR volatility bounds, and dynamically adjustable Risk/Reward mappings.
*   **Moving Averages & VWAP (Anchored)**: Standard trend baselines (EMA, SMA) togglable on/off. Also includes a day-anchored VWAP (resets daily) supported by ±1 and ±2 Standard Deviation (σ) deviation bands.
*   **Confirmed Swings (Market Structure Nubs)**: Calculates structurally intact Swing Highs and Swing Lows based on a defined bar formation lookback.
*   **Order Block (OB) Engine**: Algorithmically defines bearish/bullish order zones by measuring ATR-derived price impulses combined with historical opposing-color candles.
*   **Active Array Management**: Dynamically tracks, extends, limits (via `i_obMax`), and purges Order Blocks immediately upon structural invalidation by closing prices.
*   **Liquidity Sweeps**: Recognizes specific patterns where a wick breaks a former structural swing pivot, but the candle ultimately closes back within the original range, signifying stop-hunting or liquidity grabs.
*   **Market Structure Updates**: Tracks trends dynamically, plotting explicit "Break of Structure" (BOS) continuing trends, and "Change of Character" (CHoCH) shifts denoting trend reversals.
*   **Support & Resistance Pruning**: Maintains only the three most recent logical support/resistance levels, automatically deleting them off the chart once they are definitively broken.
*   **Institutional Zones & Target Projection**: High-volume, impulsive Order Blocks generate high-probability trade setups ("INST BUY / SELL"). It calculates visual entries, dynamic Stop Losses (SL1/SL2), and explicit visual Take Profit (TP1/TP2/TP3) boundaries based on ATR and Risk multipliers.
*   **Algorithmic Alerts**: Contains native Pine Script `alertcondition()` hooks for all underlying events (BOS, Sweeps, Institutional setups, and Order Block touches).

---

## 2. Details of Implementation & Component Impact

The following section breaks down exactly how the indicator's sub-systems compute their logic and interact to build the final trading view.

### Day-Anchored VWAP and Variance
*   **Mechanism**: Employs continuous accumulation loops for typical price (`hlc3`), volume, and squared price variance, resetting strictly upon the daily session boundary check `ta.change(time("D")) != 0`.
*   **Impact**: Acts as an institutional "fair value" baseline. The ±1σ and ±2σ bands dynamically expand with the day's volatility profile. A failure to trade outside the bands signifies range-bound conditions, while a break usually coincides with an institutional impulse.

### Swing Pivot Engine
*   **Mechanism**: Uses `ta.pivothigh` and `ta.pivotlow` across `i_swingLB` logic limits (requiring *N* bars on both the left and right sides to confirm the pivot).
*   **Impact**: This is the anchor point for over half the script. Because it requires bars *after* the pivot to close before confirming, it creates a latency factor (known as repainting prevention). Sweeps, BOS, CHoCH, and SL2 placements rely exclusively on the correct tracking of these confirmed historical markers.

### Order Block Identification and Validation
*   **Mechanism**: Finding an OB requires two steps. 
    1. A volatility breakout: The current bar's close must exceed the close of a past bar (`i_obLookback`) by an ATR-multiplied threshold (`atrVal * i_atrMult`). 
    2. Candidate identification: The system loops back to locate the most recent oppositely-colored candle that sparked the move.
*   **Impact**: Relying on ATR drastically reduces noise. If the market is highly volatile, the required threshold for an order block to form expands automatically. The OB array loop actively projects a visual `box` rightward into the future until the price specifically breaches the opposite bound of the OB, triggering array deletion.

### Stop Hunts / Liquidity Sweep Mechanics
*   **Mechanism**: By comparing the current candle mapping against `lastSH` (Swing High) and `lastSL` (Swing Low), the sweep logic dictates `high > lastSH and close < lastSH` (bearish sweep).
*   **Impact**: Extremely potent for identifying exhaustion. It flags instances where price mechanically breached resistance strictly to trigger stops above, only to sharply reverse into the body of the range.

### Structure Mapping: BOS and CHoCH
*   **Mechanism**: 
    *   **BOS (Break of Structure)** requires a `close` strictly beyond the previous confirmed pivot in the direction of the sustained trend.
    *   **CHoCH (Change of Character)** triggers when a swing high forms lower than the most recent swing high (`msLastSH`) whilst an uptrend was previously locked in, indicating failing momentum.
*   **Impact**: Instead of utilizing a slow mapping moving average, the trader receives pure price-action structural reads. Expanding `i_swingLB` makes the required BOS trends macroscopic, while narrowing it creates micro-structure intraday scalping environments.

### Institutional Entry Projections
*   **Mechanism**: Combines High Relative Volume (95th percentile over `i_volPeriod`) and an active Order Block impulse. Upon forming, it maps physical trading limits:
    *   `Risk` = distance between entry close and the fundamental invalidation low/high of the originating OB.
    *   SL bounds map to the OB (`SL1`) and the furthest macro swing structure (`SL2`).
    *   TP bounds derive symmetrically leveraging the variable inputs (`i_tp1_rr`, etc.) multiplied directly against calculated `Risk`.
*   **Impact**: Instantly provides the trader with a fully packaged, rules-based discretionary setup immediately upon candle close. This forces strict Risk-to-Reward tracking and protects against emotionally driven trade positioning.
