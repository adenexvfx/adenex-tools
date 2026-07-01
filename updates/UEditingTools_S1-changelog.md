# UEditingTools_S1 — Changelog

## [1.1.0] — UEditingToolsS1Editor UI panel (2026-07-01)

### Added
- **UEditingToolsS1Editor C++ module** — dockable Slate panel in the UE Editor
  - 3-tab layout: AGR / Settings / Misc
  - AGR tab: `.agr` file picker, output content path, sequence name
  - Settings tab: game selector (CS:GO / CS 1.6 / CSS / TF2), import toggles (Players / Weapons / POV / Camera), bounds scale, receives decals, save level, extended log, developer mode
  - TF2 mode: TF2 Assets Folder field (hidden for non-TF2 games), one-time advanced pipeline warning dialog
  - Misc tab: Coming Soon placeholder
  - Import button triggers full pipeline: validate → DataTable/TF2 skeleton lookup → FAGRReader::Parse → FAGRAnimBuilder::Build → Python uet_main.run()
- **DataTable skeleton matching** — CS:GO/CS16/CSS: resolves model names to USkeleton assets via per-game DataTable (`/Game/CSGO/Assets/data/DT_ItemsDataCSGO`, etc.)
- **TF2 folder scan** — recursively scans the configured UE content folder for USkeleton assets
- **PyToolkit toolbar button** — "UET S1" button added to PyToolkit toolbar, opens the panel

## [1.0.0] — AGRCore module (2026-07-01)

### Added
- New plugin **UEditingTools_S1** — Source 1 AGR importer for Unreal Engine 5.8
- **AGRCore C++ module** — reads `.agr` binary files directly in UE without Blender
  - Supports AGR format versions 5 and 6 (CS 1.6, CS:GO, CSS, TF2)
  - Automatically detects FPS from frame time samples in the AGR file
  - Creates `UAnimSequence` assets per entity with correct bone tracks
  - Source 1 → UE coordinate conversion (position and rotation)
  - Shortest-path quaternion interpolation (no animation flipping)
- **Temp JSON output** for Python sequencer pipeline:
  - `visibility.json` — per-entity visibility keyframes in seconds
  - `camera.json` — camera position/rotation/FOV in UE space
  - `asset_map.json` — entity → UE asset path mapping + detected FPS
- Temp files written to `ProjectIntermediateDir/UEditingTools_S1/` and cleaned up after use

### Technical
- AGRCore module: `FAGRTypes`, `FAGRReader`, `FAGRAnimBuilder`
- O(1) entity lookup via handle map (no performance degradation on large AGR files)
- Modular plugin, depends on PyToolkit
