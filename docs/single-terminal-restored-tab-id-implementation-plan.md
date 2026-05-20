# Single-terminal restored-tab identity (v1) implementation plan

## Summary

This document turns the analysis in [`docs/persisted-window-restore-opencode-analysis.md`](./persisted-window-restore-opencode-analysis.md) into an implementation-ready plan for a **WT-native restored-tab identifier**.

The accepted v1 design is:

- generate a new WT-owned persisted identifier for a restored tab
- store it in persisted `NewTerminalArgs`
- carry it through restore and WT-internal relaunch paths
- expose it to the restored child process as `WT_RESTORED_TAB_ID`

This plan is intentionally limited to **one terminal per tab**. **Split panes are explicitly unsupported in v1** and must not receive a `WT_RESTORED_TAB_ID`.

## Goals

- Provide a WT-native identity that survives persisted window restore for a single terminal tab.
- Make the identity available to the restored child process through `WT_RESTORED_TAB_ID`.
- Keep the design scoped to the existing persisted action-list restore model.
- Preserve the identifier through WT-internal round-trips that use `NewTerminalArgs` and `ToCommandline()`.

## Non-goals

- No support for split panes or multi-terminal tabs.
- No attempt to define a general-purpose durable tab identity for all WT tabs.
- No changes to user-facing restore settings or restore policy.
- No new external/session-management contract beyond `WT_RESTORED_TAB_ID` for restored single-terminal tabs.
- No changes to OpenCode in this work item.

## Proposed design

### Identifier shape

- Add a new `NewTerminalArgs` string property: `restoredTabId`.
- Persist it only for restore scenarios.
- Use a GUID-derived string value in plain-text form (no braces), matching the style used for `WT_SESSION`.

### Generation rule

- Generate `restoredTabId` only when serializing a tab with `BuildStartupKind::Persist`.
- Only generate it when the tab serializes as a **single terminal `newTab` action**.
- Do **not** generate it for:
  - split-pane tabs
  - non-terminal tabs such as settings UI
  - non-persist startup paths

For v1, the easiest safe rule is:

1. In `Tab::BuildStartupActions(BuildStartupKind::Persist)`, inspect the pre-existing pane startup state.
2. If the tab is represented by exactly one terminal pane and no split actions are needed, assign a fresh `restoredTabId` onto the `NewTerminalArgs` before wrapping it in `NewTabArgs`.
3. Otherwise leave `restoredTabId` unset.

This keeps split tabs explicitly out of scope without changing the broader restore format.

### Restore behavior

- The persisted action payload stores `restoredTabId` inside `NewTerminalArgs`.
- On restore, the deserialized `NewTerminalArgs` carries `restoredTabId` into terminal settings creation.
- Connection creation passes the value into `ConptyConnection`.
- `ConptyConnection` adds `WT_RESTORED_TAB_ID=<restoredTabId>` to the child environment.
- `WT_RESTORED_TAB_ID` is also added to `WSLENV` so WSL child processes receive it.

### Commandline round-trip behavior

WT already uses `NewTerminalArgs::ToCommandline()` for WT-internal relaunch flows such as elevation/new-window handling. To avoid dropping the identifier in those paths:

- `NewTerminalArgs::ToCommandline()` must emit `--restoredTabId` when present.
- `AppCommandlineArgs` must parse `--restoredTabId` back into `NewTerminalArgs`.

This parser support is implementation plumbing, not a new documented end-user feature.

## Data flow

1. `TerminalPage::PersistState()` asks each tab for `BuildStartupActions(BuildStartupKind::Persist)`.
2. `Tab::BuildStartupActions()` detects the v1-supported case: one terminal pane, no splits.
3. `Tab::BuildStartupActions()` stamps `NewTerminalArgs.restoredTabId` with a fresh WT-generated identifier.
4. `ActionArgs` JSON serialization writes `restoredTabId` into the persisted `newTab` action.
5. WT restart restores the window layout and deserializes `ActionAndArgs`/`NewTerminalArgs`.
6. `TerminalSettings::CreateWithNewTerminalArgs()` copies `restoredTabId` into the resolved settings object.
7. `TerminalPage::_CreateConnectionFromSettings()` passes `restoredTabId` into `ConptyConnection::CreateSettings()`.
8. `ConptyConnection::Initialize()` stores the value.
9. `ConptyConnection::_LaunchAttachedClient()` injects `WT_RESTORED_TAB_ID` into the child environment and includes it in `WSLENV`.
10. The restored child process can read `WT_RESTORED_TAB_ID`.

## Exact file-level implementation plan

### 1) `src/cascadia/TerminalSettingsModel/ActionArgs.idl`

Add a new property on `NewTerminalArgs`:

- `String RestoredTabId;`

Reason:

- this is the persisted payload shape used by restore
- it keeps the identifier attached to the existing `NewTerminalArgs` object model

### 2) `src/cascadia/TerminalSettingsModel/ActionArgs.h`

Update `implementation::NewTerminalArgs` to include:

- `ACTION_ARG(winrt::hstring, RestoredTabId, L"");`
- `static constexpr std::string_view RestoredTabIdKey{ "restoredTabId" };`

Update these methods so `RestoredTabId` participates in the same way as the other persisted properties:

- `Equals`
- `FromJson`
- `ToJson`
- `Copy`
- `Hash`

Do **not** treat this as `Guid`; keep it as `String` to avoid brace-format churn and to keep the persisted value identical to the environment variable payload.

### 3) `src/cascadia/TerminalSettingsModel/ActionArgs.cpp`

Update `NewTerminalArgs::ToCommandline()`:

- emit `--restoredTabId "<value>"` when `RestoredTabId()` is non-empty

Place it near the other WT-owned launch metadata (`--sessionId` is the closest existing analogue).

### 4) `src/cascadia/TerminalApp/AppCommandlineArgs.h`

Extend `NewTerminalSubcommand` with:

- `CLI::Option* restoredTabIdOption;`

Add parser state storage:

- `std::string _restoredTabId;`

Ensure reset logic clears it alongside the other `NewTerminalArgs` parser fields.

### 5) `src/cascadia/TerminalApp/AppCommandlineArgs.cpp`

Update `_addNewTerminalArgs()`:

- add an internal `--restoredTabId` option

Update `_getNewTerminalArgs()`:

- when the option was supplied, set `args.RestoredTabId(...)`

This is required so WT-internal relaunches do not drop the identifier.

### 6) `src/cascadia/TerminalApp/Tab.cpp`

Update `Tab::BuildStartupActions(BuildStartupKind kind)`.

Implementation rule:

- only for `BuildStartupKind::Persist`
- only when the tab is a single terminal tab
- only when `newContentArgs` is `NewTerminalArgs`
- do not stamp anything for split tabs or settings tabs

Concrete implementation approach:

1. Build the pane startup state as today.
2. Before constructing the top-level `NewTabArgs`, inspect the state:
   - `state.args.empty()` must still be true before the top-level `newTab` insertion; this identifies the one-pane/no-splits case.
   - `newContentArgs.try_as<NewTerminalArgs>()` must succeed.
3. Generate a fresh GUID via the existing utility path and convert it to plain string form.
4. Set `terminalArgs.RestoredTabId(...)`.

This is the key v1 scope gate. If the tab needed split actions, leave `RestoredTabId` unset.

### 7) `src/cascadia/TerminalSettingsAppAdapterLib/TerminalSettings.h`

Add a non-UI setting slot for the new value, parallel to the existing session/elevation plumbing:

- `SIMPLE_OVERRIDABLE_SETTING(hstring, RestoredTabId, L"");`

Reason:

- `TerminalPage::_CreateConnectionFromSettings()` receives resolved settings, not raw `NewTerminalArgs`
- this mirrors the existing `SessionId` flow

### 8) `src/cascadia/TerminalSettingsAppAdapterLib/TerminalSettings.cpp`

Update `TerminalSettings::CreateWithNewTerminalArgs()`:

- if `newTerminalArgs.RestoredTabId()` is non-empty, copy it to `defaultSettings->_RestoredTabId`

No other settings resolution logic should depend on it.

### 9) `src/cascadia/TerminalConnection/ConptyConnection.idl`

Extend `ConptyConnection::CreateSettings(...)` with an additional `String restoredTabId` parameter.

Keep this as the final/additional launch metadata argument so the call sites stay easy to audit.

### 10) `src/cascadia/TerminalConnection/ConptyConnection.h`

Update the `CreateSettings(...)` declaration to take `restoredTabId`.

Add a private field:

- `hstring _restoredTabId{};`

### 11) `src/cascadia/TerminalConnection/ConptyConnection.cpp`

Update `CreateSettings(...)`:

- insert `restoredTabId` into the `ValueSet` when non-empty

Update `Initialize(...)`:

- read `L"restoredTabId"` into `_restoredTabId`

Update `_LaunchAttachedClient()`:

- if `_restoredTabId` is non-empty, add `WT_RESTORED_TAB_ID` to the environment map
- add `WT_RESTORED_TAB_ID` to the built-in `WSLENV` propagation list

Do not set `WT_RESTORED_TAB_ID` when `_restoredTabId` is empty.

### 12) `src/cascadia/TerminalApp/TerminalPage.cpp`

Update `_CreateConnectionFromSettings(...)`:

- pass `settings.RestoredTabId()` into `ConptyConnection::CreateSettings(...)`

This should be done for the normal ConPTY path. No extra behavior is needed beyond passing the value through.

### 13) Tests

#### `src/cascadia/UnitTests_SettingsModel/CommandTests.cpp`

Add `NewTerminalArgs::ToCommandline()` coverage for `restoredTabId`.

Expected assertion shape:

- `--restoredTabId "..."` appears when set
- it is omitted when unset

#### `src/cascadia/LocalTests_TerminalApp/CommandlineTest.cpp`

Add parser coverage for `--restoredTabId`:

- parse a `new-tab --restoredTabId <id>` command
- verify the resulting `NewTerminalArgs.RestoredTabId()` matches exactly

#### `src/cascadia/LocalTests_TerminalApp/TabTests.cpp` (or the closest existing terminal-app restore test area)

Add coverage for the persist stamping rule:

- single terminal tab + `BuildStartupKind::Persist` => `restoredTabId` present
- split tab + `BuildStartupKind::Persist` => `restoredTabId` absent
- non-persist build path => `restoredTabId` absent

If an existing restore-focused test file is better suited, use that instead; the important part is to pin the v1 gating behavior.

## Supported cases in v1

- Persisted restore of a tab that contains exactly one terminal pane.
- ConPTY-backed restored terminals whose child process can read environment variables.
- WSL terminals, provided `WT_RESTORED_TAB_ID` is added to `WSLENV`.
- WT-internal relaunch paths that round-trip through `ToCommandline()` / `AppCommandlineArgs`.

## Unsupported cases in v1

- **Any split-pane tab.**
- Multi-terminal tabs of any kind.
- Non-terminal content tabs such as settings UI.
- Any expectation that `WT_RESTORED_TAB_ID` is a general stable tab ID outside persisted restore.
- Backfilling this identifier onto already-running tabs that were not created via persisted restore.

For unsupported cases, WT should simply omit `restoredTabId` and therefore omit `WT_RESTORED_TAB_ID`.

## Validation plan

### Automated

1. Run the updated unit/local tests covering:
   - `ActionArgs` JSON/commandline behavior
   - commandline parse round-trip
   - tab persist stamping rules
2. Verify no existing `sessionId` behavior regresses.

### Manual

1. Launch WT with a single standard terminal tab.
2. Enable persisted restore in local settings if needed.
3. Close WT so layout is persisted.
4. Inspect the persisted layout payload in local state and confirm the restored `newTab` action contains `restoredTabId`.
5. Restart WT and, inside the restored shell, verify `WT_RESTORED_TAB_ID` is set.
6. Repeat with WSL and verify the variable is present inside Linux.
7. Create a split tab, persist/restore, and verify `WT_RESTORED_TAB_ID` is **not** set in either pane.

## Risks

- `ToCommandline()` / parser plumbing makes the field externally invocable even though it is intended as internal restore metadata.
- If future restore work changes how tabs are serialized, the single-pane detection rule in `Tab::BuildStartupActions()` may need to move.
- The v1 behavior is intentionally asymmetric: supported tabs get the variable; split tabs do not.

## Follow-ups

- Define whether a future split-pane design should use one tab-level ID, one pane-level ID, or both.
- Decide whether WT should eventually surface restored-tab metadata through a more formal API instead of environment variables alone.
- If more restore-only launch metadata is added later, consider grouping it rather than continuing to expand `NewTerminalArgs` one field at a time.

## Implementation notes for reviewers

- This plan deliberately reuses the existing persisted action-list architecture instead of introducing a new saved-tab object model.
- The scope gate for v1 is **where the identifier is stamped**: only one terminal per tab, no splits.
- The environment variable contract for the restored child process is exactly `WT_RESTORED_TAB_ID`.
