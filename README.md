# EMA Trigger-Entry Signal Indicator

A TradingView PineScript indicator that identifies high-probability long entry signals based on price action relative to dual EMA structure.

## Overview

This indicator uses a two-phase approach:

1. **Trigger Phase**: Identifies candles with specific geometric properties relative to EMAs
2. **Entry Phase**: Confirms breakout above trigger level after price retests the fast EMA

```
+-----------------------------------------------------------------------------+
|                        HIGH-LEVEL EXECUTION FLOW                            |
|                                                                             |
|   [Price Action] --> [Trigger Detection] --> [EMA Touch] --> [Entry Signal] |
|                              |                    |                |        |
|                              v                    v                v        |
|                         Mark Trigger         Track State      Fire Signal   |
|                         Set Level            Update Flag      Reset State   |
+-----------------------------------------------------------------------------+
```

## Installation

1. Open TradingView and navigate to the Pine Editor
2. Copy the contents of `src/pinescript/trigger_entry_indicator.pine`
3. Paste into the Pine Editor
4. Click "Add to chart"

## Parameters

| Parameter | Type | Default | Range | Description |
|-----------|------|---------|-------|-------------|
| Fast EMA Period | int | 10 | 1-200 | Period for the fast EMA |
| Slow EMA Period | int | 20 | 1-200 | Period for the slow EMA |
| Touch Buffer % | float | 0.3 | 0.0-5.0 | Buffer zone for EMA touch detection |

## Signal Logic

### Trigger Candle Requirements

A candle qualifies as a trigger when ALL conditions are met:

```
        +-------+
        |       |
        | Close |  --> Must be > both EMAs
        |       |
        +---+---+  A (low wick length)
            |
            |      B (gap to EMA)
  ==========+====  <-- Higher EMA
            |
  ===============  <-- Lower EMA

  CONDITIONS:
  - close > fast_ema AND close > slow_ema
  - low > fast_ema AND low > slow_ema
  - A < B (low wick shorter than gap to higher EMA)
```

### EMA Touch Detection

Price "touches" the fast EMA when:
```
low <= fast_ema * (1 + touch_buffer_pct / 100)
```

This allows for near-touches within the buffer zone without requiring actual contact.

### Entry Signal Conditions

An entry signal fires when:
- Trigger is active
- EMA has been touched since trigger
- Close is above trigger level
- Fast EMA > Slow EMA (bullish alignment)
- Current bar is after trigger bar

## Visual Elements

| Element | Type | Color | Description |
|---------|------|-------|-------------|
| Fast EMA | Line | Blue (#2962FF) | 10-period EMA |
| Slow EMA | Line | Orange (#FF6D00) | 20-period EMA |
| Trigger Marker | Triangle | Green | Below bar, marks trigger candle |
| Entry Marker | Triangle | Lime | Above bar, marks entry signal |
| Trigger Level | Dashed Line | Yellow/Green | Horizontal at trigger high |

### Trigger Level Line Colors

- **Yellow**: Waiting for EMA touch
- **Green**: EMA touched, waiting for breakout

## State Machine

```
+-------------+         Valid Trigger         +--------------------+
|             | ----------------------------> |                    |
|    IDLE     |                               |   TRIGGER ACTIVE   |
|             |<----------------------------- |   (waiting touch)  |
+-------------+     Close < slow_ema          +--------+-----------+
       ^              (invalidation)                   |
       |                                               | Low <= adjusted_ema
       |                                               v
       |                                     +--------------------+
       |         Close > trigger_level       |                    |
       |         AND ema_fast > ema_slow     |   TRIGGER ACTIVE   |
       +------------------------------------ |   (EMA touched)    |
                    (ENTRY SIGNAL)           +--------------------+
```

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Entry candle is also valid trigger | Entry fires, then same candle becomes new trigger |
| New trigger with higher high | Replaces previous trigger, resets touch flag |
| New trigger with lower/equal high | Ignored, keeps previous trigger |
| Candle closes below slow EMA | Invalidates trigger, resets all state |
| Fast EMA crosses below slow EMA | Entry blocked until realigned |

## Project Structure

```
.
+-- src/
|   +-- pinescript/
|       +-- trigger_entry_indicator.pine    # Main indicator
+-- tests/
|   +-- test_cases.md                       # Manual test scenarios
+-- .dev-resources/
|   +-- architecture/
|       +-- trigger-entry-indicator.md      # Architecture specification
+-- .gitignore
+-- README.md
```

## Testing

See `tests/test_cases.md` for comprehensive manual test scenarios to validate indicator behavior in TradingView.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-11 | Initial release |

## License

[Add license information here]
