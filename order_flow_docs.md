# Documentation: Order Flow Pane [Intraday] (order.pine)

## 1. Overview of All Points

The `order.pine` script is an indicator designed to be plotted in a separate pane below the main chart. Its primary goal is to provide intraday order flow approximations and volume profile analysis. 

Here are the key components and features:

*   **Variables and Inputs**: Customizable parameters for High Volume Trade (HVT) levels, Moving Average limits, Cumulative Delta visibility, and Institutional order block tracking settings.
*   **Volume Delta Calculation**: Approximates the separation of total volume into "Buy Volume" and "Sell Volume" by checking if a candle closed higher or lower than its open.
*   **Cumulative Delta**: Maintains a running tally of buy vs. sell pressure throughout the current trading session (resets daily). It normalizes this value as a percentage between -100% and +100%.
*   **Heatmap Histogram**: Plots the individual candle volume delta as a histogram, applying a color-coded heatmap based on how the current volume compares to its rolling simple moving average (SMA).
*   **High Volume Trade (HVT) Detection**: Flags individual bars specifically where the volume strongly outpaces the moving average, applying special background colors and labels.
*   **Institutional Marker Synchronization**: Mirrors the order block logic from the companion `smc.pine` script. When a high-volume institutional impulse occurs on the main chart, it syncs a corresponding label ("INST BUY" or "INST SELL") directly on the volume delta histogram.
*   **Disclosure Warning**: A persistent UI table to remind the user that Pine Script does not have native Level 2 order book access, so the resulting delta is an algorithmic approximation.

---

## 2. Details of Implementation & Component Impact

The following section describes how each component functions mathematically and how it impacts the signals generated.

### Volume Delta & Approximation Logic
*   **Mechanism**: The script categorizes total candle volume as `Buy Volume` if `close >= open`, or `Sell Volume` if `close < open`. 
*   **Impact**: While true order flow uses tick-by-tick bid/ask lifting (delta), this approximation acts as a structural volume bias. It relies strictly on the candle's resulting momentum. Therefore, a large wick rejecting price may falsely categorize heavy selling volume as buying volume if the open/close slightly favors a green candle.

### Cumulative Delta (Session-Based)
*   **Mechanism**: A session-spanning accumulator (`_cumBuy` and `_cumSell`). It automatically resets back to zero whenever the daily trading session (`ta.change(time("D")) != 0`) changes.
*   **Impact**: It visualizes the macroscopic control of a session. A strong, rising cumulative delta line implies sustained programmatic buying. By normalizing the result to a percentage, the curve remains bounded and proportional, meaning the user can easily observe divergence (e.g., if price makes a new high but cumulative delta drops, suggesting weak participation).

### Volumetric Heatmap Coloring
*   **Mechanism**: It takes a rolling `i_hvtPeriod` length SMA of standard volume and evaluates the ratio of the current volume divided by this average (`volRatio`).
    *   Ratio > 2.5 → Red (Extreme)
    *   Ratio > 1.5 → Orange (High)
    *   Ratio > 1.0 → Teal (Above Average)
    *   Ratio > 0.5 → Blue (Average)
    *   Ratio < 0.5 → Gray (Low)
*   **Impact**: By tying the histogram color strictly to anomalous volume expansion rather than just bullish/bearish alignment, it inherently alerts the trader to institutional participation or climax/capitulation bars. The exact sensitivity heavily relies on `i_hvtPeriod`. 

### High Volume Trade (HVT) Detection
*   **Mechanism**: Takes a condition `if volume > i_hvtMult * volAvg` mapped to background highlights.
*   **Impact**: This makes identifying volatility expansion completely passive for the user. If the user shifts `i_hvtMult` from `1.5` to `2.0`, the system will yield significantly fewer signals, but the signals flagged will represent far more powerful deviations from the mean context, filtering out intraday noise.

### Institutional Marker Synchronization
*   **Mechanism**: Instead of strictly being a volume tool, it recalculates the SMC logic found in the main overlay:
    *   Calculates the Average True Range (`i_atrPeriod`).
    *   Finds an "Impulse Array" by measuring if the current `close` surpasses the `close` of the respective candle `i_obLookback` bars ago *plus* the ATR factor (`atrVal * i_atrMult`).
    *   Requires volume to be globally significant (in the 95th percentile relative to the last `i_volPeriod` bars).
*   **Impact**: It ensures structural integrity between panes. The volume pane is not just a secondary indicator; it actively informs the user of a validated Smart Money Order Block on the specific bar that created the structural divergence. By marrying High Volatility + Price Impulse against an ATR threshold + Historical Order Block, it filters out low-probability range structures.
