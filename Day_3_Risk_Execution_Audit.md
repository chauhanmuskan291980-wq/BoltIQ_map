# BoltIQ Gold Portfolio Engine --- Day 3 Work Record

## Quantitative Validation Preparation, MT5 Strategy Tester Setup, Compilation & Compatibility Investigation

**Project:** BoltIQ Gold Portfolio Engine\
**Primary EA:** `BoltIQ_Gold_Portfolio_Engine.mq5`\
**Environment:** MetaTrader 5 / MetaEditor / Strategy Tester\
**Status:** Strategy Tester preparation is in progress; the quantitative
backtest has not started yet because compilation/dependency
compatibility must be resolved first.

------------------------------------------------------------------------

## 1. Day 3 objective

Day 3 is the transition from the code-level risk/execution audit into
quantitative validation.

The central question is:

> **What actually happens when the BoltIQ risk, sizing, execution, and
> strategy rules are applied to historical market data?**

The intended validation is not optimization. It is measurement of
whether the implementation behaves consistently with the risk/execution
architecture identified during Days 1--2.

### Planned validation areas

1.  Actual trade-risk validation
2.  Gold Elite risk validation
3.  Portfolio risk/exposure validation
4.  Execution and slippage validation
5.  Drawdown and trade-distribution analysis
6.  Strategy-level comparison
7.  Comparison of configured/expected risk against realized historical
    behavior

------------------------------------------------------------------------

## 2. Work completed before Day 3

### Day 1 --- Risk & execution architecture audit

Day 1 focused on the core risk and execution architecture:

-   Daily loss controls
-   Emergency stop / emergency close controls
-   Risk-per-trade limits
-   Volume normalization
-   Mathematical SL/TP calculations
-   Broker tick-value/tick-size handling
-   Hard risk-cap checks
-   Market-order execution
-   `OrderCheck()`
-   `OrderSend()`
-   Filling-mode fallback
-   News/session/weekend filters
-   Capital-preservation gates
-   Position/exposure checks
-   Closed-trade recording
-   Trade telemetry
-   Persistent state
-   Post-fill tracking

Day 1 established that the EA contains multiple layers of risk and
execution controls, but their historical behavior still needs to be
measured.

### Day 2 --- Strategy-level execution and risk mapping

Day 2 traced how strategy groups reach execution.

High-level architecture:

``` text
                         BoltIQ EA
                            |
                         OnTick()
                            |
             +--------------+--------------+
             |              |              |
        Portfolio       High Trade      Micro Scalp
             |              |              |
       Strategy layer   Risk engine     Risk engine
             |              |              |
             +--------------+--------------+
                            |
                   Common market execution
                            |
                    SendMarketOrder()
                            |
                 Hard risk-cap validation
                            |
                  OrderCheck / OrderSend
```

A separate Gold Elite path was identified:

``` text
                  Gold Elite / London
                         |
                 GE_RunDailyLondonBreakout()
                         |
                 GE_CalculateSLTP()
                         |
                 GE_PlacePendingOrder()
                         |
                 Pending BUY STOP /
                 Pending SELL STOP
```

------------------------------------------------------------------------

## 3. Day 2 findings carried into Day 3

### RISK-SIZING-001 --- Multiple sizing architectures

The EA uses different position-sizing architectures.

**High Trade / Micro Scalp:** dynamic risk-percentage engine.

**Legacy Portfolio:** legacy minimum-lot execution path.

**Gold Elite:** separate fixed-lot architecture using `GE_FixedLotSize`.

Day 3 therefore needs to validate each architecture separately.

### RISK-CALC-002 --- Mathematical SL vs actual volume

The legacy portfolio mathematical SL calculation uses fixed-lot
assumptions involving `FixedLotSize` and `VALUE_PER_LOT`, while the
downstream execution path uses `LegacyOrderVolume()`.

The intended monetary risk and actual submitted volume therefore need
empirical validation.

This is a validation item, not yet a confirmed production bug.

### RISK-CALC-003 --- Different value-per-price mechanisms

The EA uses both:

``` text
VALUE_PER_LOT
```

and broker-derived:

``` text
SYMBOL_TRADE_TICK_VALUE
SYMBOL_TRADE_TICK_SIZE
```

Day 3 must determine whether these mechanisms materially differ under
tester conditions.

### PORTFOLIO-RISK-003 --- Portfolio risk scope

The reviewed `IsPortfolioRiskOkVolume()` logic filters positions by the
current symbol.

Therefore the currently reviewed calculation is symbol-scoped.

Day 3 needs to establish whether this matches the intended risk
definition or whether account-wide aggregate exposure is intended.

### GE-RISK-001 --- Gold Elite risk boundary

`GE_PlacePendingOrder()` directly submits pending orders using
`GE_FixedLotSize` and does not call the common
`IsOrderRiskWithinHardCap()` inside the pending-order function.

`GE_CalculateSLTP()` performs GE-specific validation using its fixed-lot
assumptions.

Day 3 must measure the actual historical GE risk before deciding whether
this separate architecture creates a practical risk-limit issue.

### TELEMETRY-001 --- Post-fill tracking

The common market-order path scans positions by symbol + magic and
tracks the first matching position.

Because some branches can allow same-magic overlap, Day 3 should
validate tracking accuracy under actual test conditions.

### GE-DUP-001 --- GE duplicate protection

The GE path checks for existing live breakout orders/positions using
symbol + `MAGIC_LONDON_BREAKOUT` before placing new pending orders.

### GE-EXEC-001 --- Separate GE filling path

GE pending orders use their own filling-mode fallback:

``` text
RETURN → IOC → FOK
```

The common market-order path has its own filling fallback.

### EXEC-SLIP-001 --- `deviation=50`

`request.deviation = 50` is an execution tolerance setting, not realized
slippage measurement.

Actual slippage must be calculated from requested/expected price versus
actual deal/fill price when the tester data permits.

------------------------------------------------------------------------

## 4. Day 3 quantitative formulas

### Actual risk

``` text
Risk Money
=
|Entry Price - Stop Loss Price|
×
Volume
×
Value Per Price Unit
```

### Actual risk percentage

``` text
Actual Risk %
=
Risk Money
/
Equity
×
100
```

Compare actual risk with configured/intended risk.

### Slippage

``` text
Realized Slippage
=
Actual Fill Price
-
Requested/Expected Price
```

The sign convention must be documented when the final calculation is
implemented.

------------------------------------------------------------------------

## 5. Planned Day 3 result table

  -----------------------------------------------------------------------------
  Strategy        Trades Avg Risk % Max Risk %   Win Rate        Avg     Max DD
  Group                                                     Slippage 
  ----------- ---------- ---------- ---------- ---------- ---------- ----------
  High Trade         TBD        TBD        TBD        TBD        TBD        TBD

  Micro Scalp        TBD        TBD        TBD        TBD        TBD        TBD

  Portfolio          TBD        TBD        TBD        TBD        TBD        TBD

  Gold Elite         TBD        TBD        TBD        TBD        TBD        TBD
  -----------------------------------------------------------------------------

Additional measurements:

-   Average/minimum/maximum risk
-   Average and maximum volume
-   Average win/loss
-   Win rate
-   Profit factor
-   Expectancy
-   Consecutive losses
-   Trades per day
-   Daily drawdown
-   Maximum drawdown
-   Exposure by symbol
-   Exposure by direction
-   Slippage by strategy/symbol/session
-   Rejected orders
-   Observable risk-gate blocks

------------------------------------------------------------------------

## 6. Initial Strategy Tester setup

The MT5 Strategy Tester was opened.

The initial visible configuration was:

``` text
Expert:       GoldGuardian_EA__5.ex5
Symbol:       XAUUSD
Timeframe:    M15
Date:         Custom period
From:         2024.01.01
To:           2026.01.01
Forward:      No
Modelling:    Every tick
Deposit:      10000 USD
Leverage:     1:100
Optimization: Disabled
Visual mode:  Disabled
```

### Important

`GoldGuardian_EA__5.ex5` was the initially selected Expert, but this is
not the target EA for the BoltIQ quantitative validation.

The target must be the correctly compiled:

``` text
BoltIQ_Gold_Portfolio_Engine.ex5
```

The initial tester configuration is therefore setup evidence, not final
BoltIQ backtest evidence.

------------------------------------------------------------------------

## 7. Adding the BoltIQ source/dependencies

The BoltIQ source and dependency files were placed into the MT5 MQL5
Experts structure.

The relevant project structure became visible under:

``` text
MQL5
└── Experts
    └── BoltIQ
        ├── Constants.mqh
        ├── CoreRiskEngine.mqh
        ├── DashboardEngine.mqh
        ├── ExecutionEngine.mqh
        ├── GoldEliteHelpers.mqh
        ├── LegacyStrategies.mqh
        ├── PersistenceEngine.mqh
        ├── PositionManagement.mqh
        ├── ResearchModes.mqh
        ├── SignalEngine.mqh
        ├── TelegramEngine.mqh
        ├── TelemetryEngine.mqh
        ├── TesterLogging.mqh
        └── VolatilityFilter.mqh
```

The main EA is:

``` text
BoltIQ_Gold_Portfolio_Engine.mq5
```

------------------------------------------------------------------------

## 8. Initial compilation blocker

The first main-EA compilation showed:

``` text
file 'Experts\BoltIQ\CoreRiskEngine.mqh' not found
```

Additional errors appeared in `GoldEliteHelpers.mqh`, including missing
identifiers such as:

``` text
SavePersistentData
RecordTradeBlock
RecordEADecision
CancelAllPendingOrders
CloseAllPositions
```

These were treated cautiously because a missing core dependency can
create cascading compilation errors.

------------------------------------------------------------------------

## 9. CoreRiskEngine dependency restored

The original `CoreRiskEngine.mqh` was located and added to the correct
BoltIQ folder.

After recompilation, the missing-file error disappeared and the large
cascade of `GoldEliteHelpers.mqh` errors was substantially reduced.

This confirmed that the dependency path was now being resolved
correctly.

------------------------------------------------------------------------

## 10. Current compilation state

The latest compilation shows:

``` text
6 errors
5 warnings
```

The remaining errors are concentrated in:

-   `TesterLogging.mqh`
-   `TelegramEngine.mqh`

The earlier large `GoldEliteHelpers.mqh` cascade is no longer present in
the latest result.

------------------------------------------------------------------------

## 11. Current error --- TesterQuietLogging

Compiler error:

``` text
undeclared identifier 'TesterQuietLogging'
```

Location:

``` text
TesterLogging.mqh
Line 43
Column 12
```

Relevant function:

``` cpp
bool TesterQuietLoggingActive()
{
   return (TesterQuietLogging && (bool)MQLInfoInteger(MQL_TESTER));
}
```

The main EA already contains:

``` cpp
input bool TesterQuietLogging = false;
```

Therefore the variable exists in the main EA source.

The current issue is being treated as a declaration/include visibility
problem rather than as a missing trading feature.

**No duplicate variable should be created until the include/declaration
structure is confirmed.**

------------------------------------------------------------------------

## 12. Current Telegram errors

`TelegramEngine.mqh` reports undeclared identifiers:

``` text
TG_FreeTeaserEnabled
TelegramChatIDFree
TG_FreeCtaHandle
```

Relevant code inspected:

``` cpp
bool TG_FreeChannelActive()
{
   return (TelegramEnabled &&
           TG_FreeTeaserEnabled &&
           TelegramChatIDFree != "");
}
```

The next investigation is to search the complete source tree for the
declarations of these identifiers before adding anything.

------------------------------------------------------------------------

## 13. Current warnings

The latest compilation reports two warnings:

``` text
return value of 'OrderSend' should be checked
```

in:

``` text
GoldEliteHelpers.mqh
Line 299
Line 324
```

These warnings do not currently block compilation.

They should be reviewed later because they concern execution behavior,
but they should not be changed casually during tester compatibility
work.

------------------------------------------------------------------------

## 14. Founder authorization

The project founder explicitly authorized editing and fixing the EA and
its dependencies for MT5 Strategy Tester compatibility.

The authorization specifically covers:

-   Compilation errors
-   Dependency conflicts
-   `TesterLogging.mqh`
-   `TesterQuietLogging`
-   `GoldEliteHelpers.mqh`
-   Tester-specific compatibility problems

The explicit constraint is:

> Keep the core production trading logic intact.

Do not unintentionally change:

-   Entries
-   Exits
-   Risk management
-   Position sizing
-   Filters
-   Execution behavior

Every modification must document:

1.  What changed
2.  Why it changed
3.  Whether production trading logic was affected

------------------------------------------------------------------------

## 15. Modification policy

All fixes follow:

``` text
Tester compatibility fix
        ↓
No strategy redesign
        ↓
No entry changes
        ↓
No exit changes
        ↓
No risk-model changes
        ↓
No position-sizing changes
        ↓
No filter changes
        ↓
No execution-policy changes
```

If a fix could alter trading behavior, it must be isolated and reviewed
explicitly.

------------------------------------------------------------------------

## 16. Modification log

### COMP-001 --- CoreRiskEngine dependency

**Issue:** `Experts\BoltIQ\CoreRiskEngine.mqh` was not found.

**Action:** Restored the original dependency to the expected BoltIQ
folder.

**Reason:** Resolve the intended project dependency structure for
compilation.

**Trading logic changed:** No.

**Status:** Resolved.

### COMP-002 --- TesterLogging / TesterQuietLogging

**Issue:** `TesterLogging.mqh` reports `TesterQuietLogging` as
undeclared.

**Evidence:** Main EA already declares
`input bool TesterQuietLogging = false;`.

**Action:** No final code modification recorded yet.

**Next step:** Verify declaration/include visibility.

**Trading logic changed:** No.

**Status:** Investigating.

### COMP-003 --- Telegram configuration/state identifiers

**Issue:** `TG_FreeTeaserEnabled`, `TelegramChatIDFree`, and
`TG_FreeCtaHandle` are not currently visible to `TelegramEngine.mqh`.

**Action:** No final code modification recorded yet.

**Next step:** Search the complete source tree for existing
declarations.

**Trading logic changed:** No.

**Status:** Investigating.

------------------------------------------------------------------------

## 17. What has NOT been completed yet

The following are not yet valid Day 3 quantitative results:

-   No final BoltIQ baseline backtest
-   No actual risk-per-trade distribution
-   No maximum realized risk
-   No GE realized risk
-   No portfolio aggregate exposure measurement
-   No realized slippage measurement
-   No strategy-level win-rate calculation
-   No final drawdown measurement
-   No profit factor
-   No expectancy
-   No final strategy comparison

The current work is **tester preparation and compilation
compatibility**, not quantitative evidence.

------------------------------------------------------------------------

## 18. Exact next steps

### Step 1 --- Resolve current declarations

Search the complete project for:

``` text
TesterQuietLogging
TG_FreeTeaserEnabled
TelegramChatIDFree
TG_FreeCtaHandle
```

Determine whether each is:

-   already declared elsewhere
-   declared after the relevant include
-   declared under another name
-   genuinely missing

### Step 2 --- Apply minimal compatibility fixes

Only after confirming the source structure:

-   Fix declaration/include visibility
-   Restore genuinely missing configuration declarations if necessary
-   Do not redesign strategy logic
-   Do not change risk calculations

### Step 3 --- Recompile the main EA

Target:

``` text
0 errors
```

Then review warnings separately.

### Step 4 --- Confirm correct EX5

Strategy Tester must use:

``` text
BoltIQ_Gold_Portfolio_Engine.ex5
```

not the initially selected unrelated EA.

### Step 5 --- Run controlled baseline test

Initial baseline should use:

``` text
Symbol: XAUUSD
Timeframe: M15
Model: Every Tick
Optimization: Disabled
Visual mode: Disabled
Forward: No
```

Document the exact historical dates and tester settings.

------------------------------------------------------------------------

## 19. Day 3 evidence to capture

For the final work record, capture screenshots of:

### A. Strategy Tester configuration

-   Expert name
-   Symbol
-   Timeframe
-   Date range
-   Modelling mode
-   Deposit
-   Leverage
-   Optimization setting

### B. Successful compilation

Capture:

``` text
0 errors
```

and remaining warnings, if any.

### C. Tester result summary

Capture:

-   Total trades
-   Net result
-   Drawdown
-   Balance/equity curve
-   Other available statistics

### D. Trade/deal data

Capture or export data needed for:

-   Entry
-   Exit
-   Volume
-   SL
-   TP
-   Strategy/comment/magic
-   Requested/actual price where available
-   Profit
-   Commission
-   Swap
-   Time

### E. Graph

Capture the Strategy Tester graph.

### F. Journal

Capture relevant tester events:

-   Order attempts
-   Rejections
-   Filling issues
-   Risk blocks
-   Execution messages
-   Errors

------------------------------------------------------------------------

## 20. Evidence classification

Every Day 3 conclusion should be classified as:

### Verified

Directly demonstrated by tester data or source code.

### Observed

Visible in logs/results but requiring interpretation.

### Inferred

Reasonable conclusion derived from observations.

### Hypothesis

Possible explanation requiring further validation.

### Not tested

No sufficient evidence yet.

This prevents a code observation from being presented as a proven
live-trading behavior.

------------------------------------------------------------------------

## 21. Day 3 success criteria

``` text
[ ] Correct BoltIQ EA compiled
[ ] Dependencies resolved
[ ] Tester compatibility fixes documented
[ ] Core production trading logic preserved
[ ] Baseline test completed
[ ] Actual risk measured
[ ] GE risk measured
[ ] Portfolio exposure measured
[ ] Execution/slippage measured where data permits
[ ] Drawdown measured
[ ] Trade distribution measured
[ ] Strategy-level results separated
[ ] Findings classified
[ ] Screenshots attached
[ ] Final Day 3 report completed
```

------------------------------------------------------------------------

## 22. Important interpretation rule

A profitable or unprofitable backtest by itself does not prove whether
the risk architecture is correct.

The primary Day 3 validation chain is:

``` text
Configured risk
        ↓
Calculated risk
        ↓
Submitted volume
        ↓
Historical execution
        ↓
Realized risk
        ↓
Portfolio exposure
        ↓
Drawdown
```

Performance statistics are secondary to validating whether the
implementation behaves as designed.

------------------------------------------------------------------------

## 23. Current Day 3 status

  Area                             Status
  -------------------------------- -------------
  Day 1 findings carried forward   Complete
  Day 2 findings carried forward   Complete
  MT5 Strategy Tester opened       Complete
  EA/dependency setup              Complete
  CoreRiskEngine restored          Complete
  Main EA compilation              In progress
  Tester compatibility             In progress
  Baseline backtest                Pending
  Risk validation                  Pending
  GE validation                    Pending
  Portfolio exposure validation    Pending
  Slippage validation              Pending
  Drawdown analysis                Pending
  Final Day 3 report               Pending

------------------------------------------------------------------------

## 24. Current conclusion

Day 3 has correctly reached the tester-preparation and
compilation-validation stage.

The initial missing `CoreRiskEngine.mqh` dependency was resolved,
reducing the compilation problem to a small set of
declaration/configuration issues.

The current blockers are primarily:

``` text
TesterQuietLogging
TG_FreeTeaserEnabled
TelegramChatIDFree
TG_FreeCtaHandle
```

The next technical action is to identify the intended declarations for
these identifiers and make the smallest possible tester-compatibility
corrections.

**No quantitative backtest result should be recorded as a Day 3 finding
until the correct BoltIQ EA compiles and the Strategy Tester run has
been completed.**

The final Day 3 record should combine this setup/compatibility evidence
with the actual quantitative results once the baseline test is
available.
