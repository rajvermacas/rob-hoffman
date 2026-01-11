# EMA Trigger-Entry Indicator - Manual Test Cases

## Overview

These test scenarios are designed for manual validation in TradingView. Apply the indicator to various charts and verify behavior matches expected outcomes.

---

## Test Environment Setup

1. Open TradingView and create a new indicator
2. Copy the code from `src/pinescript/trigger_entry_indicator.pine`
3. Apply to a chart (any instrument, any timeframe)
4. Default settings: Fast EMA=10, Slow EMA=20, Touch Buffer=0.3%

---

## Test Cases

### TC-001: Indicator Compilation

**Objective:** Verify indicator compiles and loads without errors

**Steps:**
1. Paste indicator code into Pine Editor
2. Click "Add to chart"

**Expected Results:**
- [ ] No compilation errors
- [ ] Two EMA lines appear on chart (blue and orange/red)
- [ ] No runtime errors in console

---

### TC-002: EMA Line Display

**Objective:** Verify EMA lines display correctly

**Steps:**
1. Apply indicator to chart
2. Verify line colors and positions

**Expected Results:**
- [ ] Yellow line (Fast EMA 10) plots correctly
- [ ] Green line (Slow EMA 20) plots correctly
- [ ] Gray line (Trend EMA 200) plots correctly
- [ ] Fast EMA responds quicker to price changes
- [ ] Lines are continuous with no gaps

---

### TC-017: Wick to Gap Ratio Parameter
**Objective:** Verify Wick to Gap Ratio parameter affects trigger detection
**Steps:**
1. Find a candle where the lower wick is 1.2x the gap to EMA
2. Verify trigger marker APPEARS with default 1.5 ratio (since 1.2 < 1.5)
3. Change "Wick to Gap Ratio" input to 1.0
4. Check for trigger marker
**Expected Results:**
- [ ] 1.5 ratio: Trigger marker APPEARS for 1.2x gap
- [ ] 1.0 ratio: Trigger marker DISAPPEARS for 1.2x gap
- [ ] User can adjust the "tightness" of the trigger geometry

---

### TC-003: Basic Trigger Detection

**Objective:** Verify trigger candles are correctly identified (New Geometry)

**Steps:**
1. Find a candle where:
   - Close is above both EMAs
   - Low is above both EMAs
   - Lower wick is shorter than 1.5x the gap between low and higher EMA
2. Check for trigger marker

**Expected Results:**
- [ ] Amber triangle appears below the trigger candle
- [ ] Yellow dashed horizontal line appears at the high of trigger candle
- [ ] Line extends to the right

**Visual Reference:**
```
        ┌───┐
        │   │
        │   │  Close > both EMAs
        ┴───┤  Low > both EMAs
            │  (short wick - A)
  ══════════│  ← EMA
            │  (larger gap - B)
  ══════════│  ← EMA

      ▲ (Amber marker below bar)
      COND: A < 1.5 * B
```

---

### TC-004: Invalid Trigger - Wick Too Long
**Objective:** Verify candles with long lower wicks are NOT triggers
**Steps:**
1. Find a candle above both EMAs but with:
   - Lower wick LONGER than 1.5x gap to EMA (e.g., wick is 2x gap)
**Expected Results:**
- [ ] NO trigger marker appears
- [ ] NO trigger level line appears

---

### TC-005: Invalid Trigger - Low Below EMA

**Objective:** Verify candles with low below EMA are NOT triggers

**Steps:**
1. Find a candle where:
   - Close is above both EMAs
   - Low is below one or both EMAs (wick pierces EMA)

**Expected Results:**
- [ ] NO trigger marker appears
- [ ] NO trigger level line appears

---

### TC-006: EMA Touch Detection

**Objective:** Verify EMA touch is detected within buffer zone

**Steps:**
1. Wait for a trigger to be set (yellow line visible)
2. Find subsequent candle where low approaches or touches the fast EMA

**Expected Results:**
- [ ] Trigger level line changes from YELLOW to GREEN
- [ ] Touch is detected even if low is slightly above EMA (within 0.3%)

---

### TC-007: Entry Signal - All Conditions Met

**Objective:** Verify entry signal fires when conditions are met

**Steps:**
1. Observe a complete setup:
   - Trigger candle appears (green T, yellow line)
   - Price touches fast EMA (line turns green)
   - Price closes above trigger level

**Expected Results:**
- [ ] Green triangle appears below the entry candle
- [ ] Trigger level line is removed
- [ ] State resets (no active trigger)

---

### TC-008: Entry Blocked - EMA Misaligned

**Objective:** Verify entry is blocked when fast EMA < slow EMA

**Steps:**
1. Find a trigger setup
2. Wait for EMA touch
3. Find situation where fast EMA crosses below slow EMA before breakout

**Expected Results:**
- [ ] NO entry signal even if close > trigger level
- [ ] Trigger remains active until invalidated

---

### TC-009: Trigger Invalidation

**Objective:** Verify trigger invalidates when price closes below slow EMA

**Steps:**
1. Wait for trigger to be set (yellow/green line visible)
2. Wait for a candle to CLOSE below the slow EMA

**Expected Results:**
- [ ] Trigger level line is REMOVED
- [ ] State resets completely
- [ ] New trigger can be set on future candles

---

### TC-010: Trigger Replacement - Higher Level

**Objective:** Verify new trigger with higher high replaces existing trigger

**Steps:**
1. Wait for trigger A to be set (note the trigger level)
2. Find another valid trigger candle B with higher high

**Expected Results:**
- [ ] New amber marker appears below candle B
- [ ] Trigger level line moves UP to new high
- [ ] Previous trigger level is replaced
- [ ] ema_touched flag is RESET to false

---

### TC-011: Trigger NOT Replaced - Lower Level

**Objective:** Verify new trigger with lower/equal high does NOT replace

**Steps:**
1. Wait for trigger A to be set
2. Find another valid trigger candle B with LOWER or EQUAL high

**Expected Results:**
- [ ] NO new trigger marker on candle B
- [ ] Trigger level line stays at original level
- [ ] Original trigger remains active

---

### TC-012: Entry Candle Becomes New Trigger

**Objective:** Verify entry candle can simultaneously become new trigger

**Steps:**
1. Find complete setup ending in entry signal
2. Check if entry candle also satisfies trigger geometry conditions

**Expected Results:**
- [ ] Entry signal fires (green marker)
- [ ] If entry candle meets trigger criteria, new trigger marker (amber) also appears
- [ ] New trigger level line set at entry candle's high

---

### TC-013: Parameter Validation - Invalid Periods

**Objective:** Verify warning when fast_ema_period >= slow_ema_period

**Steps:**
1. Open indicator settings
2. Set Fast EMA Period = 20
3. Set Slow EMA Period = 10 (or equal values)
4. Set Trend EMA Period = 5 (invalid sequence)

**Expected Results:**
- [ ] Warning indicator "!" appears on chart
- [ ] Indicator does not generate signals with invalid config

---

### TC-014: Touch Buffer Adjustment

**Objective:** Verify touch buffer affects detection sensitivity

**Steps:**
1. Set touch buffer to 0% and observe touch detection
2. Set touch buffer to 2% and compare

**Expected Results:**
- [ ] 0% buffer: Touch only detected when low <= exact EMA value
- [ ] 2% buffer: Touch detected when low is within 2% above EMA
- [ ] Higher buffer = more lenient touch detection

---

### TC-015: Data Window Values

**Objective:** Verify debug values in data window

**Steps:**
1. Apply indicator
2. Open TradingView Data Window (press 'D' or from menu)
3. Hover over various bars

**Expected Results:**
- [ ] "Trigger Level" shows current level when trigger active

---

### TC-016: EMA Trend Filter (200 EMA)

**Objective:** Verify trigger candles are only identified when price is in an uptrend (10 & 20 EMA > 200 EMA)

**Steps:**
1. Find a candle that meets all geometry requirements but where:
   - Fast EMA (10) is BELOW Trend EMA (200) OR
   - Slow EMA (20) is BELOW Trend EMA (200)
2. Check for trigger marker

**Expected Results:**
- [ ] NO trigger marker appears
- [ ] NO trigger level line appears
- [ ] Trigger ONLY appears when both 10 and 20 EMAs are above 200 EMA

---

## Edge Case Scenarios

### EC-001: First Bars of Chart

**Objective:** Verify indicator handles beginning of chart data

**Steps:**
1. Scroll to very beginning of chart data

**Expected Results:**
- [ ] EMAs calculate and display (even if not fully warmed up)
- [ ] No errors or crashes
- [ ] Signals start appearing once enough data exists

---

### EC-002: Gap Up/Down Candles

**Objective:** Verify correct handling of gap candles

**Steps:**
1. Find candles with significant gaps from previous close
2. Test both gap-up and gap-down scenarios

**Expected Results:**
- [ ] Trigger detection works correctly with gaps
- [ ] EMA touch detection accounts for gap opens
- [ ] Entry signals fire correctly on gap breakouts

---

### EC-003: Volatile Price Action

**Objective:** Verify stability during high volatility

**Steps:**
1. Apply to volatile instrument or during news events
2. Observe rapid price swings through EMAs

**Expected Results:**
- [ ] Indicator remains stable
- [ ] No excessive signal generation
- [ ] Invalidations occur correctly on volatile drops

---

## Test Log Template

| Test ID | Date | Timeframe | Instrument | Pass/Fail | Notes |
|---------|------|-----------|------------|-----------|-------|
| TC-001 | | | | | |
| TC-002 | | | | | |
| TC-003 | | | | | |
| ... | | | | | |

---

## Regression Testing

When making changes to the indicator, re-run all test cases to ensure no regressions are introduced.

Priority test cases for quick validation:
1. TC-001 (Compilation)
2. TC-003 (Trigger Detection)
3. TC-007 (Entry Signal)
4. TC-009 (Invalidation)
