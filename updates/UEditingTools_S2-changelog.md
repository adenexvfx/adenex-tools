# UEditingTools_S2 Changelog

## v1.3.0 — 2026-07-14

### Player Camera Matching — Dead Player Fix (10/10)

**Problem:** With the `nuke_staehr_USP_2k_usp_30fps.dmx` file, only 3/10 player models were matched to player names. 7 player models belonged to dead players whose DmeChannelsClip contained no `rootTransform_p` channel — only material-override and jiggle channels.

**Fix 1 — Stage 2 sampling (`uet_s2_init.py`):**
- `_match_pos_samples_to_camera` previously used only 21 evenly-spaced `match_samples` from the camera timeline instead of all camera samples (`all_samples`)
- Changed `coords.get("match_samples", [])` → `coords.get("all_samples", [])` in the comparison loop
- This fixed the ctm_sas (Staehr) match: score went from 1/21 → 20/21

**Fix 2 — Stage 4: Dead player static-position matching (`uet_s2_init.py`):**
- Added Stage 4 after the `rootTransform_p` block
- Dead player models have their static death position stored in `FDMXModel.DefaultPos` (from DmeTransform.position in the DMX file), exposed as `m['default_pos']` in Python
- Stage 4 calls `_match_nearest_camera_sample(dp[0], dp[1], dp[2], camera_data, tolerance=5.0)` to match the static position against all camera CSV samples
- All 7 dead players matched with dist=0.00 to 0.01 (exact)

**Result:** 10/10 player models matched for `nuke_staehr_USP_2k_usp_30fps.dmx`

---

## v1.2.0 — (prior)

- Binary DMX v9 parser (FDMXParser.cpp): GuidStr fix for stable transform ID linking
- FDMXModel: added `DefaultPos`, `FolderName`, `ChildTransformIds` fields
- GetModelList binary serialization includes `folder_name` and child IDs
- Initial camera matching pipeline (Stages 1–3) in `uet_s2_init.py`
