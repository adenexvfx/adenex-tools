# UEditingTools_S1 — Changelog

## [1.4.10] — Camera V4 Scope track fix (2026-07-21)

### Fixed
- **Camera V4 Scope track binding** (`uet_main.py`) — `Scope` bool track was added to `comp_binding` (CameraComponent) instead of `cam_binding` (the Camera_v4 blueprint actor itself); now correctly appears directly under the blueprint actor in the sequencer, not nested under CameraComponent.

## [1.4.9] — logging refactor + Camera V4 per-camera (2026-07-21)

### Changed
- **Logging refactor** (`uet_main.py`) — removed `_log` and `_logv` helpers; all logging now goes through `log()` from `adenexlib_core` so every message is captured in the adl buffer and included in server error reports. Verbose-only messages are still gated with `if cfg.extended_log:` inline.
- **Camera V4 spawns for every camera** — `_import_camera_v4` replaced by `_spawn_camera_v4_actor(label, frames, cam_v4_asset)`: spawns one Camera_v4 blueprint per camera (primary afxCam + each entity camera). Asset validation and TimeController are handled once in `_import_cameras`: kit assets are checked upfront, TimeController subsequence is added to the sequence exactly once after all cameras are spawned with range covering the longest camera.

### Fixed
- **Camera V4 not spawned when only entity cameras exist** — previously V4 was only called when `afxCam` frames were present; now it is called for every camera regardless of source.

## [1.4.8] — adenexlib_core exception wrapper + banner logging (2026-07-21)

### Added
- **adenexlib_core integration** (`uet_main.py`) — imports `adenexlib_core as adl` with minimum version check (v19). `_logv()` now routes through `adl.log()` so verbose messages are buffered and included in error reports.
- **Exception wrapper** — `run()` is wrapped in `try/except/finally`. On unhandled exception: if verbose logging was off, all buffered adl logs are dumped to `unreal.log_warning`; `adl.handle_exception()` shows a "send error log to developer?" dialog that uploads logs + traceback to the server; a FAIL notification is shown; the exception is re-raised.
- **Banner logging** at start of `run()`: `adl.set_verbose_logging(cfg.extended_log)`, plugin version (`get_plugin_version_name("UEditingTools_S1")`), PyToolkit version, config dump (verbose only).
- **Success notification** — `show_notification()` with elapsed time on successful import.
- **`finally` block** — always logs `log_execution_times()` and total elapsed time with closing banner.

## [1.4.7] — Camera V4 by kit (2026-07-21)

### Added
- **Add Camera V4 by kit** setting (`AGRImporterSettings`, `SAGRImporterEditorTab.cpp`, `uet_main.py`, default OFF) — when enabled alongside **Import Camera**, spawns `/Game/_kitmvm/Blueprints/Editing/Camera_v4` blueprint actor with the same transform and focal-length keyframes as the primary `afxCam`. Attaches to the root anchor. Adds a `MovieSceneSubTrack` referencing `/Game/_kitmvm/Sequences/24fps_TimeController` spanning the full camera duration. Actor is placed in the **Camera** sequencer folder alongside `afxCam`. Both assets are validated with `does_asset_exist()` before spawning — if either is missing the Camera V4 step is skipped entirely (import continues normally) and a warning dialog is shown after import completes listing the missing asset paths.

## [1.4.6] — CS:GO gloves spawning (2026-07-21)

### Added
- **CS:GO gloves actor spawning** (`uet_main.py`, `SAGRImporterEditorTab.cpp`) — for CS:GO imports, `DT_ItemsDataCSGO` is read from a fixed path (`/Game/CSGO/Assets/data/DT_ItemsDataCSGO`). If missing, a warning dialog is shown. For each player entity, `additional_data.world_gloves` is looked up by row name (supports variant naming: `ctm_sas_varianta` falls back to `ctm_sas`). A duplicate actor is spawned with the same `AnimSequence` (shared skeleton) and the gloves mesh applied. The `skintone` MaterialInstance from `additional_data.skintone` is applied to material slot 1 to match the player's skin colour. The gloves actor gets the `[gloves]` suffix label and is placed in the same sequencer **Players** folder. A `game` field was added to `ImportConfig` and passed from C++ (`"csgo"` when `Game == EUETGame::CSGO`).

## [1.4.5] — Visibility keyframe fix (slot pool ghost entries) (2026-07-20)

### Fixed
- **Visibility track missing hide/show keyframes** (`FAGRReader.cpp`) — When `afxHiddenOffset` released a handle, the AGR kept emitting `entity_state` entries for that same handle with `bVisible=false` ("ghost entries") in the same frame and subsequent frames. Previously, `entity_state` would re-activate the slot from the free pool for these ghost entries and write a spurious `{T, true}` key. The blink filter in Python then saw the `{T, false}` (from `afxHiddenOffset`) and `{T, true}` (from entity_state) at the same timestamp (gap = 0 ≤ 1 frame) and removed both — leaving no visibility keyframe. Additionally, the consumed slot was unavailable when the model genuinely re-appeared, causing a new slot to be created instead of the existing one being reused.
  - **Fix**: `entity_state` now skips slot activation entirely when the handle is not currently active **and** `bVisible=false` (ghost entry guard). Root transform data is still consumed from the binary stream.
  - **Fix**: Visibility keys are now written on `bVisible` state changes (tracked via `LastEntityVis` map per slot) instead of only on new handle activation. This correctly captures hide events that come via `entity_state` `bVisible=false` transitions, and show events via `bVisible=true` re-appearances.
  - **Fix**: `afxHiddenOffset` and `deleted` now only write a hide key if `LastEntityVis[slot] == true` (avoids duplicate keys) and update `LastEntityVis` to `false` so subsequent ghost entity_state entries don't write a spurious show key.
  - **Result**: AWP hide at weapon-switch and AWP/pistol show/hide at re-draw now produce correct keyframes. Tested on 8-9 AGR files.

## [1.4.4] — CS:GO viewmodel bone rotation fix (forearms + gloves) (2026-07-20)

### Fixed
- **CS:GO viewmodel forearm/foretwist bone rotation** (`FAGRAnimBuilder.cpp`) — forearm and foretwist bones on stripped-rig weapon VMs (AWP, Deagle, etc.) were −90° off in Yaw because `bIsCSGOVMTwistedBone` used a direct-parent check (`GetParentIndex == VWeaponUEIdx`) that failed for AWP, where an intermediate Bip01 bone sits between v_weapon and the forearms. Fixed by replacing the parent-index check with `bHasFullBodyBip01Rig`: at entity load time, the UE skeleton is scanned for any direct child of v_weapon whose name ends with `"bip01"` (present in glove VMs as `v_weapon_bip01`, absent in weapon VMs). Stripped-rig entities (`bHasFullBodyBip01Rig = false`) apply `ConvRotAlt` to forearms; full-body rig entities (`true`) apply standard `mirrorY`. Tested on 8–9 AGR files covering AWP, Deagle, and v_glove_fullfinger — all correct.

## [1.4.3] — CS:GO viewmodel angle fix + import progress dialog (2026-07-20)

### Fixed
- **CS:GO viewmodel root bone rotation** (`FAGRAnimBuilder.cpp`) — v_model MDL-root bones (e.g. `v_weapon` in v_snip_awp) were incorrectly receiving the `valveMatrixToBlender` 90°Z correction meant for world entities; this caused arms/sleeves/weapon to point strongly upward. CS:GO viewmodel entities now use the standard mirror-Y conversion (`(-X, Y, -Z, W)`) for their MDL-root bones. Non-CSGO games (CSS, CS16, TF2) are unaffected.

### Added
- **Import progress dialog** (`SAGRImporterEditorTab.cpp`) — `FScopedSlowTask` dialog now appears during import with three labeled phases: "Parsing AGR file…", "Building animation sequences…", "Spawning actors and building sequence…"

## [1.4.2] — Visibility blink filter + global scale setting (2026-07-03)

### Added
- **Filter Visibility Blinks** setting (`AGRImporterSettings`, `uet_main.py`, default ON) — iteratively removes single-frame show/hide blinks caused by rapid slot recycling; each pass removes key[i] where key[i+1] is ≤1 frame later, then re-deduplicates, until stable
- **Global Scale** setting (`AGRImporterSettings`) — float multiplier passed to `FAGRReader::Parse` and applied to all positions and bone transforms; default 1.0 (Source 1 units)

## [1.4.1] — Camera Euler unwrapping + visibility key fix (2026-07-03)

### Fixed
- **Camera yaw/pitch/roll 360° spikes** (`uet_main.py`) — `Quat.rotator()` returns angles in [-180, 180], causing single-frame ±360° jumps; now unwraps each channel by tracking previous value and adjusting by ±360° when delta exceeds 180°
- **Per-frame visibility keyframes** (`FAGRReader.cpp`) — `entity_state/baseentity` was writing `{T, bVisible}` every frame for every active entity; now writes `{T, true}` only on new handle activation (slot reuse or creation), matching Blender's behaviour

## [1.4.0] — Weapon bone mapping + camera rotation fix (2026-07-03)

### Fixed
- **afxHiddenOffset pre-processing** (`FAGRReader.cpp`) — handles now released into pool BEFORE entity_state entries of the same frame, matching Blender behaviour; eliminated duplicate actors for same-model entities (e.g. v_deagle afx.6/7 → one actor)
- **SMD root bone detection** (`FAGRAnimBuilder.cpp`) — `valveMatrixToBlender` (90°Z) now applied to ALL SMD root bones (parent index == 0 in UE skeleton), not just AGR bone 0; fixes v_deagle Bone01 (left arm root at AGR index 18) having wrong rotation
- **Attachment bone skipping for weapon models** (`FAGRAnimBuilder.cpp`) — Crowbar inserts virtual `attachment_N` bones into the FBX/UE skeleton that don't exist in MDL/AGR; linear AGR→UE bone mapping was shifted by these, causing Bone02 (weapon body) to receive Bone03 animation data etc. Fixed by building `MDLToUE` table that skips `attachment_*` bones
- **Camera rotation channels** (`uet_main.py`) — roll and pitch were swapped due to Blender's `blenderCamUpQuat` (90°X post-multiply); corrected to `roll=-rotator.pitch`, `pitch=-rotator.roll`
- **Sequencer auto-open** (`uet_main.py`) — created LevelSequence now opens automatically in Sequencer on import completion

## [1.3.0] — Blender-style entity slot pooling (2026-07-03)

### Fixed
- **Entity proliferation from handle recycling** — AGR format creates a new handle each time an entity appears (draw/holster weapon, player respawn, etc.), previously resulting in thousands of separate AnimSequence assets and hundreds of animation sections per sequencer track
- **FAGRReader** now implements Blender-style slot pool: when a handle is released (hidden/deleted), its slot is returned to a free pool keyed by model name; the next handle with the same model reuses that slot, appending keyframes at their global times into the same AnimSequence
- `1.agr` (previously 3536 entities) now produces ~21 entities matching Blender output
- Each entity gets exactly **one AnimSequence** and **one sequencer section** (was: 200+ sections per track)
- Visibility track on merged slot correctly reflects all hide/show events across all handle appearances

### Changed
- `FAGREntity.Handle` now stores slot ObjNr (1-based stable ID) instead of raw AGR handle number
- `uet_main.py` entity import simplified: removed slot-grouping logic (now done at parse time in C++)
- Shortest-path quaternion continuity keyed by entity slot index (persists across handle reuse)

## [1.2.0] — CS1.6 bone transforms + sequencer timing fix (2026-07-02)

### Fixed
- **CS1.6 player bone rotations** — correct coordinate conversion for all bones:
  - Bone 0 (Bip01): applies Blender `valveMatrixToBlender` (90°Z) + empirically verified axis mapping
  - Non-root bones: negate X and Z quaternion components to fix Yaw/Roll sign flip
- **CS1.6 bone positions** — separate conversion per bone type:
  - Bone 0: `(-Y, -X, Z)` swap (Source1→UE world space)
  - Non-root bones: `(X, -Y, Z)` (local bone space, Y negated)
- **Sequencer animation placement** — each entity now placed at its actual start time in the sequence (`start_t` from AGR), fixing misalignment between players and weapons

### Changed
- `asset_map.json` now includes `"start_t"` (seconds) per entity — first keyframe time from AGR

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
