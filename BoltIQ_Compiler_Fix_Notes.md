# BoltIQ Compiler Issue Fixes

## Purpose
Document the small code changes made to get `BoltIQ_Gold_Portfolio_Engine.mq5` compiling with **0 errors**.

## 1. Fixed `TesterLogging.mqh`

### Issue
`TesterLogging.mqh` had an invalid Unicode dash in `TesterQuietLoggingActive()`:

```cpp
return (–TesterQuietLogging && (bool)MQLInfoInteger(MQL_TESTER));
```

The `–` character was an en dash, not the MQL5 logical NOT operator.

### Change
Replaced it with the correct `!` operator:

```cpp
bool TesterQuietLoggingActive()
{
   return (!TesterQuietLogging && (bool)MQLInfoInteger(MQL_TESTER));
}
```

The main EA already had:

```cpp
input bool TesterQuietLogging = false;
```

before including `TesterLogging.mqh`, so no duplicate variable was added.

---

## 2. Restored Missing Telegram Configuration Variables

### Issue
`TelegramEngine.mqh` referenced these identifiers, but they were not declared anywhere in the project:

- `TG_FreeTeaserEnabled`
- `TelegramChatIDFree`
- `TG_FreeCtaHandle`

This caused 5 compiler errors.

### Change
Added the missing configuration variables near the existing Telegram settings in `BoltIQ_Gold_Portfolio_Engine.mq5`:

```cpp
input bool   TG_FreeTeaserEnabled = false;
input string TelegramChatIDFree   = "";
input string TG_FreeCtaHandle     = "";
```

These defaults keep the free-channel/teaser functionality disabled unless explicitly configured.

---

## 3. Compilation Result

After the above changes:

```text
0 errors, 2 warnings
```

The remaining two warnings are in `GoldEliteHelpers.mqh`:

```text
return value of 'OrderSend' should be checked
```

at lines 299 and 324.

These warnings were **not changed** because they do not prevent compilation and are part of the separate execution-path review.

## Modification Scope

- No trading strategy logic changed.
- No risk calculation changed.
- No order-entry logic changed.
- No Telegram production destination was invented or changed.
- Changes were limited to compiler/dependency compatibility.
