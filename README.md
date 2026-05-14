# EFT DMA Radar — Silk.NET Edition

A modern DMA (Direct Memory Access) radar overlay for **Escape from Tarkov**, built on [Silk.NET](https://github.com/dotnet/Silk.NET) (Windowing / Input / OpenGL), [ImGui.NET](https://github.com/ImGuiNET/ImGui.NET) panels, and [SkiaSharp](https://github.com/mono/SkiaSharp) 2D rendering. Ships with an embedded ASP.NET Core web radar for browser / phone / tablet buddies.

This repo is the **Silk variant only**, split out from a larger multi-front-end monorepo for easier consumption and contribution.

---

## Repo Layout

```
eft-dma-radar-silk/
├── eft-dma-radar-silk.sln       # Visual Studio solution (VmmSharpEx + src-silk)
├── Directory.Build.props        # Common MSBuild props (net10.0-windows, x64, unsafe)
├── version.json                 # Nerdbank.GitVersioning version source
├── LICENSE                      # BSD Zero Clause License
├── Maps/                        # EFT map SVGs + JSON metadata (Customs, Streets, …)
├── Resources/                   # Embedded font + default item DB
├── lib/
│   └── VmmSharpEx/              # Managed MemProcFS / LeechCore wrapper + native DLLs
├── docs/
│   └── UX_MODERNIZATION_PLAN.md # Phase-by-phase modernization log
└── src-silk/                    # The radar itself (entry: Program.cs → SilkProgram.Main)
```

---

## Requirements

- **DMA hardware** supported by [MemProcFS](https://github.com/ufrisk/MemProcFS) (FPGA card, `usb3380`, etc.)
- **Windows 10 / 11 (x64)** — project targets `net10.0-windows`, `PlatformTarget=x64`
- **[.NET 10 SDK / Runtime](https://dotnet.microsoft.com/en-us/download/dotnet/10.0)**
- **Visual Studio 2022 17.12+** (or 2026 Insiders) with the **.NET desktop development** workload
- The native MemProcFS binaries (`vmm.dll`, `leechcore.dll`, `FTD3XX.dll`, …) ship under `lib/VmmSharpEx/native/` and are copied to the build output automatically.

---

## Build & Run

```powershell
git clone https://github.com/HuiTeab/eft-dma-radar-silk.git
cd eft-dma-radar-silk

# Build (Release, x64)
dotnet build eft-dma-radar-silk.sln -c Release

# Run
dotnet run --project src-silk\eft-dma-radar.csproj -c Release
```

Pass `-debug` on the command line (or set `debugLogging=true` in the config) for verbose startup logging.

In Visual Studio: open `eft-dma-radar-silk.sln`, set `eft-dma-radar` as the startup project, press **F5**.

---

## Highlights

A six-phase UX modernization is logged in [`docs/UX_MODERNIZATION_PLAN.md`](docs/UX_MODERNIZATION_PLAN.md). Everything below is in the repo today:

**Desktop shell**
- **Icon sidebar** with two tiers — five primary panels (Players · Loot · Aimview · Quests · Settings) plus a compact secondary row (Loot Filters · Killfeed · Hideout · Quest Planner · Player History · Watchlist · Hotkeys) and ESP at the bottom. Every slot is a single click; hotkey hints come from the user's actual binding via `HotkeyManager.GetBindingDisplay`.
- **Top command bar** — pill-style toggles in the same chip language as the bottom status bar: Follow/Free · Battle · Preset · Aim/Loot/Exfils · Restart · More. Right cluster shows current map + FPS.
- **Big-chip status bar** at the bottom: raid state · players (segmented `T/P/S/AI` counts) · vitals · FPS · DMA · map — readable on AnyDesk / TV.
- **Radial quick menu** (hold-to-open / release-to-confirm), **command palette** (`Ctrl+K`), **toast system**, **first-run tour**, all configurable hotkeys.

**Presets** (Stealth · Loot Run · PvP · Quests · Custom) bundle 13 toggles each and are intentionally distinct — Stealth = silent extract, Loot Run = max info, PvP = hunter mode, Quests = objectives-only. Drift detection auto-demotes to Custom on manual tweaks.

**Loot Filters panel** — full-width toggle rows, integer steppers (auto-repeat), combo rows, four **Quick View** chips (All Loot · Important+ · Wishlist · Quest), live `visible / total` counter.

**Web radar** (`src-silk/Web/wwwroot/`)
- Mobile-first **bottom tab bar** (Players · Loot · Layers · Settings) with slide-up **bottom sheets**, swipe-down to dismiss.
- **FAB radial** mirroring the desktop quick menu — hold-to-open / release-on-slice on touch, tap-then-tap on click.
- **Follow-me default** with **double-tap recenter** on empty map space and pinch-to-zoom.
- **Independent web presets** (Spotter · Battle Buddy · Loot Hunter · Quest Helper · Custom) — separate from the desktop host's preset; each buddy picks their own view. Top-center chip is tap-to-cycle.

**Config**
- `%AppData%\eft-dma-radar-silk\config.json` — debounced JSON persistence.
- IL2CPP offsets resolved at startup and cached to `il2cpp_offsets.json`; hard-coded fallbacks live in `src-silk/SDK/Offsets.cs`.

---

## Project Details

### `lib/VmmSharpEx`

A managed C# wrapper around [MemProcFS](https://github.com/ufrisk/MemProcFS) (`vmm.dll`) and LeechCore (`leechcore.dll`). Provides a high-level `Vmm` handle (read / write / VFS / process enumeration), a `LeechCore` device wrapper, a scatter API for batched gathers / writes, a memory search engine, a refresh manager, strongly-typed flag enums, a Win32 virtual-key DMA input manager, and a `VmmPointer` abstraction with a rich `VmmException` hierarchy.

- TFM: `net10.0-windows`, `Nullable=enable`, doc-file generated.
- Native bin: `lib/VmmSharpEx/native/` (`vmm.dll`, `leechcore.dll`, `leechcore_driver.dll`, `FTD3XX.dll`, `dbghelp.dll`, `symsrv.dll`, `tinylz4.dll`, `vcruntime140.dll`).
- License: **AGPL-3.0** — original MemProcFS API © Ulf Frisk; `VmmSharpEx` modifications © Lone (Lone DMA), 2025.

### `src-silk`

- AssemblyName: `eft-dma-radar` · RootNamespace: `eft_dma_radar.Silk`
- Entry point: [`SilkProgram.Main`](src-silk/Program.cs)
- Packages: `ImGui.NET 1.91.6.1`, `Silk.NET.Windowing/Input/OpenGL/OpenGL.Extensions.ImGui 2.23.0`, `SkiaSharp 3.119.2`, `Svg.Skia 3.0.3`, `Open.Nat.imerzan 2.2.0` (+ `Microsoft.AspNetCore.App` framework reference for the web radar).
- In-tree docs: `src-silk/Docs/DEBUG_OUTPUT_REFERENCE.md`, `src-silk/Docs/MIGRATION_ROADMAP.md`.

---

## License

[BSD Zero Clause License](LICENSE) for `src-silk`. `lib/VmmSharpEx` is **AGPL-3.0** (see the wrapper's own license headers and the upstream MemProcFS license).

---

## Credits

- Built on the work of **lone-dma** and the broader EFT DMA community.
- MemProcFS by **Ulf Frisk** (<https://github.com/ufrisk/MemProcFS>).
- Reference data from [tarkov.dev](https://tarkov.dev/) (see in-app credits).
