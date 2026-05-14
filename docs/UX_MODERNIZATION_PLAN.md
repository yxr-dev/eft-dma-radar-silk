# UX Modernization Plan

Living plan for modernizing the EFT DMA Radar (Silk.NET) desktop UI and web client.

Audience: this app is mostly used by:
- Players on a **controller / second keyboard+mouse via InputManager**.
- Players viewing through **AnyDesk / remote desktop** (high latency, lossy input, often on a TV).
- Other players using the **web client**, auto-centered on their own player (often on phone/tablet).

Therefore every UX decision must favor:
- **Few inputs, big targets, low-latency interactions.**
- **No required dragging or precise mouse work.**
- **Hotkey-first** (configurable through `HotkeyManager`), controller-friendly.
- **Parity between desktop and web** in mental model, different shells.

---

## Guiding principles

1. **Hotkey-first, mouse-optional.** Every common action reachable via one hotkey / controller button / one big click.
2. **No floating windows for primary panels.** Use a fixed shell (sidebar + dock) on desktop, bottom sheets on web.
3. **Progressive disclosure.** Quick toggles always visible. Advanced settings collapsed by default.
4. **One accent color, one radius, one border, one focus style.** Use the existing cyan as the only active-state color.
5. **Presets over micromanagement.** A user should be able to switch profiles instead of toggling 30 options every raid.
6. **Auto-follow is the default.** Free-pan is opt-in and auto-recenters after a short idle.

---

## Phases

### Phase 1 — Foundation (small, safe, high impact)
- [x] **Preset system**: data model in `SilkConfig` (`ActivePresetId`) + `UI/Presets/PresetManager.cs` with built-ins `Stealth`, `LootRun`, `PvP`, `Quests`, `Custom`. Presets touch only radar-layer & player-display toggles.
- [x] **Top-bar preset selector** in `RadarWindow.ImGui.cs` (combo + drift reconciliation).
- [x] **Cycle hotkeys** `PresetCycleNext` / `PresetCyclePrev` registered via `HotkeyManager` (user binds keys in the Hotkeys panel; defaults to unbound so they don't clash).
- [x] **Big-chip status bar** at the bottom consolidating: raid state, players, vitals, FPS, DMA, map — each piece rendered as a discrete label+value chip for AnyDesk / TV readability.

### Phase 2 — Layout
- [x] **Left sidebar** (icon nav) for: Players, Loot, Aimview, Quests, Settings (+ ESP). Icons fall back to bold letters so they render in the default ImGui font (no extra glyph ranges required).
- [x] **Right dock** for Players / Loot / Quests via `UI/Shell/RightDock.cs`: a layout-only helper that forces the existing widget windows into a stacked column on the right edge when `SilkConfig.DockSidePanels` is on. The widgets keep their content/visibility logic; only their position+size are taken over.
- [x] Panel hotkeys: `1` Players, `2` Loot, `3` Aimview, `4` Quests, `5` Settings, sidebar visibility toggle, **dock toggle** — all registered as configurable actions (`SidebarSlot1..5`, `ToggleSidebar`, `ToggleSidePanelsDock`).
- [x] **Aimview as fixed PiP**: Off / S / M / L cycled via `AimviewCycleSize`; corner cycled via `AimviewCycleCorner`. PiP now also respects the right-dock width so it never overlaps the dock.
- [x] Map always centered and never covered by primary panels (sidebar reserves left edge, right dock reserves right edge — map renders between).
- [x] **Big-chip status bar** redesign landed alongside the settings rebuild — see `RadarWindow.ImGui.cs::DrawStatusBar` / `DrawChip`.
- [x] **Collapsible sidebar & status bar**: sidebar has a bottom `<` chevron and leaves a thin `>` handle on the left edge when hidden; status bar has a `v` chevron and leaves a small `^` handle in the bottom-right corner when hidden (`SilkConfig.ShowStatusBar`). Sidebar now sizes against the actual status bar height (`Sidebar.StatusBarHeight`) so it no longer draws behind it.

### Phase 3 — Settings rebuild
- [x] New `SettingsPanel` shell: window grown to 720×640 with a left category nav + scrolling content pane. Tab bodies (`GeneralTab`/`PlayersTab`/`EspTab`/`MapTab`/`QuestZonesTab`/`HotkeysTab`/`MemWritesTab`) are now plain content renderers dispatched from `_categories`.
- [x] Replace sliders with `– value +` **steppers**: `UIControls.Stepper` / `UIControls.StepperFloat` landed; migration completed across `GeneralTab` (UI Scale, Target FPS), `PlayersTab` (Aimline Length, all Aimview tuning ranges, distance-aware tuning), `EspTab` (Target FPS, player/loot distance, crosshair scale), `MapTab` (zoom, loot dot/font, height threshold, killfeed max/TTL, door proximity), `QuestZonesTab` (max distance), and `MemWritesTab` (recoil/sway %, lean amount, long-jump multiplier, move-speed multiplier, full-bright brightness, extended reach).
- [x] **Full-width toggle rows** (≥36px), whole row clickable/focusable: `UIControls.ToggleRow` landed across every settings category.
- [x] **Combo rows** (`UIControls.ComboRow`): full-width row with chunky `<` / `>` arrows + a clickable centered value (opens a stock combo as a keyboard / mouse fallback). Migrated `EspTab` (Render Mode, Crosshair Style) and `MemWritesTab` (Wide Lean Direction). Arrows auto-repeat while held so AnyDesk / controller users can cycle quickly.
- [x] Collapse rarely-used controls into an **Advanced** section per category. `UIControls.BeginAdvanced` / `EndAdvanced` landed; applied to `PlayersTab` (aimview tuning sliders, distance-aware multipliers) and `MapTab` (Map Setup calibration). More categories can adopt as needed.
- [x] **Focus navigation**: `ImGuiConfigFlags.NavEnableKeyboard` + `NavEnableGamepad` enabled in `RadarWindow.Initialization.cs`, so arrow keys / D-pad navigate rows, left/right adjusts steppers, and `Esc` clears focus. Full radial / `B`-to-go-back wiring deferred to Phase 4 quick menu.
- [x] Cyan focus ring on the focused element: `ImGuiCol.NavCursor` and `NavWindowingHighlight` styled in `ApplyImGuiDarkStyle()` so controller / keyboard / remote users get a clearly visible focus indicator.

### Phase 4 — Quick menu
- [x] **Radial menu** opened by the `QuickMenuOpen` hotkey (bind `Q` on keyboard or `LB` on controller in the Hotkeys panel). Implemented in `src-silk/UI/Shell/QuickMenu.cs` with 8 slices for in-raid toggles: Battle Mode, Aimlines, Loot, High Alert, Connect Groups, Exfils, Doors, Airdrops.
- [x] Hold-to-open / release-to-confirm: handler is wired to both key-down and key-up edges so the radial behaves like a controller LB-hold radial — hold, point with mouse/stick, release on a slice to toggle it. Active slices show a cyan ring + dot so state is readable at a glance.
- [x] **Command palette** `Ctrl+K` as a power-user fallback (keyboard-only). Implemented in `src-silk/UI/Shell/CommandPalette.cs` — fuzzy-searches every registered hotkey action plus a set of "Open panel" entries (Settings, Hotkeys, Loot Filters, Killfeed, Player History, Watchlist, Quest Planner, Hideout) and runs them with Enter.

### Phase 5 — Web client (`src-silk/Web/wwwroot/`)
- [x] **Bottom tab bar**: Players · Loot · Layers · Settings — fixed nav at the bottom of the viewport with large touch targets and cyan active state.
- [x] **Bottom sheets** for Players / Loot (Layers / Settings too) — slide up from above the tab bar with a drag-handle, swipe-down-to-dismiss, max-height 62vh (70vh on phone). Sheet rows are declarative (toggle / range / chips / select / button / section / hint) and proxy to the legacy sidebar inputs so no state is duplicated.
- [x] **Follow-me default** — `sidebarCollapsed` default flipped to `true` so first-load shows the bottom shell; `freeMode` stays `false`. **Double-tap recenter** added on both `touchend` (within 320ms / 24px) and `dblclick` (empty map space); existing pinch-to-zoom on the map preserved.
- [x] **FAB radial** mirroring the desktop QuickMenu: hold the FAB → radial opens at center; drag onto a slice → release confirms (controller-LB-style). Short-tap also opens the radial for click-tap users; tapping the backdrop closes. 8 slices: Aim · Loot · Exfils · Doors · Airdrop · Containers · Names · Quests.
- [x] Stripped settings: the Settings sheet exposes only Zoom · Poll Rate · Free Mode · Center on Local · Hover-open Sidebar · Reset, plus an `Open Advanced Sidebar` button that re-opens the legacy right sidebar for power users.
- [x] **Web-client presets** (independent of the desktop host): `WEB_PRESETS` in `app.js` defines four buddy-focused presets — `Spotter`, `Battle Buddy`, `Loot Hunter`, `Quest Helper` — plus `Custom`. Each preset is a partial state object that's applied by dispatching `change` events on the existing legacy sidebar inputs (so persistence + render pipelines stay identical). The top-center `PRESET · <name>` chip now shows the **web** preset and is tap-to-cycle; the Settings sheet has a chip-row selector. Drift detection (`reconcileWebPresetDrift`) runs as a microtask after every config change and demotes to Custom if any tracked key no longer matches. The host's `WebRadarUpdate.ActivePreset` field is still emitted but the web client no longer surfaces it — buddies pick their own view.

### Phase 6 — Polish
- [x] Unified `UITheme` (single accent, radius, border, focus style) — `src-silk/UI/UITheme.cs` now owns the accent (cyan), the three radii (`RadiusSmall=4`, `RadiusMedium=6`, `RadiusLarge=10`), the three border weights (`BorderThin/Default/Focus`), and the `FocusRing` color. `ApplyImGuiDarkStyle()` in `RadarWindow.ImGui.cs` reads these constants instead of inlining magic numbers, so the desktop now has one source of truth for "what does an interactive surface look like".
- [x] **Toast notification system** (`src-silk/UI/Shell/ToastManager.cs`) — bottom-right stacked, fade in/out, severity colors, coalesces duplicates, never steals input. Wired so preset changes emit `ToastManager.Info("Preset: …")` and `NotifyConfigSaved()` emits a success toast alongside the existing chip indicator. Anchors above the status bar (respects collapsed/handle height).
- [x] **First-run tour** (`src-silk/UI/Shell/FirstRunTour.cs`) — 5 cards highlighting Sidebar / Status bar / Presets / Quick menu + Command palette. Persists completion via `SilkConfig.FirstRunTourCompleted`. Reachable any time from the General settings page (`✨ Show Welcome Tour` button) or the command palette (`Show Welcome Tour`). Driven entirely by ImGui — never touches the player render path.
- [x] **Players-first status chip**: `PLAYERS` chip in the bottom status bar now shows `total · {T}T {P}P {S}S {AI}AI` — teammate / PMC / player-scav / AI counts segmented inline. Cached so the string only rebuilds when the underlying counts change.
- [x] **Top command bar redesign** (`RadarWindow.ImGui.cs::DrawMainMenuBar`): replaced the legacy `View` + `Windows` text dropdowns with a pill-style command bar. New helpers `TopBarPill` / `TopBarDivider` / `DrawFollowFreePill` / `DrawMorePopup` / `DrawTopBarRightInfo` share the chip language of the bottom status bar (rounded fill, cyan accent for active state). Layout: Follow/Free · Battle · Preset · │ · Aim · Loot · Exfils · │ · Restart · More · (right) Map · FPS. Every previously-available action lives either in a top-level pill, the Preset combo, or the `⋯ More` popup (rarely-used layer toggles, secondary panels, Command Palette, Close All).
- [x] Tooltips with hotkey hints on every toolbar button — `HotkeyManager.GetBindingDisplay(actionId)` / `HotkeyManager.WithHint(desc, actionId)` look up the user's current binding (so the hint matches whatever they bound, not a hard-coded letter). Wired into `RadarWindow.ImGui.cs::DrawMainMenuBar` for Follow/Free, Battle, Loot, Exfils, and into `DrawMorePopup` for Doors / Connect Groups. Sidebar slot tooltips already showed `[1]`–`[5]` / `[E]` / `[Tab]`. Restart pill still has a contextual tooltip (no hotkey); Aim has no dedicated action so its tooltip stays bare.

---

## Open questions (need user input before implementation)

1. **Preset contents** — bundle just radar-layer toggles, or also ranges (player/loot), min loot value, ESP toggles, and memory-write features?
2. **Built-in presets** — keep `Stealth / Loot Run / PvP / Quests / Custom`, or add per-map ones (`Streets Night`, `Labs`)?
3. **Hotkeys** — are `[` `]` `Q` `Tab` `V` `1-4` free in `HotkeyManager`? Any conflicts?
4. **Storage** — store presets inside `SilkConfig` or in a separate `presets.json` so users can share/import?
5. **Web parity** — should web show the preset selector and switch via WebSocket, or is web read-only on presets initially?
6. **Execution mode** — implement phase-by-phase with review between phases, or autonomous through Phase 6?

---

## Status

- **Phase 1 (Foundation)**: preset system + top-bar selector + cycle hotkeys landed.
  - New: `src-silk/UI/Presets/PresetManager.cs`, `SilkConfig.ActivePresetId`, `PresetCycleNext` / `PresetCyclePrev` hotkey actions.
  - Status-bar redesign deferred to Phase 2 (will be done with the layout pass).
  - Decisions taken without user input (revisit if wrong):
    - Presets bundle only radar-layer + player-display toggles (no ranges, no ESP, no mem-writes).
    - Built-ins: Stealth / LootRun / PvP / Quests / Custom.
    - Cycle hotkeys default to unbound to avoid clashing with existing bindings; user binds them in the Hotkeys panel.
    - Storage: inside `SilkConfig` (no separate `presets.json` yet).
    - Web parity: not wired yet.
  - **Differentiation fix**: Stealth and PvP previously had identical toggles (so cycling between them was a no-op). The four presets now each bundle a distinct set of nine toggles (Battle, Loot, **Corpses**, **Containers**, Exfils, Doors, Airdrops, Switches, **Transits**) plus three player-display toggles (Aimlines, ConnectGroups, HighAlert, **PlayersOnTop**) — Stealth = silent extract / no clutter / players-on-top, Loot Run = max info / HighAlert off / players-not-on-top, PvP = hunter mode / corpses + doors + airdrops on / players-on-top, Quests = objectives only / no corpses or containers. `PresetManager.Apply()` and `MatchesActive()` cover all the new fields so drift detection still snaps to Custom correctly.
- **Phase 2 (Layout)** — landed.
  - Sidebar: `src-silk/UI/Shell/Sidebar.cs` (icon nav with letter glyphs, hotkey badges, active indicator, ESP slot).
  - Right dock: `src-silk/UI/Shell/RightDock.cs` — non-invasive layout helper that pins Players/Loot/Quests to the right edge in a stacked column the FIRST time each panel opens (uses `ImGuiCond.FirstUseEver`). After that the user can freely drag and resize them; ImGui persists positions in `imgui.ini`. A "Reset Side Panels" hotkey re-snaps them to the docked layout.
  - Aimview PiP: also drag-/resize-able after initial snap; "Cycle Aimview Size/Corner" hotkeys re-snap it.
  - `SilkConfig`: `ShowSidebar`, `DockSidePanels`, `RightDockWidth` (default 460), `AimviewPipSize` (0=Floating/1=S/2=M/3=L), `AimviewPipCorner` (0=TL/1=TR/2=BR/3=BL, default BR).
  - `AimviewWidget` PiP respects both sidebar and right-dock widths.
  - `HotkeyManager` adds: `ToggleSidebar`, `ToggleSidePanelsDock`, `SidebarSlot1..5`, `AimviewCycleSize`, `AimviewCycleCorner` (all unbound by default).
  - Status-bar redesign deferred to be done alongside Phase 3 (settings rebuild) for a single visual pass.
- **Next**: All six phases of the plan have landed in `src-silk`. Future polish items can be tracked as new entries below this line.
- **Phase 3 (Settings rebuild)** — landed.
  - New: `src-silk/UI/Controls/UIControls.cs` — controller/AnyDesk-friendly primitives:
    - `UIControls.Section(label)` — bold cyan section header replacing `ImGui.SeparatorText`.
    - `UIControls.ToggleRow(label, ref bool, tooltip?)` — full-width (≥36px scaled) clickable row with a pill switch; whole row is hit-testable.
    - `UIControls.Stepper(label, ref int, min, max, step, format?, tooltip?)` and `StepperFloat(...)` — `– value +` clusters with 32×32 (scaled) buttons; auto-repeat while held.
    - `UIControls.ComboRow(label, ref int, options, tooltip?)` — full-width row with chunky `<` / `>` arrows + clickable center value (opens a stock combo as keyboard fallback).
    - `UIControls.BeginAdvanced` / `EndAdvanced` — collapsed-by-default group for power-user controls.
  - Settings window grown to 720×640 to give the new row-based layout room.
  - Left-nav + content-pane shell landed: `SettingsPanel._categories` drives a vertical list of big icon+label buttons on the left and routes the selected category into a scrolling content child on the right. Tab bodies converted to plain content renderers (no more `BeginTabItem`/`EndTabItem`).
  - **Slider / combo migration completed across every category**:
    - `GeneralTab`: UI Scale, Target FPS (`Stepper`).
    - `PlayersTab`: Aimline Length, all Aimview tuning (Player/Loot range, Eye Height, Zoom, Min Loot ₽, Max Loot/Corpses/Containers), and Distance-aware tuning (Near/Mid ranges, Gear/Hands Mid/Far multipliers) — all `Stepper` / `StepperFloat`.
    - `EspTab`: Target FPS, player + loot Max Distance, Crosshair Scale → steppers. Render Mode and Crosshair Style → `ComboRow`.
    - `MapTab`: Zoom, Loot Dot Size, Label Font, Height Threshold, Killfeed Max Entries + TTL, Door Proximity → steppers.
    - `QuestZonesTab`: Max Distance → `StepperFloat`.
    - `MemWritesTab`: Recoil %, Sway %, Wide Lean Amount, Long Jump Multiplier, Move Speed Multiplier, Full Bright Brightness, Extended Reach Distance → steppers. Wide Lean Direction → `ComboRow`.
  - Advanced collapses: `PlayersTab` aimview-tuning + distance-aware multipliers, `MapTab` Map Setup (Calibration).
  - Focus navigation + focus ring landed: keyboard and gamepad nav flags both enabled in `RadarWindow.Initialization.cs`, and `ApplyImGuiDarkStyle()` sets a bright cyan `ImGuiCol.NavCursor` (plus matching `NavWindowingHighlight`).
  - Big-chip status bar redesign landed: status / players / vitals on the left, FPS / DMA / map on the right, each rendered as a discrete label+value chip via `DrawChip`. Save notification is now a transient chip too.
- **Phase 4 (Quick menu)** — radial menu landed.
  - New: `src-silk/UI/Shell/QuickMenu.cs` and `QuickMenuOpen` hotkey action (defaults to unbound — user binds `Q` / `LB` in the Hotkeys panel).
  - 8 slices: Battle Mode, Aimlines, Loot, High Alert, Connect Groups, Exfils, Doors, Airdrops. Hold to open, release on a slice to toggle; active slices show a cyan dot so state is readable at a glance.
  - Command palette landed: `src-silk/UI/Shell/CommandPalette.cs` opens on `Ctrl+K` (handled directly through ImGui input so it does not need a separate hotkey binding), supports arrow keys + Enter, and invokes both hotkey actions and synthetic "Open panel" commands.
- **Local shortcut layer** — `RadarWindow.ImGui.cs::HandleLocalShortcuts` runs first each frame so panel letters (`S/P/L/T/A/E/H/Q`), sidebar slots (`1`–`5`), `Tab` (toggle sidebar), and `Ctrl+K` work on the radar PC regardless of which sub-window has focus. Modifier-only combos and active text inputs are ignored. When the Settings panel is open, its category glyphs (`G/P/E/M/Q/K/W`) take priority via `SettingsPanel.HandleCategoryShortcuts` so they switch tabs instead of toggling underlying panels.
- **Sidebar secondary slots** — the sidebar grew a second tier below the five primary buttons so every other panel is reachable in a single click instead of buried in the `⋯ More` popup. New secondary items: `Loot Filters` (`L`), `Killfeed`, `Hideout` (`H`), `Quest Planner`, `Player History`, `Watchlist`, `Hotkeys`. Each is drawn at 30px (vs 44px for the primary five) with icon-only buttons and tooltip hints; a thin separator splits the two tiers. ESP and the collapse chevron stay anchored at the very bottom.
- **Loot Filters panel refresh** — `LootFiltersPanel` switched to `UIControls.Section` / `ToggleRow` / `Stepper` / `ComboRow` so it matches the Phase 3 settings-panel language. Steppers (5K / 10K rouble step, auto-repeat) replace the fiddly `DragInt` price sliders. The Show Loot toggle, Important Only, the five always-show categories (Meds / Food / Backpacks / Keys / Wishlist), and the Options section (Price Source / Price Per Slot / Show Corpses) are all full-width clickable rows now. New **Quick View** chip row at the top — four one-tap modes (`All Loot`, `Important+`, `Wishlist`, `Quest`) snap a sensible combination of flags; the currently-matching mode is highlighted cyan.
- **Phase 5 (Web client)** — landed.
  - New mobile-first bottom shell layered on top of the legacy right sidebar:
    - `index.html` adds a top-center `#presetChip`, a `#bottomBar` nav (Players · Loot · Layers · Settings), a `#bottomSheet` slide-up panel with a drag-grabber + close X, a `#fab` quick-toggle FAB, and a `#radialOverlay` for the radial menu.
    - `app.css` adds the full bottom-shell stylesheet — single-accent cyan (`--accent: #34d4d4`), 64–68px tab bar with safe-area-inset padding, sheets capped at 62vh / 70vh on phone, FAB above the bar, full-screen radial overlay with 8 circular slices, plus media-query bumps for ≤480px.
    - `app.js` appends a self-contained mobile-shell module: declarative `SHEET_DEFS` for each tab; row builder that PROXIES toggles / ranges / selects / chips back to the existing legacy inputs by ID and dispatches `input`/`change` so the existing `listen()`/`bind()` pipeline does all the persistence — no duplicated state. Sheet swipe-down to dismiss; tab click toggles; the open sheet auto-refreshes when any underlying input changes (so the FAB radial / sidebar / sheet stay in sync).
    - FAB radial: hold-to-open / release-on-slice for touch (mirrors desktop QuickMenu's LB-hold), with short-tap → toggle-open fallback for click users. 8 web-relevant actions: Aim, Loot, Exfils, Doors, Airdrop, Containers, Names, Quests. The window-level mousemove/up/touchmove/end handlers are gated by a `_radialPressActive` flag so they only fire while the user is actually interacting with the FAB — without this, every click anywhere on the page opened the radial and blocked map / sheet input. Hold gestures also abandon if the user moves >14px before the timer fires (so a swipe over the FAB doesn't trap them), and the synthetic click that follows a touch release is suppressed for 500ms so the underlying canvas / sheet doesn't double-fire. Slices keep `pointer-events: none` and tap-mode hit-tests by position via `radialSliceAt`.
    - Double-tap recenter: empty-map double-tap (touch within 320ms / 24px, or desktop `dblclick`) clears `freeMode` and `followTarget` so the camera snaps back to the local player. Player-target double-click path preserved.
    - Preset sync: `WebRadarUpdate.ActivePreset` is set every tick in `WebRadarServer.cs::Worker` from `SilkProgram.Config.ActivePresetId`, and `fetchRadar()` calls `updatePresetChip(radarData.activePreset)` which shows / hides the top-center chip with a human label (Stealth / Loot Run / PvP / Quests / Custom).
    - Default flipped: `sidebarCollapsed` now defaults to `true` so first-load shows the bottom shell only. The legacy sidebar remains accessible via Settings → `Open Advanced Sidebar` (or the original hamburger).
- **Phase 6 (Polish)** — landed.
  - `ToastManager` (`src-silk/UI/Shell/ToastManager.cs`) lands the toast notification system: bottom-right stacked, fade in/out, severity colors (info/success/warn/error), duplicate coalescing, anchors above status bar / collapsed handle. Preset apply + config-save events now emit toasts.
  - **First-run tour** (`src-silk/UI/Shell/FirstRunTour.cs`) — 5-card walkthrough (Welcome → Sidebar → Status bar → Presets → Quick menu / Command palette). Auto-opens once per install via `SilkConfig.FirstRunTourCompleted`. Arrow / Enter / Space advance; Backspace / Left arrow back; Esc / Skip dismiss. Reachable any time from the General settings page (`✨ Show Welcome Tour`) or the command palette (`Show Welcome Tour`).
  - **Players-first status chip**: bottom-bar `PLAYERS` chip now segments the count by team — `total · {T}T {P}P {S}S {AI}AI` — to surface what the radar actually cares about at a glance. Counts are cached per-frame so the chip string only rebuilds when the underlying counts change. Player render/position pipeline is untouched.
  - **Top command bar pill redesign** (`RadarWindow.ImGui.cs`): the legacy `View` / `Windows` dropdown menus are gone. The bar is now a row of pill-style toggles in the same chip language as the bottom status bar:
    - Follow/Free mode pill (free state highlighted in cyan).
    - Battle Mode pill.
    - Preset selector (label + chip-styled combo).
    - Divider · Aim · Loot · Exfils quick-toggle pills · divider.
    - Restart icon pill (disabled outside raid/hideout).
    - `⋯ More` overflow popup containing everything not promoted to the bar: Doors / Airdrops / Switches, Connect Groups / High Alert, Loot Filters / Hotkeys / Hideout / Quest Planner / Player History / Watchlist, Command Palette shortcut, Close All.
    - Right cluster: `{map}  ·  {fps} FPS` mini-chip.
    - All cyan-on / dark-off states match the existing Phase 3 accent palette. Dead `ColorFreeModeBtn*` constants in `RadarWindow.cs` were removed.
  - **Unified `UITheme`** (`src-silk/UI/UITheme.cs`): centralizes the single accent (`Accent` / `AccentSoft` / `AccentFaint`), the radius family (`RadiusSmall=4`, `RadiusMedium=6`, `RadiusLarge=10`), the border weights (`BorderThin=1`, `BorderDefault=1.5`, `BorderFocus=2`), and the `FocusRing` color. `ApplyImGuiDarkStyle()` reads these constants instead of inlining magic numbers, so the desktop UI has one source of truth for accent / radius / border / focus across windows, popups, frames, and the nav cursor.
  - **Hotkey-hint tooltips on toolbar buttons** — new helpers `HotkeyManager.GetBindingDisplay(actionId)` and `HotkeyManager.WithHint(description, actionId)` look up the user's current binding (so the hint matches whatever they bound, not a hard-coded letter). Wired into the top-bar Follow / Battle / Loot / Exfils pills, and into the More popup's Doors / Connect Groups entries. Sidebar slot tooltips already showed bound letters via `Sidebar.cs`. Build verified clean (12 pre-existing warnings, 0 errors).

Update this file at the end of every change so progress is checkable from the repo.
