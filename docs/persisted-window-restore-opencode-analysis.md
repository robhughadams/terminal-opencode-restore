## Windows Terminal persisted restore analysis for OpenCode session restore

### Scope

This note captures what Windows Terminal (WT) persists today, and what that means for restoring OpenCode sessions if we assume **one terminal per tab** and **no splits** as the initial scope.

This document does **not** change user settings. In local testing, WT restore was enabled outside this repository via `firstWindowPreference=persistedWindowLayout` / the modern persisted-layout setting equivalent, but that settings change is only described here.

### Summary

WT restore is supported today, but the persisted model is a **replayable action list**, not a first-class saved tab/session object model. That makes basic restore workable, but any attempt to map restored tabs back to OpenCode sessions is necessarily **heuristic** rather than backed by a strong stable WT tab identity contract.

For the initial one-terminal-per-tab scope, a per-tab mapping looks viable if OpenCode treats the restored terminal as the unit of recovery and tolerates ambiguity.

## Supported functionality today

### Reliable today

- WT can persist window layouts into local app state under `persistedWindowLayouts`.
- On startup, WT can read persisted window layouts and recreate windows from them.
- Each persisted tab is reconstructed from one or more actions, with `newTab` as the first action for a normal terminal tab.
- For terminal content persisted with `BuildStartupKind::Persist`, WT includes `sessionId` in `NewTerminalArgs` when the underlying connection has one.
- Persisted window metadata includes window size, position, launch mode, focused tab restoration, and window rename state.

### Evidence

- `src/cascadia/TerminalApp/TerminalPage.cpp`: `TerminalPage::PersistState()` gathers each tab's `BuildStartupActions(BuildStartupKind::Persist)`, appends window-level actions, and writes a `WindowLayout` to `ApplicationState::PersistedWindowLayouts`.
- `src/cascadia/TerminalApp/Tab.cpp`: `Tab::BuildStartupActions()` explicitly serializes a tab as commands to recreate it, beginning with `ShortcutAction::NewTab` for normal terminal content.
- `src/cascadia/TerminalApp/TerminalPaneContent.cpp`: `GetNewTerminalArgs(BuildStartupKind::Persist)` copies the connection `SessionId()` into `args.SessionId(id)` when present.
- `src/cascadia/TerminalSettingsModel/ActionArgs.h` and `ActionArgs.cpp`: `NewTerminalArgs` defines `sessionId` serialization and emits `--sessionId` on the reconstructed command line.
- `src/cascadia/WindowsTerminal/WindowEmperor.cpp`: startup reads `ApplicationState::PersistedWindowLayouts()` and dispatches restored windows.
- `src/cascadia/TerminalApp/TerminalWindow.cpp`: persisted layouts are loaded by index when persisted layout usage is enabled.
- Local observation from `LocalState/state.json`: `persistedWindowLayouts` contained a window whose `tabLayout` contained `newTab` actions for the Debian WSL profile, confirming the runtime shape matches the source analysis.

## Unsupported or not strongly guaranteed

### Not a reliable WT contract

- WT does **not** persist a first-class stable tab object with a durable tab ID.
- WT restore is based on replaying actions, not rehydrating named tab records.
- There is no strong source-level evidence of a stable tab identifier intended for external consumers to use for exact tab-to-app-session reconciliation.

### Out of initial scope

- Split panes and multi-terminal tabs.
- Non-terminal tabs with special behavior (for example settings UI handling).
- Exact recovery of pane topology, focus semantics within a tab, or any OpenCode state that assumes more than one terminal object per tab.

### Fragile / heuristic areas

- A `sessionId` exists for the terminal connection, but this is not documented in code as a stable **tab identity** contract.
- A restored tab may be reproducible as a WT action sequence without giving OpenCode a guaranteed one-to-one durable tab identifier across restarts.
- Matching a restored WT tab back to an OpenCode session would need to rely on heuristics such as `sessionId`, command line, profile, starting directory, tab title, or a combination of those fields.

## Assumptions for OpenCode

- Initial scope is **one terminal per tab**.
- No split panes are considered.
- OpenCode only needs to restore at the terminal-tab level, not reconstruct arbitrary WT pane graphs.
- OpenCode can tolerate heuristic matching and can define a fallback when an exact match is not possible.

Under these assumptions, the practical unit of restore is the persisted `newTab` action plus its serialized terminal arguments, not a WT-owned stable tab record.

## Implications for OpenCode session restore

### What looks viable

- Use one WT tab as one OpenCode session container.
- Treat WT persisted restore as the mechanism that reopens the terminal.
- Attempt to associate the reopened terminal with an OpenCode session using persisted terminal attributes, with `sessionId` as the best available candidate when present.

### What remains heuristic

- Whether a restored tab corresponds to the exact same logical OpenCode session as before shutdown.
- Whether `sessionId` is stable enough across the relevant lifecycle for OpenCode to depend on it long-term.
- Whether future WT changes to persisted action generation preserve the same mapping behavior.

## Reliable vs heuristic summary

### Reliable today

- WT persists and reloads window/tab layout state.
- Persisted tab state is represented as replayable actions.
- Normal terminal tabs restore through `newTab` actions.
- Persisted terminal args can include `sessionId`.

### Heuristic / fragile

- Using WT persisted restore as a durable OpenCode session identity layer.
- Assuming `sessionId` is equivalent to a stable tab ID.
- Any mapping beyond the single-terminal-per-tab model.

## Source references

- `src/cascadia/TerminalApp/TerminalPage.cpp` - `TerminalPage::PersistState`
- `src/cascadia/TerminalApp/Tab.cpp` - `Tab::BuildStartupActions`
- `src/cascadia/TerminalApp/TerminalPaneContent.cpp` - `GetNewTerminalArgs(BuildStartupKind::Persist)`
- `src/cascadia/TerminalSettingsModel/ActionArgs.h` - `NewTerminalArgs::SessionIdKey`
- `src/cascadia/TerminalSettingsModel/ActionArgs.cpp` - `NewTerminalArgs::ToCommandline`
- `src/cascadia/WindowsTerminal/WindowEmperor.cpp` - persisted window restore on startup
- `src/cascadia/TerminalApp/TerminalWindow.cpp` - persisted layout loading
- `src/cascadia/TerminalSettingsModel/TerminalSettingsSerializationHelpers.h` - `firstWindowPreference` enum mapping
- `src/cascadia/TerminalSettingsModel/GlobalAppSettings.cpp` - `ShouldUsePersistedLayout()` and legacy setting fixup

## Recommended next steps

1. Validate whether `sessionId` survives exactly as needed across the OpenCode restore lifecycle, rather than assuming it is durable enough.
2. Prototype restore using the strict one-terminal-per-tab model only.
3. Define a fallback matching strategy when `sessionId` is missing or insufficient (for example profile + command line + starting directory).
4. Keep split panes explicitly unsupported until the single-tab path is proven.
5. If OpenCode needs stronger guarantees, treat WT persisted restore as a best-effort transport and maintain OpenCode's own session identity separately.
