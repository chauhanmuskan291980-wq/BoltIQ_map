# BoltIQ — Day 2 Research Report
## Strategy Execution, Risk Architecture & Order-Flow Mapping

**Project:** BoltIQ Gold Portfolio Engine  
**Research Phase:** Day 2  
**Focus:** Strategy-level execution, risk sizing, order placement, safety gates and execution architecture  
**Status:** Code-level tracing complete  
**Next Phase:** Day 3 — Quantitative Validation

---

## 1. Executive Summary

Day 2 focused on tracing how trading signals move through the BoltIQ system from strategy-level decision making to actual MT5 order submission.

The objective was not to modify code or optimize strategies.

The objective was to answer:

> **When a strategy decides to trade, what risk controls, sizing mechanisms, validation gates and execution paths are actually applied before the order reaches MT5?**

The review covered:

- Gold Elite / London Breakout
- Portfolio / Legacy strategies
- High Trade
- Micro Scalp
- Common market-order execution
- Risk-volume calculation
- Portfolio-risk checks
- SL/TP construction
- Execution fallback mechanisms
- Position tracking
- Magic-number ownership
- Anti-duplicate controls
- ONNX-related gates
- Daily loss and capital-preservation controls

### Main architectural conclusion

BoltIQ does **not** use one single universal risk-sizing architecture.

Instead, several execution/risk architectures operate inside the same EA:

```text
                         BOLT IQ EA
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
          ▼                  ▼                  ▼
     Portfolio /         High Trade /       Gold Elite /
       Legacy            Micro Scalp        London Breakout
          │                  │                  │
          ▼                  ▼                  ▼
   Legacy minimum       Dynamic risk %      Fixed GE lot
        lot                  │                  │
          │                  ▼                  ▼
          │            CalculateRiskVolume   Pending orders
          │                  │                  │
          ▼                  ▼                  ▼
   Common market       Common market        Direct GE
      execution           execution         execution
          │                  │                  │
          └──────────────────┼──────────────────┘
                             ▼
                            MT5
```

This does **not** automatically mean that any architecture is incorrect. It means that the risk model needs to be evaluated at the architecture level before making claims about system-wide risk consistency.

---

## 2. Day 2 Objective

The research framework used for this review is:

```text
Observation
    ↓
Evidence
    ↓
Hypothesis
    ↓
Proposed Change
    ↓
Backtest / Validation
    ↓
Expected Impact
```

Day 2 therefore concentrated on:

1. Mapping strategy execution paths.
2. Identifying where risk is calculated.
3. Identifying where risk is enforced.
4. Identifying where different strategies bypass common infrastructure.
5. Mapping order-volume calculation.
6. Mapping SL/TP calculation.
7. Mapping execution/filling behavior.
8. Mapping duplicate-order protection.
9. Identifying architecture coupling.
10. Preparing measurable validation questions for Day 3.

**No production strategy changes were made during this review.**

---

## 3. Scope of Review

| Area | Reviewed |
|---|---:|
| High Trade | Yes |
| Micro Scalp | Yes |
| Portfolio / Legacy | Yes |
| Gold Elite / London Breakout | Yes |
| Common market execution | Yes |
| Dynamic risk sizing | Yes |
| Fixed-lot sizing | Yes |
| Legacy volume path | Yes |
| SL/TP construction | Yes |
| Hard per-trade risk boundary | Yes |
| Portfolio risk | Yes |
| Daily loss protection | Yes |
| Capital-preservation gates | Yes |
| News protection | Yes |
| Spread protection | Yes |
| Margin protection | Yes |
| ONNX gate integration | Yes |
| Duplicate-order protection | Yes |
| Magic-number ownership | Yes |
| Post-fill tracking | Yes |
| Actual historical slippage | Not yet quantitatively validated |
| Actual realized risk distribution | Not yet quantitatively validated |
| Strategy profitability | Day 3 |
| Out-of-sample performance | Day 3 |

---

## 4. High-Level Execution Architecture

```text
                              ┌─────────────────────┐
                              │     BoltIQ EA       │
                              └──────────┬──────────┘
                                         │
              ┌──────────────────────────┼──────────────────────────┐
              │                          │                          │
              ▼                          ▼                          ▼
     ┌─────────────────┐       ┌─────────────────┐       ┌──────────────────┐
     │ Portfolio/Legacy│       │ High Trade /    │       │ Gold Elite       │
     │ Strategies      │       │ Micro Scalp     │       │ London Breakout  │
     └────────┬────────┘       └────────┬────────┘       └────────┬─────────┘
              │                         │                         │
              ▼                         ▼                         ▼
      Mathematical SL/TP        ATR-based SL/TP             GE SL/TP
              │                         │                         │
              ▼                         ▼                         ▼
      Legacy volume path        Dynamic risk-volume        GE_FixedLotSize
              │                         │                         │
              ▼                         ▼                         ▼
      PassesQualityFilter       Strategy gate              GE-specific guards
              │                         │                         │
              ▼                         ▼                         ▼
      PlaceMarketOrder()        PlaceMarketOrderVolume()    GE_PlacePendingOrder()
              │                         │                         │
              ▼                         ▼                         ▼
       SendMarketOrder()        SendMarketOrder()          OrderSend()
              │                         │                         │
              ▼                         ▼                         ▼
      Common hard-risk cap     Common hard-risk cap        Separate path
              │                         │                         │
              └─────────────────────────┼─────────────────────────┘
                                        ▼
                                       MT5
```

---

## 5. Finding RISK-SIZING-001
### Multiple Risk-Sizing Architectures Exist

### Observation

BoltIQ does not have one universal position-sizing mechanism.

Different strategy families use different approaches.

### Architecture A — Dynamic Risk Percentage

Used by:

- High Trade
- Micro Scalp

Conceptually:

```text
Account Equity
      │
      ▼
Risk %
      │
      ▼
Risk Money
      │
      ▼
SL Distance
      │
      ▼
Value per Price Unit
      │
      ▼
Position Volume
      │
      ▼
Margin Cap
      │
      ▼
Final Volume
```

The requested risk is also capped by:

```text
BOLTIQ_MAX_RISK_PER_TRADE_PCT
```

### Architecture B — Legacy Minimum-Lot Volume

The reviewed Portfolio/Legacy path can ultimately use:

```text
LegacyOrderVolume()
        ↓
SYMBOL_VOLUME_MIN
        ↓
NormalizeTradeVolume()
```

The actual submitted volume can therefore be the broker's minimum normalized volume rather than a dynamic risk-derived volume.

### Architecture C — Gold Elite Fixed Lot

Gold Elite uses:

```text
GE_FixedLotSize
```

directly inside its pending-order request.

```text
GE strategy
     ↓
GE SL/TP
     ↓
GE_FixedLotSize
     ↓
Pending Order
```

### Why this matters

Multiple sizing architectures are not inherently wrong.

However, each architecture must be quantitatively validated:

> **Does each architecture produce the intended actual risk percentage under realistic broker conditions?**

Code structure alone cannot answer that.

---

## 6. Finding RISK-CALC-002
### Portfolio Mathematical SL/TP Uses Fixed-Lot Assumptions

The reviewed mathematical SL/TP logic calculates monetary risk using:

```text
Balance
×
StopLossPercent
```

and converts that monetary value into price distance using:

```text
FixedLotSize
×
VALUE_PER_LOT
```

Conceptually:

```text
Expected Risk Money
        │
        ▼
Fixed Lot Assumption
        │
        ▼
VALUE_PER_LOT
        │
        ▼
SL Price Distance
```

However, the downstream Portfolio execution path can use:

```text
LegacyOrderVolume()
```

for the actual order volume.

Therefore:

```text
Nominal SL Risk
        ≠ necessarily
Actual Submitted-Volume Risk
```

This is an architecture observation, **not yet a confirmed defect**.

The next step is to compare:

```text
Expected Risk
vs
Actual Risk
```

using historical or controlled MT5 executions.

---

## 7. Finding RISK-CALC-003
### Multiple Price-Value Calculation Mechanisms Exist

The system uses more than one mechanism for converting price movement into monetary value.

One architecture uses:

```text
VALUE_PER_LOT
```

while the dynamic risk engine uses broker-provided:

```text
SYMBOL_TRADE_TICK_VALUE
SYMBOL_TRADE_TICK_SIZE
```

Conceptually:

```text
Broker Tick Value
        ÷
Broker Tick Size
        =
Value Per Price Unit
```

Gold-related logic also contains a fallback involving `VALUE_PER_LOT` for certain gold-symbol conditions.

### Why this matters

A risk engine can only be trusted if:

```text
Price Distance
        ×
Volume
        ×
Value Per Price Unit
```

accurately represents broker monetary exposure.

Day 3 should validate this against actual MT5 deal results.

---

## 8. High Trade Execution Path

```text
High Trade Signal
        │
        ▼
TryOpenHighTrade()
        │
        ├── Strategy enabled?
        ├── Direction valid?
        ├── Cooldown?
        ├── Existing position?
        └── Branch guards
        │
        ▼
CalculateHighTradeSLTP()
        │
        ▼
ATR-based SL / TP
        │
        ▼
EffectiveHighTradeRiskPct()
        │
        ▼
CalculateRiskVolumePct()
        │
        ▼
Margin Cap
        │
        ▼
PassesHighTradeGate()
        │
        ├── Emergency stop
        ├── Permissions
        ├── Daily loss
        ├── Daily trades
        ├── Spread
        ├── News
        ├── Volatility
        ├── Capital preservation
        ├── Open-position limits
        ├── Direction controls
        ├── Portfolio risk
        ├── Margin checks
        └── ONNX/confirmation conditions
        │
        ▼
PlaceMarketOrderVolume()
        │
        ▼
SendMarketOrder()
        │
        ▼
Hard Risk Cap
        │
        ▼
OrderSend()
        │
        ▼
MT5
```

This separates:

- signal generation,
- SL/TP generation,
- volume calculation,
- strategy-level gate,
- common execution,
- final hard-risk check.

---

## 9. High Trade Gate

The High Trade gate checks multiple categories of protection.

### Safety

```text
Emergency stop
Emergency close
Trading permissions
Invalid volume
```

### Time

```text
Weekend
Trading hours
Winning-hour restrictions
```

### Account protection

```text
Daily loss
Capital preservation
Maximum open positions
```

### Market conditions

```text
Spread
Volatility
News
```

### Strategy constraints

```text
Direction
Mixed-direction restrictions
Confirmation
ONNX conditions
Daily probe limits
```

### Risk

```text
Portfolio risk
Margin projection
```

The important point is that High Trade does not simply calculate volume and immediately send an order. It passes through multiple layers of gating.

---

## 10. Micro Scalp Execution Path

```text
Micro Scalp Signal
        │
        ▼
CalculateMicroScalpSLTP()
        │
        ▼
ATR / Base Stop
        │
        ▼
Reward/Risk Constraint
        │
        ▼
CalculateMicroScalpVolume()
        │
        ▼
EffectiveMicroScalpRiskPct()
        │
        ▼
CalculateRiskVolumePct()
        │
        ▼
PassesMicroScalpGate()
        │
        ▼
PlaceMarketOrderVolume()
        │
        ▼
SendMarketOrder()
        │
        ▼
Hard Risk Cap
        │
        ▼
MT5
```

The Micro Scalp gate includes:

- emergency controls,
- daily loss,
- daily trade count,
- spread,
- hourly quota,
- news,
- capital preservation,
- maximum open positions,
- direction restrictions,
- portfolio-risk check,
- projected-margin check.

---

## 11. Common Market Execution Path

```text
SendMarketOrder()
       │
       ├── Daily Entry Stop
       ├── Weekend Block
       ├── Winning Hour
       ├── News Block
       ├── Trading Session
       ├── Normalize Volume
       │
       ▼
IsOrderRiskWithinHardCap()
       │
       ▼
Prepare Telemetry
       │
       ▼
MqlTradeRequest
       │
       ▼
SendOrderWithFillingFallback()
       │
       ▼
MT5
```

The important safety boundary is:

```text
Final Submitted Volume
        +
Current Entry Price
        +
SL
        ↓
Hard Risk Cap
```

Therefore strategies that eventually pass through `SendMarketOrder()` receive a final common risk boundary.

---

## 12. Hard Per-Trade Risk Boundary

The common market path checks:

```text
OrderRiskPct(entry, SL, volume)
```

against:

```text
BOLTIQ_MAX_RISK_PER_TRADE_PCT
```

Architecture:

```text
Strategy calculates volume
            │
            ▼
Common execution receives volume
            │
            ▼
Normalize volume
            │
            ▼
Calculate actual order risk
            │
            ▼
Compare against hard cap
            │
       ┌────┴────┐
       │         │
      PASS      FAIL
       │         │
       ▼         ▼
    OrderSend   Reject
```

This evaluates final order parameters instead of trusting only the strategy's original risk calculation.

---

## 13. Important Boundary: GE Does Not Use the Same Market Path

Gold Elite is structurally different.

Common market path:

```text
Strategy
   ↓
SendMarketOrder()
   ↓
Hard Risk Cap
   ↓
OrderSend()
```

GE path:

```text
GE_RunDailyLondonBreakout()
   ↓
GE_CalculateSLTP()
   ↓
GE_PlacePendingOrder()
   ↓
TRADE_ACTION_PENDING
   ↓
OrderSend()
```

Therefore the reviewed GE pending-order path does not visibly pass through:

```text
SendMarketOrder()
```

and does not visibly pass through:

```text
IsOrderRiskWithinHardCap()
```

inside this call chain.

---

## 14. Finding GE-RISK-001
### Gold Elite Pending Orders Use a Separate Risk Boundary

### Observation

Gold Elite directly submits pending orders using:

```cpp
req.volume = GE_FixedLotSize;
```

### Evidence

```text
GE_RunDailyLondonBreakout()
        ↓
GE_CalculateSLTP()
        ↓
GE_PlacePendingOrder()
        ↓
GE_FixedLotSize
        ↓
TRADE_ACTION_PENDING
        ↓
OrderSend()
```

There is no visible common market-order hard-risk check in this path.

### Verified conclusion

The GE pending-order architecture is separate from the common market-order hard-risk architecture.

### What is NOT yet proven

It has **not** been proven that GE exceeds the intended risk limit.

That requires actual numerical validation.

### Required validation

For each GE trade:

```text
Actual Risk Money
=
|Entry - SL|
×
Volume
×
Value Per Price Unit
```

Then:

```text
Actual Risk %
=
Actual Risk Money
÷
Account Equity
×
100
```

Compare this with:

```text
BOLTIQ_MAX_RISK_PER_TRADE_PCT
```

over a meaningful sample.

---

## 15. Gold Elite London Breakout Execution

```text
GE_RunDailyLondonBreakout()
        │
        ├── Already traded?
        ├── Orders already placed?
        │
        ▼
    London hour?
        │
        ▼
  Static execution guard
        │
        ▼
GE_HasLiveBreakoutOrders()
        │
        ▼
Trading permissions
        │
        ▼
GE_CalcATR()
        │
        ▼
GE_CalcSR()
        │
        ▼
Calculate breakout levels
        │
        ├───────────────┐
        ▼               ▼
 BUY STOP           SELL STOP
        │               │
        ▼               ▼
GE_CalculateSLTP   GE_CalculateSLTP
        │               │
        ▼               ▼
GE_PlacePendingOrder()
        │               │
        └───────┬───────┘
                ▼
        Both successful?
          │           │
         YES          NO
          │           │
          ▼           ▼
       Keep        Cancel any
       both        successful leg
```

---

## 16. GE Anti-Duplicate Protection

The GE implementation contains an explicit market-state idempotence mechanism.

It checks:

```text
Pending Orders
       +
Open Position
       ↓
symbol + MAGIC_LONDON_BREAKOUT
```

before creating a new straddle.

This protects against EA restart/reinitialization.

The function also uses:

```text
lastExecDay
lastExecHour
```

as static variables.

Static variables can reset after reinitialization, so the market-state check is important because it queries actual MT5 orders and positions.

Architecture:

```text
Static Guard
     +
Market-State Guard
     ↓
Duplicate Protection
```

---

## 17. Finding TELEMETRY-001
### Post-Fill Position Tracking Requires Validation Under Same-Magic Overlap

The common market execution path scans positions using:

```text
symbol + magic
```

and can track the first matching position.

This is generally sufficient when one symbol + one magic corresponds to one active position.

However, some strategies allow overlap or multiple trades under the same magic.

Validation question:

> Can multiple positions exist simultaneously with the same symbol and magic, and if so, does post-fill tracking associate the correct deal with the correct strategy event?

This is **not classified as a defect**. It is a telemetry integrity validation item.

---

## 18. Finding PORTFOLIO-RISK-003
### Reviewed Aggregate Portfolio Risk Is Symbol-Scoped

The reviewed `IsPortfolioRiskOkVolume()` logic filters positions using:

```text
if(symbol != _Symbol)
    continue;
```

Therefore the reviewed risk calculation aggregates managed positions for the current symbol rather than automatically aggregating every managed symbol.

Conceptually:

```text
Account
 ├── XAUUSD positions ──┐
 ├── BTCUSD positions   │
 ├── EURUSD positions   │
 └── GBPUSD positions   │
                         │
                Current symbol
                       │
                       ▼
              Risk calculation
```

The correct description is:

> **symbol-scoped aggregate risk in the reviewed function**

rather than automatically calling it a full account-wide portfolio-risk calculation.

---

## 19. Why Symbol Scope Matters

BoltIQ is a multi-symbol engine:

```text
XAUUSD
BTCUSD
EURUSD
GBPUSD
```

If the intended risk control is:

```text
Maximum risk across the entire account
```

then a symbol-scoped calculation would not fully represent cross-symbol exposure.

If the intended control is:

```text
Maximum risk per symbol
```

then the current behavior may be consistent with that design.

Validation should determine whether the intended product policy is:

```text
A. Maximum risk per symbol

or

B. Maximum risk across all managed symbols

or

C. Both
```

Do not change implementation until the intended risk policy is confirmed.

---

## 20. SL/TP Architecture Comparison

| Strategy Family | SL/TP Method | Volume Method | Common Hard Cap |
|---|---|---|---|
| High Trade | ATR-based | Dynamic risk % | Yes, via common market path |
| Micro Scalp | Base + ATR | Dynamic risk % | Yes, via common market path |
| Portfolio/Legacy | Mathematical | Legacy volume path | Yes, via common market path |
| Gold Elite | Mathematical GE logic | `GE_FixedLotSize` | Separate pending path |

This table is the starting point for Day 3 quantitative validation.

---

## 21. Execution/Filling Architecture

The common market path uses a filling fallback mechanism:

```text
Market Order
     │
     ▼
Preferred Filling
     │
     ├── Success
     │
     └── Unsupported
             │
             ▼
       Alternative Filling
             │
             ▼
            MT5
```

GE has its own pending-order fallback:

```text
ORDER_FILLING_RETURN
        │
        ▼
If unsupported
        │
        ▼
ORDER_FILLING_IOC
        │
        ▼
If unsupported
        │
        ▼
ORDER_FILLING_FOK
```

Execution fallback logic is therefore not completely centralized.

---

## 22. `deviation = 50` Interpretation

The reviewed execution requests contain:

```text
deviation = 50
```

This should **not** be interpreted as measured realized slippage.

It represents execution tolerance.

Actual slippage must be calculated from MT5 deal history:

```text
Requested Price
       vs
Actual Fill Price
       ↓
Realized Slippage
```

Day 3 should measure:

- average slippage,
- median slippage,
- worst slippage,
- slippage by symbol,
- slippage by strategy,
- slippage during high-volatility periods.

---

## 23. Risk-Control Layering

```text
                    SIGNAL
                       │
                       ▼
              Strategy Conditions
                       │
                       ▼
                SL / TP Logic
                       │
                       ▼
                 Risk Sizing
                       │
                       ▼
              Strategy-Level Gate
                       │
             ┌─────────┼─────────┐
             │         │         │
          Spread      News     Daily Loss
             │         │         │
             └─────────┼─────────┘
                       │
                       ▼
                Portfolio Risk
                       │
                       ▼
                 Margin Check
                       │
                       ▼
              Common Execution
                       │
                       ▼
             Hard Risk Boundary
                       │
                       ▼
                  OrderSend
                       │
                       ▼
                     MT5
```

This layered design is particularly visible in High Trade and Micro Scalp.

GE follows a different branch after SL/TP calculation.

---

## 24. Important Architecture Separation

The system should be analyzed as:

```text
Strategy Logic
       ≠
Risk Sizing
       ≠
Execution
       ≠
Portfolio Control
       ≠
Telemetry
```

Conceptually:

```text
Strategy asks:
"Should I trade?"

Risk engine asks:
"How large should the trade be?"

Risk gate asks:
"Is this trade allowed?"

Execution layer asks:
"How should the order be sent?"

Telemetry asks:
"What actually happened?"
```

This separation is essential for quantitative validation.

---

## 25. Finding Summary

| ID | Finding | Status |
|---|---|---|
| `RISK-SIZING-001` | Multiple risk-sizing architectures exist | Verified |
| `RISK-CALC-002` | Portfolio mathematical SL/TP uses fixed-lot assumptions while final volume can follow legacy volume path | Verified architecture difference |
| `RISK-CALC-003` | Multiple price-value calculation mechanisms exist | Verified |
| `PORTFOLIO-RISK-003` | Reviewed aggregate risk calculation is symbol-scoped | Verified |
| `GE-RISK-001` | GE pending execution is outside common market hard-risk boundary | Verified |
| `TELEMETRY-001` | Same-symbol/same-magic post-fill tracking requires overlap validation | Validation item |
| `GE-DUP-001` | GE has market-state anti-duplicate protection | Verified |
| `GE-EXEC-001` | GE has independent pending-order filling fallback | Verified |
| `EXEC-SLIP-001` | `deviation=50` does not provide realized slippage measurement | Verified interpretation |

---

## 26. Findings: Observation → Evidence → Hypothesis → Validation

### RISK-SIZING-001

**Observation:** Multiple position-sizing architectures are active.

**Evidence:** High Trade/Micro use dynamic risk-volume calculations. Portfolio can use legacy normalized minimum volume. GE uses `GE_FixedLotSize`.

**Hypothesis:** Actual risk consistency may vary between strategy families.

**Proposed Change:** Do not change code yet. First calculate realized risk percentage per trade.

**Validation:** Compare expected risk % vs actual risk % by strategy family.

**Expected Impact:** Establish whether risk is actually standardized across the engine.

---

### RISK-CALC-002

**Observation:** Portfolio mathematical SL/TP uses fixed-lot assumptions.

**Evidence:** SL distance is derived from balance, risk percentage, fixed lot and `VALUE_PER_LOT`. Final order volume can come from legacy volume normalization.

**Hypothesis:** Nominal risk and actual risk may differ.

**Validation:** Reconstruct risk from actual executed entry, SL and volume.

**Expected Impact:** Determine actual risk exposure rather than relying on nominal configuration.

---

### RISK-CALC-003

**Observation:** Multiple price-value calculation methods exist.

**Evidence:** Some paths use broker tick value/tick size while others use `VALUE_PER_LOT`.

**Hypothesis:** Symbol/broker conditions could produce differences.

**Validation:** Compare calculated monetary risk against actual MT5 P/L for controlled moves/trades.

**Expected Impact:** Identify whether risk calculations are internally consistent.

---

### PORTFOLIO-RISK-003

**Observation:** Reviewed aggregate risk calculation filters to `_Symbol`.

**Evidence:** Positions for other symbols are skipped.

**Hypothesis:** Cross-symbol portfolio exposure may not be represented by this particular risk function.

**Validation:** Determine intended product requirement and compare current behavior against it.

**Expected Impact:** Clarify account-level versus symbol-level risk policy.

---

### GE-RISK-001

**Observation:** GE pending orders bypass the common market hard-risk boundary.

**Evidence:** `GE_PlacePendingOrder()` directly submits `GE_FixedLotSize + SL + TP` using `TRADE_ACTION_PENDING`.

**Hypothesis:** GE may represent a separate risk regime.

**Validation:** Calculate actual GE trade risk percentages.

**Expected Impact:** Determine whether GE is within the intended global risk envelope.

---

### TELEMETRY-001

**Observation:** Post-fill tracking uses symbol + magic matching.

**Evidence:** The tracking process identifies positions using these attributes.

**Hypothesis:** Multiple simultaneous same-magic positions could complicate attribution.

**Validation:** Search live/historical trades for same-symbol/same-magic overlap.

**Expected Impact:** Confirm telemetry integrity.

---

## 27. GE-Specific Execution Diagram

```text
                 GE_RunDailyLondonBreakout()
                            │
                            ▼
                 Daily / Hour Guards
                            │
                            ▼
              GE_HasLiveBreakoutOrders()
                            │
                  ┌─────────┴─────────┐
                  │                   │
                Exists              None
                  │                   │
                  ▼                   ▼
                Skip            Trading Permission
                                      │
                                      ▼
                                  GE_CalcATR
                                      │
                                      ▼
                                  GE_CalcSR
                                      │
                                      ▼
                            Breakout Price Levels
                                      │
                       ┌──────────────┴──────────────┐
                       │                             │
                       ▼                             ▼
                 BUY STOP                       SELL STOP
                       │                             │
                       ▼                             ▼
               GE_CalculateSLTP                GE_CalculateSLTP
                       │                             │
                       ▼                             ▼
               GE_PlacePendingOrder          GE_PlacePendingOrder
                       │                             │
                       └──────────────┬──────────────┘
                                      ▼
                              Both Successful?
                                │          │
                               YES         NO
                                │          │
                                ▼          ▼
                             Keep       Cancel
                             both       successful leg
                                │          │
                                └────┬─────┘
                                     ▼
                                   MT5
```

---

## 28. Common Market Execution Diagram

```text
              Strategy Signal
                    │
                    ▼
              Strategy SL/TP
                    │
                    ▼
             Strategy Volume
                    │
                    ▼
             Strategy Gate
                    │
                    ▼
          PlaceMarketOrderVolume()
                    │
                    ▼
             SendMarketOrder()
                    │
        ┌───────────┼────────────┐
        │           │            │
        ▼           ▼            ▼
   Daily Stop     News        Session
        │           │            │
        └───────────┼────────────┘
                    ▼
            Normalize Volume
                    │
                    ▼
         IsOrderRiskWithinHardCap
                    │
             ┌──────┴──────┐
             │             │
            PASS          FAIL
             │             │
             ▼             ▼
        Trade Request    Reject
             │
             ▼
   Filling Fallback
             │
             ▼
          OrderSend
             │
             ▼
             MT5
```

---

## 29. Day-2 Architecture Conclusion

The most important conclusion is not:

> "The system has a bug."

The correct conclusion is:

> **BoltIQ contains several strategy-specific risk and execution architectures, and the next step is to quantitatively verify whether those architectures produce consistent and intended real-world risk.**

The code review has now established enough evidence to stop expanding the strategy-by-strategy trace.

Continuing to inspect every strategy without measuring actual outcomes would provide diminishing value.

The next research phase should therefore move from:

```text
CODE TRACE
```

to:

```text
DATA VALIDATION
```

---

## 30. Day 3 Validation Plan

Day 3 should answer questions that code inspection cannot answer.

### A. Actual Risk

For every trade:

```text
Actual Risk %
=
|Entry - SL|
×
Volume
×
Value Per Price Unit
÷
Equity
×
100
```

Compare with configured risk.

### B. Risk Distribution

Measure:

```text
Mean Risk %
Median Risk %
P95 Risk %
Maximum Risk %
```

grouped by:

```text
Strategy
Symbol
Preset
Timeframe
Market Regime
```

### C. Drawdown

Measure:

```text
Maximum Drawdown
Daily Drawdown
Strategy Drawdown
Symbol Drawdown
Consecutive Losses
```

### D. Trade Distribution

Measure:

```text
Trades per day
Trades per strategy
Trades per symbol
Win rate
Average win
Average loss
Profit factor
Expectancy
```

### E. Execution Quality

Measure:

```text
Requested Entry
Actual Fill
Realized Slippage
Spread at Entry
Spread at Exit
Execution Rejections
Filling Fallback Frequency
```

### F. Portfolio Exposure

Measure:

```text
XAUUSD exposure
BTCUSD exposure
EURUSD exposure
GBPUSD exposure
Total managed exposure
```

---

## 31. Day 3 Risk Validation Matrix

| Question | Required Data | Result |
|---|---|---|
| Does actual risk match configured risk? | Entry, SL, volume, equity | Pending |
| Does GE remain within risk cap? | GE trades | Pending |
| Does Portfolio risk match mathematical expectation? | Portfolio trades | Pending |
| Is symbol-value conversion accurate? | Tick value/size + deal P/L | Pending |
| Is portfolio risk truly account-wide? | Multi-symbol positions | Pending |
| How much slippage occurs? | MT5 deal history | Pending |
| Are execution fallbacks common? | Order history/logs | Pending |
| Does same-magic overlap occur? | Position history | Pending |
| Does telemetry identify trades correctly? | Trade/event logs | Pending |

---

## 32. What Should NOT Be Changed Yet

Based on Day 2 code tracing alone, do not immediately change:

- `GE_FixedLotSize`
- `VALUE_PER_LOT`
- `BOLTIQ_MAX_RISK_PER_TRADE_PCT`
- portfolio risk limits
- GE execution architecture
- magic numbers
- strategy entry logic
- ONNX thresholds
- SL/TP multipliers

Reason:

```text
Code observation
        ≠
Measured trading defect
```

First establish the actual behavior.

---

## 33. Recommended Evidence Hierarchy

For Day 3 and later research:

```text
1. Actual MT5 deal/order history
              ↓
2. Account equity / balance history
              ↓
3. Strategy telemetry
              ↓
4. Execution logs
              ↓
5. Backtest results
              ↓
6. Code-derived expectations
```

When actual trade data contradicts a code-level assumption, investigate the actual execution data first.

---

## 34. Day-2 Deliverables

### Strategy execution map

```text
Signal
 ↓
SL/TP
 ↓
Sizing
 ↓
Gate
 ↓
Execution
 ↓
MT5
```

### Risk architecture map

```text
Dynamic Risk
Legacy Volume
GE Fixed Lot
```

### Execution architecture map

```text
Common Market Execution
+
GE Pending Execution
```

### Risk findings

```text
RISK-SIZING-001
RISK-CALC-002
RISK-CALC-003
PORTFOLIO-RISK-003
GE-RISK-001
```

### Telemetry/architecture findings

```text
TELEMETRY-001
GE-DUP-001
GE-EXEC-001
EXEC-SLIP-001
```

### Day-3 validation questions

Defined and ready for quantitative testing.

---

## 35. Final Day-2 Conclusion

### Status: CODE-LEVEL REVIEW COMPLETE

Day 2 established that BoltIQ has a layered trading architecture with multiple independent risk-sizing and execution paths.

The strongest common safety boundary is present in the market-order execution path:

```text
Final Volume
+
Entry
+
SL
        ↓
Hard Risk Cap
```

High Trade and Micro Scalp use this common execution infrastructure after extensive strategy-level gating.

Portfolio/Legacy strategies also eventually reach the common market execution path, although their mathematical SL/TP construction and final volume path use different assumptions that require quantitative validation.

Gold Elite is architecturally different.

Its London Breakout strategy:

```text
Calculates SL/TP
        ↓
Uses GE_FixedLotSize
        ↓
Creates pending orders directly
```

and therefore does not visibly pass through the common market-order hard-risk boundary.

This makes GE the highest-priority item for **risk measurement**, not necessarily the highest-priority item for code modification.

The correct next step is therefore not to rewrite the execution architecture.

The correct next step is:

```text
                    DAY 2
                      │
              CODE-LEVEL MAPPING
                      │
                      ▼
              What does the code do?
                      │
                      ▼
                    DAY 3
                      │
            QUANTITATIVE VALIDATION
                      │
                      ▼
              What actually happens?
                      │
                      ▼
                    DAY 4
                      │
          PERFORMANCE / REGIME ANALYSIS
                      │
                      ▼
                    DAY 5+
                      │
       PRODUCT + MQL5 MARKET VALIDATION
                      │
                      ▼
              FINAL RECOMMENDATIONS
```

---

## 36. Research Principle

The Day-2 analysis should be interpreted using one core principle:

> **Do not convert an architectural difference into a defect until the difference has been measured against the intended product requirement and real trading behavior.**

This keeps the BoltIQ research process evidence-driven.

The next stage is therefore to measure:

```text
Expected
   vs
Actual
```

for:

- risk,
- exposure,
- execution,
- slippage,
- drawdown,
- trade distribution,
- strategy behavior,
- cross-symbol portfolio interaction.

That quantitative evidence will determine which Day-2 observations become confirmed engineering improvements and which are simply intentional architecture.
