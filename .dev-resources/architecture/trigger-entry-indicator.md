# Trading View Indicator Architecture
## EMA Trigger-Entry Signal System

**Version:** 1.0
**Date:** 2026-01-11
**Status:** Finalized

---

## Table of Contents

1. [Overview](#overview)
2. [Technology Stack](#technology-stack)
3. [Configurable Parameters](#configurable-parameters)
4. [Core Concepts](#core-concepts)
5. [State Machine](#state-machine)
6. [Algorithm Specification](#algorithm-specification)
7. [Validation Rules](#validation-rules)
8. [Visual Output Specification](#visual-output-specification)
9. [Edge Cases](#edge-cases)
10. [File Structure](#file-structure)

---

## Overview

This indicator identifies high-probability long entry signals based on price action relative to dual EMA structure. The system uses a two-phase approach:

1. **Trigger Phase**: Identifies candles with specific geometric properties relative to EMAs
2. **Entry Phase**: Confirms breakout above trigger level after price retests the fast EMA

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        HIGH-LEVEL EXECUTION FLOW                             │
│                                                                             │
│   [Price Action] ──► [Trigger Detection] ──► [EMA Touch] ──► [Entry Signal] │
│                              │                    │                │        │
│                              ▼                    ▼                ▼        │
│                         Mark Trigger         Track State      Fire Signal   │
│                         Set Level            Update Flag      Reset State   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Component | Technology | Version | Purpose |
|-----------|------------|---------|---------|
| Language | PineScript | v5 | TradingView native scripting |
| Platform | TradingView | - | Charting and execution |
| Indicator Type | Overlay | - | Renders on price chart |

### PineScript Built-in Functions Used

| Function | Purpose |
|----------|---------|
| `ta.ema()` | Calculate Exponential Moving Average |
| `plot()` | Draw EMA lines on chart |
| `plotshape()` | Draw trigger/entry markers |
| `line.new()` | Draw horizontal trigger level line |
| `line.delete()` | Remove previous trigger level line |
| `math.max()` | Determine higher EMA value |

---

## Configurable Parameters

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           INPUT PARAMETERS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Parameter           Type      Default    Range        Description         │
│   ─────────────────────────────────────────────────────────────────────────│
│   fast_ema_period     int       10         1-200        Fast EMA period     │
│   slow_ema_period     int       20         1-200        Slow EMA period     │
│   touch_buffer_pct    float     0.3        0.0-5.0      EMA touch buffer %  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Parameter Validation Rules

- `fast_ema_period` must be less than `slow_ema_period`
- `touch_buffer_pct` is percentage-based (timeframe agnostic)

---

## Core Concepts

### 1. Trigger Candle Definition

A candle qualifies as a **trigger candle** when ALL conditions are met:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TRIGGER CANDLE ANATOMY                               │
│                                                                             │
│        High ────────►  ┬───────────────┐                                    │
│                        │               │                                    │
│        Close ───────►  │   ▓▓▓▓▓▓▓▓▓   │  ◄── Must be > both EMAs          │
│                        │   ▓▓▓▓▓▓▓▓▓   │                                    │
│                        │               │                                    │
│        Low ─────────►  ┴ ─ ─ ─ ─ ─ ─ ─ ┤  A (low wick length)               │
│                                        │                                    │
│                                        │  B (gap to EMA)                    │
│        ════════════════════════════════┤  ◄── Higher EMA (max of 10/20)     │
│                                        │                                    │
│        ════════════════════════════════┘  ◄── Lower EMA                     │
│                                                                             │
│        CONDITIONS:                                                          │
│        ✓ close > fast_ema AND close > slow_ema                              │
│        ✓ low > fast_ema AND low > slow_ema                                  │
│        ✓ A < B  (low wick shorter than gap to higher EMA)                   │
│                                                                             │
│        RESULT: trigger_level = high of this candle                          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. EMA Touch Definition

Price "touches" the fast EMA when:

```
low <= fast_ema * (1 + touch_buffer_pct / 100)
```

This allows for near-touches within the buffer zone without requiring actual contact.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           EMA TOUCH ZONE                                     │
│                                                                             │
│                        │                                                    │
│                    ┌───┴───┐                                                │
│                    │       │                                                │
│                    │       │                                                │
│                    └───┬───┘                                                │
│                        │  ◄── Low of candle                                 │
│   ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ─  ◄── Adjusted EMA (ema * 1.003)        │
│   ═════════════════════│═════════════  ◄── Actual 10 EMA                    │
│                        │                                                    │
│                   TOUCH ZONE                                                │
│                   (0.3% buffer)                                             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3. Entry Candle Definition

A candle qualifies as an **entry candle** when ALL conditions are met:

- `trigger_active == true`
- `ema_touched == true` (at least one candle touched fast EMA since trigger)
- `close > trigger_level`
- `fast_ema > slow_ema` (bullish EMA alignment)
- `bar_index > trigger_bar_index` (not the trigger candle itself)

---

## State Machine

### State Variables

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           STATE VARIABLES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Variable            Type     Initial    Description                       │
│   ─────────────────────────────────────────────────────────────────────────│
│   trigger_active      bool     false      Is a trigger currently active?    │
│   trigger_level       float    0.0        High of trigger candle            │
│   trigger_bar_index   int      0          Bar index of trigger candle       │
│   ema_touched         bool     false      Has price touched fast EMA?       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### State Transitions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        STATE TRANSITION DIAGRAM                              │
│                                                                             │
│                                                                             │
│    ┌──────────────┐         Valid Trigger         ┌──────────────────────┐  │
│    │              │ ─────────────────────────────►│                      │  │
│    │    IDLE      │                               │   TRIGGER ACTIVE     │  │
│    │              │◄───────────────────────────── │   (waiting touch)    │  │
│    └──────────────┘     Close < slow_ema          └──────────┬───────────┘  │
│           ▲              (invalidation)                      │              │
│           │                                                  │              │
│           │                                        Low <= adjusted_ema      │
│           │                                                  │              │
│           │                                                  ▼              │
│           │                                       ┌──────────────────────┐  │
│           │         Close > trigger_level         │                      │  │
│           │         AND ema_fast > ema_slow       │   TRIGGER ACTIVE     │  │
│           └────────────────────────────────────── │   (EMA touched)      │  │
│                       (ENTRY SIGNAL)              └──────────────────────┘  │
│                                                                             │
│                                                                             │
│    ┌─────────────────────────────────────────────────────────────────────┐  │
│    │  NOTE: New valid trigger with higher level can occur in any state   │  │
│    │        This replaces the current trigger and resets ema_touched     │  │
│    └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Algorithm Specification

### Main Execution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MAIN LOOP                                       │
│                         (executes on each candle close)                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 1: CALCULATE INDICATORS                                                │
│                                                                             │
│  ema_fast ← ta.ema(close, fast_ema_period)                                  │
│  ema_slow ← ta.ema(close, slow_ema_period)                                  │
│  adjusted_ema ← ema_fast * (1 + touch_buffer_pct / 100)                     │
│  higher_ema ← math.max(ema_fast, ema_slow)                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 2: CHECK INVALIDATION (if trigger active)                              │
│                                                                             │
│  IF trigger_active AND close < ema_slow:                                    │
│      trigger_active ← false                                                 │
│      ema_touched ← false                                                    │
│      // Trigger invalidated - candle closed below slow EMA                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 3: CHECK EMA TOUCH (if trigger active and not yet touched)             │
│                                                                             │
│  IF trigger_active AND NOT ema_touched:                                     │
│      IF low <= adjusted_ema:                                                │
│          ema_touched ← true                                                 │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 4: CHECK ENTRY CONDITIONS                                              │
│                                                                             │
│  entry_signal ← false                                                       │
│                                                                             │
│  IF trigger_active AND                                                      │
│     ema_touched AND                                                         │
│     close > trigger_level AND                                               │
│     ema_fast > ema_slow AND                                                 │
│     bar_index > trigger_bar_index:                                          │
│                                                                             │
│      entry_signal ← true                                                    │
│      trigger_active ← false                                                 │
│      ema_touched ← false                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 5: CHECK FOR NEW TRIGGER CANDLE                                        │
│                                                                             │
│  low_wick ← close - low                                                     │
│  gap_to_ema ← low - higher_ema                                              │
│                                                                             │
│  is_valid_trigger ← (close > ema_fast) AND                                  │
│                     (close > ema_slow) AND                                  │
│                     (low > ema_fast) AND                                    │
│                     (low > ema_slow) AND                                    │
│                     (gap_to_ema > 0) AND                                    │
│                     (low_wick < gap_to_ema)                                 │
│                                                                             │
│  IF is_valid_trigger:                                                       │
│      new_level ← high                                                       │
│      IF (NOT trigger_active) OR (new_level > trigger_level):                │
│          trigger_level ← new_level                                          │
│          trigger_active ← true                                              │
│          trigger_bar_index ← bar_index                                      │
│          ema_touched ← false                                                │
│          trigger_signal ← true                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  STEP 6: RENDER VISUALS                                                      │
│                                                                             │
│  - Plot EMA lines                                                           │
│  - Plot trigger marker if trigger_signal                                    │
│  - Plot entry marker if entry_signal                                        │
│  - Draw/update trigger level line if trigger_active                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Detailed Pseudocode

```
FUNCTION process_candle():

    // ═══════════════════════════════════════════════════════════════════════
    // STEP 1: Calculate indicators
    // ═══════════════════════════════════════════════════════════════════════
    ema_fast ← EMA(close, fast_ema_period)
    ema_slow ← EMA(close, slow_ema_period)
    adjusted_ema_for_touch ← ema_fast * (1 + touch_buffer_pct / 100)
    higher_ema ← MAX(ema_fast, ema_slow)

    entry_signal ← false
    trigger_signal ← false

    // ═══════════════════════════════════════════════════════════════════════
    // STEP 2: If trigger is active, check invalidation first
    // ═══════════════════════════════════════════════════════════════════════
    IF trigger_active:
        IF close < ema_slow:
            // INVALIDATION: Candle closed below slow EMA
            trigger_active ← false
            ema_touched ← false
        ELSE:
            // ═══════════════════════════════════════════════════════════════
            // STEP 3: Check if this candle touches fast EMA
            // ═══════════════════════════════════════════════════════════════
            IF NOT ema_touched:
                IF low <= adjusted_ema_for_touch:
                    ema_touched ← true

            // ═══════════════════════════════════════════════════════════════
            // STEP 4: Check entry conditions
            // ═══════════════════════════════════════════════════════════════
            IF ema_touched AND
               close > trigger_level AND
               ema_fast > ema_slow AND
               bar_index > trigger_bar_index:

                entry_signal ← true
                trigger_active ← false
                ema_touched ← false

    // ═══════════════════════════════════════════════════════════════════════
    // STEP 5: Check if current candle is a valid trigger
    //         (runs after entry check - allows entry candle to be new trigger)
    // ═══════════════════════════════════════════════════════════════════════
    low_wick_length ← close - low
    gap_to_ema ← low - higher_ema

    is_valid_trigger ← (close > ema_fast) AND
                       (close > ema_slow) AND
                       (low > ema_fast) AND
                       (low > ema_slow) AND
                       (gap_to_ema > 0) AND
                       (low_wick_length < gap_to_ema)

    IF is_valid_trigger:
        new_trigger_level ← high

        // Only update if no active trigger OR new level is higher
        IF (NOT trigger_active) OR (new_trigger_level > trigger_level):
            trigger_level ← new_trigger_level
            trigger_active ← true
            trigger_bar_index ← bar_index
            ema_touched ← false
            trigger_signal ← true

    // ═══════════════════════════════════════════════════════════════════════
    // STEP 6: Return signals for plotting
    // ═══════════════════════════════════════════════════════════════════════
    RETURN {
        entry_signal,
        trigger_signal,
        trigger_level,
        trigger_active,
        ema_touched
    }
```

---

## Validation Rules

### Trigger Candle Geometry

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VALID vs INVALID TRIGGER GEOMETRY                         │
│                                                                             │
│         VALID TRIGGER                         INVALID TRIGGERS              │
│                                                                             │
│    ┌─────────────────────┐            ┌─────────────────────────────────┐   │
│    │                     │            │                                 │   │
│    │    ┬───────         │            │    ┬───────      ┬───────      │   │
│    │    │       │        │            │    │       │     │       │     │   │
│    │    │ ▓▓▓▓▓ │        │            │    │ ▓▓▓▓▓ │     │ ▓▓▓▓▓ │     │   │
│    │    │ ▓▓▓▓▓ │        │            │    │ ▓▓▓▓▓ │     │ ▓▓▓▓▓ │     │   │
│    │    │       │        │            │    │       │     │       │     │   │
│    │    ┴ ─ ─ ─ ┤ A      │            │    │       │     ┴ ─ ─ ─ ┤ A   │   │
│    │            │        │            │    ┴ ─ ─ ─ ┤ A ══════════     │   │
│    │    ════════│════    │            │  ══════════│       │     B    │   │
│    │            │ B      │            │            │ B   ══│══════    │   │
│    │    ════════│════    │            │  ══════════│       │          │   │
│    │            │        │            │            │     ──┘          │   │
│    │                     │            │                                 │   │
│    │    A < B  ✓         │            │    A > B  ✗      Low < EMA ✗   │   │
│    │    Low > EMAs ✓     │            │                                 │   │
│    │                     │            │                                 │   │
│    └─────────────────────┘            └─────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Mathematical Conditions Summary

| Condition | Formula | Description |
|-----------|---------|-------------|
| Close above EMAs | `close > ema_fast AND close > ema_slow` | Price closed above both EMAs |
| Low above EMAs | `low > ema_fast AND low > ema_slow` | Entire candle body above EMAs |
| Wick ratio | `(close - low) < (low - max(ema_fast, ema_slow))` | Low wick shorter than gap |
| EMA touch | `low <= ema_fast * 1.003` | Price within touch buffer |
| Entry valid | `ema_fast > ema_slow` | Bullish EMA alignment |

---

## Visual Output Specification

### Chart Elements

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           CHART ELEMENTS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Element              Type            Color           Description           │
│  ──────────────────────────────────────────────────────────────────────────│
│  Fast EMA Line        plot()          Blue (#2962FF)  10-period EMA         │
│  Slow EMA Line        plot()          Red (#FF6D00)   20-period EMA         │
│  Trigger Marker       plotshape()     Green           Triangle below bar    │
│  Entry Marker         plotshape()     Lime            Triangle above bar    │
│  Trigger Level        line.new()      Yellow/Green    Horizontal dashed     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Visual Timeline Example

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VISUAL TIMELINE EXAMPLE                              │
│                                                                             │
│  Bar:     1        2        3        4        5        6        7           │
│           │        │        │        │        │        │        │           │
│           │    ┌───┤    ┌───┤        │    ┌───┤    ┌───┤    ┌───┤           │
│           │    │   │    │   │    ┌───┤    │   │    │   │    │   │           │
│   ════════│════╧═══│════╧═══│════╡   ╞════╧═══│════╧═══│════╧═══│══ 10 EMA  │
│           │        │        │    │   │        │        │        │           │
│   ════════│════════│════════│════╧═══│════════│════════│════════│══ 20 EMA  │
│           │        │        │        │        │        │        │           │
│           ▲                          *        *                 ▼           │
│        TRIGGER                    TOUCH    TOUCH             ENTRY          │
│        (level=100)               10 EMA   10 EMA          (close>100)       │
│                                                                             │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─  100   │
│  [YELLOW]                        [GREEN after touch]                        │
│                                                                             │
│  Legend:                                                                    │
│  ▲ = Trigger marker (below bar)                                             │
│  ▼ = Entry marker (above bar)                                               │
│  * = EMA touch detected                                                     │
│  ─ ─ = Trigger level line                                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Trigger Level Line Color States

| State | Color | Meaning |
|-------|-------|---------|
| Waiting for touch | Yellow | Trigger active, EMA not yet touched |
| Touch confirmed | Green | EMA touched, waiting for breakout |
| No active trigger | Hidden | Line removed |

---

## Edge Cases

### Handled Scenarios

| Scenario | Expected Behavior |
|----------|-------------------|
| Entry candle is also valid trigger | Entry signal fires first, then same candle becomes new trigger for next setup |
| New trigger with higher high | Replaces previous trigger level, resets `ema_touched` |
| New trigger with lower/equal high | Ignored, keeps previous trigger level |
| Candle closes below slow EMA | Invalidates trigger, resets all state |
| 10 EMA crosses below 20 EMA after trigger | Entry blocked until 10 > 20 again |
| Gap between low and EMA is zero/negative | Not a valid trigger (low must be above both EMAs) |
| Trigger candle with open > close (red candle) | Valid if conditions met (close still used for wick calculation) |
| First few bars (EMA not stabilized) | EMAs calculate from available data, indicator starts working immediately |

### Invalidation Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         INVALIDATION SCENARIO                                │
│                                                                             │
│  Bar:     1        2        3        4        5                             │
│           │        │        │        │        │                             │
│           │    ┌───┤    ┌───┤    ┌───┤        │                             │
│           │    │   │    │   │    │   │    ╔═══╗                             │
│   ════════│════╧═══│════╧═══│════╧═══│════║   ║═══════════════════ 10 EMA  │
│           │        │        │        │    ║   ║                             │
│   ════════│════════│════════│════════│════╚═══╝═══════════════════ 20 EMA  │
│           │        │        │        │        │                             │
│           ▲                                   X                             │
│        TRIGGER                           INVALIDATED                        │
│                                       (close < 20 EMA)                      │
│                                                                             │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ │                                 │
│  [YELLOW]                           [REMOVED]                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## File Structure

```
src/
└── pinescript/
    └── trigger_entry_indicator.pine    # Main indicator script

tests/
└── test_cases.md                       # Manual test scenarios for TradingView

.dev-resources/
├── architecture/
│   └── trigger-entry-indicator.md      # This document
└── prompts/
    └── .txt                            # Original requirements
```

---

## Implementation Notes

### PineScript v5 Specifics

1. **State Persistence**: Use `var` keyword for variables that persist across bars
2. **Line Drawing**: Use `line.new()` with `xloc.bar_index` for trigger level
3. **Line Updates**: Delete previous line before drawing new one to avoid clutter
4. **Shape Plotting**: Use `plotshape()` with `location.belowbar` / `location.abovebar`

### Performance Considerations

- EMA calculations are O(1) per bar (built-in function)
- State machine is O(1) per bar
- Line management: only one active line at a time
- No lookback beyond current bar state (memory efficient)

---

## Revision History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-01-11 | Initial architecture specification |
