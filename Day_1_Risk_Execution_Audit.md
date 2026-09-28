# BoltIQ Day 1 — Risk & Execution Architecture Review

**Date:** 2026-09-28  
**Scope:** Risk controls, position sizing, SL/TP, exposure, trading conditions, order creation, execution, position management, and telemetry.  
**Status:** Day 1 first-pass review complete.  
**Method:** Observation → Evidence → Hypothesis → Validation → Expected Impact

> **Important:** This document records code-level observations from the reviewed BoltIQ EA/modules. It distinguishes verified behavior from items that still require caller tracing or empirical validation. No production code changes are proposed from this review alone.

---

## 1. Executive Summary

The BoltIQ trading engine has a layered risk/execution architecture rather than one single risk check.

High-level flow:

```text
Strategy / Signal
        ↓
Risk / trading-condition checks
        ↓
Strategy-specific volume + SL + TP
        ↓
SendMarketOrder()
        ↓
NormalizeTradeVolume()
        ↓
IsOrderRiskWithinHardCap()
        ↓
MqlTradeRequest
        ↓
OrderCheck / OrderSend via filling fallback
        ↓
Position opened
        ↓
Position management
        ↓
SL / TP / break-even / ATR trailing / emergency/manual exit
        ↓
Closed-trade scan
        ↓
Trade result + persistence + telemetry
```

The review found multiple protective layers: daily-loss controls, hard risk caps, news/session/weekend guards, manual-trade protection, broker volume normalization, order validation, position-management protections, and trade telemetry.

The main area requiring further validation is **position-sizing consistency**.

The code contains multiple sizing/risk concepts:

1. Legacy minimum-volume sizing.
2. Fixed-lot mathematical SL/TP calculation.
3. Dynamic percentage-risk functions for High Trade and Micro Scalp modes.
4. A final per-order hard-risk cap.

The existence of these paths is verified. It is **not yet proven** that the live strategies are incorrectly wired. The next step is to trace each strategy's actual caller path and identify exactly where volume, SL, and TP are generated.

---

# 2. Scope Reviewed

The Day 1 review focused on:

- Main EA initialization/runtime orchestration.
- Core risk checks.
- Position sizing.
- SL/TP calculation.
- Per-order risk verification.
- Portfolio/direction exposure controls.
- News/session/weekend conditions.
- Order creation and broker execution.
- Position management.
- Emergency/manual controls.
- Closed-trade recording and telemetry.

Reviewed code areas included:

- `BoltIQ_Gold_Portfolio_Engine.mq5`
- `BoltIQ/CoreRiskEngine.mqh`
- `BoltIQ/ExecutionEngine.mqh`
- `BoltIQ/PositionManagement.mqh`
- Relevant sizing/SL/TP functions shown during the review.

---

# 3. Runtime Execution Architecture

## 3.1 `OnInit()`

The main EA initialization performs state and infrastructure setup before normal trading.

Observed responsibilities include:

- Account/equity initialization.
- Daily/runtime state reset.
- Strategy statistics initialization.
- Persistent-data loading.
- Trade-memory reconstruction.
- News-calendar refresh.
- ONNX initialization.
- Walk-forward state initialization.
- Dashboard initialization/update.
- Live-state export.
- Startup telemetry.

### Finding

**Observation:** The EA has a substantial initialization phase and does not immediately jump into signal generation.

**Assessment:** This is a useful state-management architecture for a stateful trading system.

**Validation still required:** Restart/recovery behavior after terminal/VPS restart, connection interruption, or partial persistence.

---

# 4. `OnTick()` Runtime Safety Order

The main runtime loop applies multiple gates before strategy execution.

Observed sequence:

```text
Account metrics ready?
        ↓
Manual trade guard
        ↓
New-day handling
        ↓
Daily loss hard lock
        ↓
Profit target
        ↓
Target lock
        ↓
Daily loss limit
        ↓
Weekend guard
        ↓
Emergency-close request
        ↓
Emergency stop
        ↓
News block
        ↓
Telemetry smoke-test mode
        ↓
Strategy engines
        ↓
Pending-order monitoring
        ↓
Open-position management
        ↓
Dashboard / telemetry
```

### Finding

**Observation:** Safety controls are placed before normal strategy execution in the main runtime path.

**Assessment:** This creates a clear risk-gating layer around the strategy engines.

**Important limitation:** A gate in `OnTick()` does not by itself prove every possible order path is protected. Direct order-producing functions still need to be traced.

---

# 5. Position Sizing — Multiple Sizing Paths

## 5.1 Legacy minimum-volume path

The reviewed code contains:

```cpp
double LegacyOrderVolume()
{
   return NormalizeTradeVolume(SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN));
}
```

### Exact behavior

The function obtains:

```cpp
SYMBOL_VOLUME_MIN
```

and passes it through:

```cpp
NormalizeTradeVolume()
```

Therefore, the legacy fallback is based on the broker's **minimum allowed volume**, rather than calculating volume from account equity and a target percentage risk.

Related telemetry explicitly reports:

```cpp
return "Legacy min-volume fallback";
```

### Finding

**Observation:** A legacy sizing path exists that uses broker minimum volume.

**Evidence:** `LegacyOrderVolume()` directly requests `SYMBOL_VOLUME_MIN`.

**Implication:** Minimum-lot sizing is not equivalent to percentage-of-equity risk sizing. Resulting monetary risk depends on SL distance and broker contract specifications.

**Status:** Verified code behavior.

**Not yet proven:** Whether any currently enabled production strategy actually uses this path.

---

# 6. Dynamic Risk Paths

The code contains:

```cpp
double EffectiveHighTradeRiskPct()
{
   double risk = MathMin(
      BOLTIQ_MAX_RISK_PER_TRADE_PCT,
      MathMax(0.01, HighTradeRiskPct)
   );

   if(ChallengeSprintSafeMode)
      risk = MathMin(risk, ChallengeSprintSafeRiskCapPct());

   return MathMax(0.01, risk);
}
```

and:

```cpp
double EffectiveMicroScalpRiskPct()
{
   double risk = MathMin(
      BOLTIQ_MAX_RISK_PER_TRADE_PCT,
      MathMax(0.01, MicroScalpRiskPct)
   );

   if(ChallengeSprintSafeMode)
      risk = MathMin(risk, ChallengeSprintSafeRiskCapPct());

   return MathMax(0.01, risk);
}
```

### What this proves

High Trade and Micro Scalp modes have percentage-risk controls.

They are constrained by:

```text
BOLTIQ_MAX_RISK_PER_TRADE_PCT
```

and, when enabled:

```text
ChallengeSprintSafeRiskCapPct()
```

### Important limitation

The existence of `EffectiveHighTradeRiskPct()` does **not** prove that final order risk equals that percentage.

Actual risk depends on:

```text
volume + entry price + SL distance + symbol contract specifications
```

### Finding

**Observation:** Dynamic risk configuration exists.

**Validation required:** Trace every caller that converts the percentage into actual order volume.

---

# 7. Mathematical SL/TP Calculation

The reviewed function contains:

```cpp
bool CalculateMathematicalSLTP(
   double entryPrice,
   bool isBuy,
   double &slPrice,
   double &tpPrice
)
```

It starts with:

```cpp
double balance = AccountInfoDouble(ACCOUNT_BALANCE);
```

Then:

```cpp
double SL_money =
   balance * (StopLossPercent / 100.0);

double TP_money =
   balance * (TakeProfitPercent / 100.0);
```

Distance is calculated using:

```cpp
calculatedSLDistance =
   SL_money / (FixedLotSize * VALUE_PER_LOT);

calculatedTPDistance =
   TP_money / (FixedLotSize * VALUE_PER_LOT);
```

### Finding

**Observation:** This is a fixed-lot mathematical SL/TP model.

It derives a monetary target from account balance and percentage settings, then converts that amount into price distance using:

```text
FixedLotSize × VALUE_PER_LOT
```

### Validation point

The EA also uses broker-provided:

```cpp
SYMBOL_TRADE_TICK_VALUE
SYMBOL_TRADE_TICK_SIZE
```

Therefore, the hard-coded `VALUE_PER_LOT` assumption needs validation against actual symbol contract specifications.

---

# 8. Hard-Coded `VALUE_PER_LOT` Assumption

The reviewed code contains:

```cpp
#define VALUE_PER_LOT 100.0
// Standard pour XAUUSD (1 lot = 100$ par point)
```

### Finding

**Observation:** The mathematical SL/TP calculation contains an XAUUSD-specific monetary assumption.

The repository describes multi-symbol support including:

- XAUUSD
- BTCUSD
- EURUSD
- GBPUSD

### Potential issue

If this mathematical calculation is used for a symbol whose contract economics differ from the hard-coded assumption, calculated SL/TP distance may not represent intended monetary risk.

### Status

This is a **confirmed architectural assumption**, but not yet a confirmed live trading bug.

It becomes a confirmed defect if:

1. `CalculateMathematicalSLTP()` is used for a non-XAUUSD symbol, and
2. `VALUE_PER_LOT = 100.0` does not match that symbol's actual contract/tick economics.

### Recommended validation

Prefer symbol-aware calculations based on broker-provided:

```cpp
SYMBOL_TRADE_TICK_VALUE
SYMBOL_TRADE_TICK_SIZE
```

when deriving monetary risk.

---

# 9. `CalculateActualRiskMoney()` — Multiple Implementations

Two versions were encountered during the review.

### Version A

```cpp
double CalculateActualRiskMoney(double entryPrice, double stopLoss)
{
   double slDistance = MathAbs(entryPrice - stopLoss);
   double tickValue  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize   = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);

   if(tickSize == 0 || tickValue == 0)
      return 0.0;

   double numTicks = slDistance / tickSize;

   return numTicks * tickValue * LegacyOrderVolume();
}
```

### Version B

```cpp
double CalculateActualRiskMoney(double entryPrice, double stopLoss)
{
   double slDistance = MathAbs(entryPrice - stopLoss);
   double tickValue  = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double tickSize   = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);

   if(tickSize == 0 || tickValue == 0)
      return 0.0;

   double numTicks  = slDistance / tickSize;
   double riskMoney = numTicks * tickValue * FixedLotSize;

   return riskMoney;
}
```

### Critical clarification

If both definitions exist in the **same MQL5 compilation unit**, this is a function redefinition/compile error because the signatures are identical.

If they belong to separate files/versions and only one is included, it is not necessarily a compile error.

### Finding

**Observation:** The reviewed context contains two materially different implementations of the same function.

**Difference:**

```text
Version A → LegacyOrderVolume()
Version B → FixedLotSize
```

**Status:** Requires include/dependency verification before being classified as a confirmed compile defect.

---

# 10. `NormalizeTradeVolume()` Review

The reviewed function obtains broker constraints:

```cpp
double minLot =
   SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);

double maxLot =
   SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);

double step =
   SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
```

It then:

- Applies tester fallbacks.
- Rejects invalid broker volume metadata in live mode.
- Rejects volume below minimum.
- Caps volume at maximum.
- Floors volume to the broker step.
- Rechecks minimum volume.
- Normalizes decimal precision.

### Finding

**Assessment:** The volume-normalization mechanism is structurally reasonable.

It is preferable to passing arbitrary floating-point volume directly to the broker.

### Validation

Test all supported symbols and account/broker configurations.

---

# 11. `SendMarketOrder()` — Final Execution Boundary

The reviewed function receives:

```cpp
SendMarketOrder(
   ENUM_ORDER_TYPE orderType,
   double sl,
   double tp,
   int magic,
   string comment,
   double volume
)
```

It then applies:

```text
Daily entry stop
Weekend entry block
Winning-hour check
News block
Trading-session check
Volume normalization
Hard risk-cap check
Trade request construction
OrderCheck / OrderSend fallback
```

Important section:

```cpp
volume = NormalizeTradeVolume(volume);

if(volume <= 0.0)
{
   LogError("[" + comment + "] Order blocked: invalid/too-small lot size");
   RecordTradeError(comment + ":invalid_lot", 0, 0);
   return 0;
}
```

Then:

```cpp
double entryPrice =
   (orderType == ORDER_TYPE_BUY)
   ? SymbolInfoDouble(_Symbol, SYMBOL_ASK)
   : SymbolInfoDouble(_Symbol, SYMBOL_BID);

if(!IsOrderRiskWithinHardCap(
      entryPrice,
      sl,
      volume,
      comment))
   return 0;
```

### Finding

**Observation:** There is a final per-order risk boundary immediately before the trade request.

**Assessment:** This is an important safety control.

The strategy cannot simply submit arbitrary volume without passing the hard-cap check.

---

# 12. Hard Risk Cap vs Intended Strategy Risk

There are three separate concepts:

```text
1. Intended strategy risk
   ↓
EffectiveHighTradeRiskPct()
or
EffectiveMicroScalpRiskPct()

2. Actual candidate trade risk
   ↓
volume + entry + SL + symbol contract

3. Maximum permitted risk
   ↓
IsOrderRiskWithinHardCap()
```

Example:

```text
Intended risk:       0.50%
Hard maximum:        1.00%
Actual calculated:   0.80%
```

The hard-cap check could pass while the trade still does not match the intended 0.50% target.

### Finding

**Observation:** Hard-cap enforcement and target-risk sizing are separate concepts.

**Validation required:** Compare intended risk against actual calculated SL risk for each strategy.

---

# 13. Exposure Controls

The reviewed risk layer includes:

- Portfolio risk checks.
- Direction limits.
- Existing managed-position risk.
- Proposed-trade risk.
- Maximum long/short counts.

### Important limitation

Some exposure functions filter by:

```cpp
POSITION_SYMBOL == _Symbol
```

Therefore, a function operating on the current chart symbol does not by itself establish account-wide cross-symbol exposure control.

### Finding

**Observation:** Per-symbol exposure protection exists.

**Validation required:** Determine whether a separate account-wide layer aggregates XAUUSD/BTCUSD/EURUSD/GBPUSD simultaneously.

This matters because the repository describes multi-symbol operation.

---

# 14. News Protection

Reviewed configuration includes:

```cpp
input bool UseNewsFilter = true;
input string NewsCurrencies = "USD";
input bool NewsBlockHighImpactOnly = true;
input int NewsBlockMinutesBefore = 15;
input int NewsBlockMinutesAfter = 10;
input bool AutoCloseBeforeNews = true;
input int NewsAutoCloseMinutesBefore = 5;
input bool NewsFailOpen = false;
```

### Finding

The design is conservative when the news calendar fails because:

```cpp
NewsFailOpen = false;
```

The runtime also records news blocks and can cancel pending orders / close positions depending on configuration.

### Validation required

Measure:

- news-blocked entries,
- positions auto-closed,
- calendar failures,
- live vs tester behavior.

---

# 15. Weekend Protection

Reviewed configuration includes:

```cpp
WeekendGuardEnabled = true;
WeekendGuardEntryCutoffHour = 20;
WeekendGuardFlatCutoffHour = 20;
WeekendGuardCancelPendingOrders = true;
WeekendGuardCloseManagedPositions = true;
```

### Finding

There is an explicit governance layer intended to prevent late-Friday/weekend EA exposure.

Validate using broker server time and actual order/position state.

---

# 16. Emergency and Manual Trade Controls

The reviewed system contains:

- Emergency stop.
- Emergency close.
- Manual/unknown-magic trade detection.
- Manual pending-order cancellation.
- Manual position closing.
- Account-wide manual-trade guard configuration.
- Daily-loss hard-lock handling.

### Finding

The system treats manual/unknown trading as a separate risk category instead of silently mixing unmanaged positions with EA-managed positions.

### Validation required

Test:

1. Manual position appears.
2. Manual pending order appears.
3. EA detects it.
4. Configured action occurs.
5. Telemetry records it.
6. Managed positions remain handled correctly.

---

# 17. Position Management

The reviewed position-management layer contains:

- Broker stop-distance adjustment.
- Break-even.
- ATR trailing.
- High Trade profit runner.
- SL improvement checks.
- Emergency close.
- Pending-order cancellation.
- Daily reset.

### Break-even

The system can move SL toward entry after a configured profit threshold and checks that the new SL improves the existing SL.

### ATR trailing

The system:

1. Calculates ATR.
2. Waits for the configured profit/ATR condition.
3. Calculates a trailing SL.
4. Broker-adjusts it.
5. Applies it only if it improves the existing SL.

### High Trade profit runner

The reviewed runner includes checks for:

- positive profit,
- trigger relative to initial risk,
- trend direction,
- ATR fast/slow relationship,
- SL protection.

### Finding

**Assessment:** Position management has multiple protective conditions rather than blindly modifying SL on every tick.

### Validation required

Measure:

- MFE/MAE,
- exit reason distribution,
- average R before/after management,
- SL modification count,
- break-even exits,
- trailing exits,
- profit-runner contribution.

---

# 18. Order Execution

The reviewed execution layer uses:

```cpp
request.deviation = 50;
```

and filling-mode fallback.

### Important clarification

`deviation = 50` is an order-request parameter. It should **not** be treated as measured real-world slippage.

Actual slippage requires comparing:

```text
requested price
vs
actual execution/deal price
```

### Finding

**Observation:** Explicit broker filling handling exists.

**Validation required:** Calculate actual slippage distribution from trade/deal history.

---

# 19. Closed Trade Recording

The reviewed closed-trade scan calculates trade result using:

```text
DEAL_PROFIT
+ DEAL_SWAP
+ DEAL_COMMISSION
```

and then updates trade statistics, persistence, and telemetry.

### Finding

This is useful because gross profit alone does not represent the complete realized result after swap and commission.

### Validation required

Confirm correct aggregation for partial closes and all relevant deal types.

---

# 20. Confirmed vs Unconfirmed Findings

## Confirmed code observations

- Multiple safety gates exist.
- `LegacyOrderVolume()` uses broker minimum volume.
- High Trade and Micro Scalp have effective percentage-risk functions.
- `SendMarketOrder()` normalizes volume.
- `SendMarketOrder()` performs a final hard-risk check.
- News/session/weekend checks exist.
- Position-management protections exist.
- Closed-trade accounting includes profit, swap, and commission.
- Mathematical SL/TP uses fixed-lot and hard-coded `VALUE_PER_LOT` assumptions.
- Broker tick value/tick size are also used for actual-risk calculation.
- The reviewed context contains two different `CalculateActualRiskMoney()` implementations.

## Not yet confirmed

- Which live strategy uses which sizing function.
- Whether the legacy minimum-volume path is active in production.
- Whether `CalculateMathematicalSLTP()` is used by active production strategies.
- Whether all dynamic risk percentages are converted correctly into final volume.
- Whether cross-symbol exposure is fully account-wide.
- Actual execution slippage distribution.
- Real-world effectiveness of break-even/trailing/profit-runner logic.
- Whether both `CalculateActualRiskMoney()` definitions coexist in one compilation unit.

---

# 21. Primary Day-1 Finding

## Finding ID: RISK-SIZING-001

### Title

**Multiple position-sizing and risk-calculation paths require strategy-level mapping.**

### Observation

The EA contains:

- legacy minimum-volume sizing,
- fixed-lot mathematical SL/TP,
- dynamic High Trade risk,
- dynamic Micro Scalp risk,
- final hard per-order risk protection.

### Evidence

Legacy:

```cpp
double LegacyOrderVolume()
{
   return NormalizeTradeVolume(
      SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN)
   );
}
```

Dynamic:

```cpp
EffectiveHighTradeRiskPct()
EffectiveMicroScalpRiskPct()
```

Mathematical:

```cpp
calculatedSLDistance =
   SL_money / (FixedLotSize * VALUE_PER_LOT);
```

Final execution gate:

```cpp
if(!IsOrderRiskWithinHardCap(
      entryPrice,
      sl,
      volume,
      comment))
   return 0;
```

### Hypothesis

Different strategy engines may use different risk-sizing methodologies, potentially causing differences between configured target risk and actual trade risk.

### Validation

Trace:

```text
Strategy
  ↓
volume source
  ↓
SL source
  ↓
TP source
  ↓
SendMarketOrder / PlacePendingOrder
  ↓
hard risk check
```

for every active strategy.

### Expected impact

Establish whether configured risk percentages correspond to actual per-trade monetary risk across the production strategy set.

---

# 22. Secondary Finding: Symbol-Aware Risk Calculation

## Finding ID: RISK-CALC-002

### Title

**Fixed XAUUSD monetary assumption requires multi-symbol validation.**

### Observation

The mathematical SL/TP function uses:

```cpp
#define VALUE_PER_LOT 100.0
```

while the system is described as multi-symbol.

### Hypothesis

If this mathematical calculation is used for symbols whose contract economics differ from the hard-coded assumption, calculated SL/TP distance may not represent intended monetary risk.

### Validation

For:

```text
XAUUSD
BTCUSD
EURUSD
GBPUSD
```

compare:

```text
hard-coded calculation
vs
broker tick-value/tick-size calculation
```

### Expected impact

Prevent incorrect monetary-risk translation when the mathematical risk function is used outside its intended instrument assumptions.

---

# 23. Secondary Finding: Duplicate Function Definition Risk

## Finding ID: CODE-CONSISTENCY-003

### Title

**Two materially different `CalculateActualRiskMoney()` implementations were encountered.**

### Observation

Version A uses:

```cpp
LegacyOrderVolume()
```

Version B uses:

```cpp
FixedLotSize
```

### Hypothesis

If both definitions are included in the same compilation unit, compilation should fail because the function signature is identical.

If they belong to separate versions/modules and only one is included, this is a source-consistency issue rather than a compile defect.

### Validation

Inspect:

```text
#include dependency tree
preprocessor structure
file locations
duplicate definitions
```

### Expected impact

Ensure there is one authoritative implementation of actual-risk calculation.

---

# 24. Recommended Day-2 Validation

Do not change risk parameters yet.

First build this matrix:

| Strategy | Active? | Volume source | Risk % source | SL source | TP source | Hard cap | Cross-symbol exposure |
|---|---|---|---|---|---|---|---|
| Gold Elite | ? | ? | ? | ? | ? | ? | ? |
| London Breakout | ? | ? | ? | ? | ? | ? | ? |
| Portfolio | ? | ? | ? | ? | ? | ? | ? |
| High Trade | ? | ? | ? | ? | ? | ? | ? |
| Micro Scalp | ? | ? | ? | ? | ? | ? | ? |
| Challenge Sprint | ? | ? | ? | ? | ? | ? | ? |

Then validate:

### A. Sizing

```text
Target risk %
      vs
Calculated SL risk %
      vs
Hard maximum %
```

### B. Exposure

```text
Existing positions
      +
Proposed position
      ↓
Total risk
```

across all symbols.

### C. Execution

Measure:

```text
Requested price
Actual deal price
Slippage
OrderCheck result
OrderSend retcode
```

### D. Position management

Measure:

```text
Initial SL
Break-even modification
ATR trailing modification
Profit-runner modification
Final exit
```

---

# 25. Day-1 Conclusion

The Day-1 review establishes that BoltIQ has a substantial risk and execution-control architecture.

The main conclusion is **not that the system is broken**.

The evidence instead shows:

> **The system has multiple risk/sizing paths that must be mapped to individual strategies before we can determine whether configured risk, calculated risk, and realized risk are aligned.**

The next investigation should move from **architecture** to **strategy-level execution tracing**.

No parameter tuning or production code modification should be made solely from today's findings.

---

## Day-1 Status

- **Risk architecture review:** Complete
- **Execution boundary review:** Complete
- **Position-management review:** Complete
- **Sizing-path identification:** Complete
- **Strategy-to-sizing mapping:** Pending
- **Empirical risk validation:** Pending
- **Cross-symbol exposure validation:** Pending
- **Slippage validation:** Pending
- **Code changes:** None recommended yet

---

## Review Method

Continue using:

```text
Observation
    ↓
Evidence
    ↓
Hypothesis
    ↓
Proposed Change
    ↓
Validation
    ↓
Expected Impact
```

Maintain the distinction between **verified**, **inferred**, and **requires validation** throughout the remaining review.