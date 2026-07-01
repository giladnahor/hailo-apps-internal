# Release Validation Findings

**Validated on two devices:**
- **Hailo-8** — HailoRT 4.24.0 + TAPPAS 5.3.1 (Model Zoo v2.19.0)
- **Hailo-10H** — HailoRT 5.3.0 + TAPPAS 5.3.0 (Model Zoo v5.3.0), + gen-ai extras & HEFs
- x86_64 Ubuntu 24.04, kernel 6.17.0-35, Python 3.12
**Date:** 2026-07-01

## Fix status (branch `fix/release-validation-findings`)

| # | Issue | Status |
|---|---|---|
| 1 | C++ compat map missing 4.24.0 (auto-download broken) | ✅ fixed + validated on H8 (added 5.3.0/5.4.0 for H10 too) |
| 2 | instance_seg default model incompatible | ✅ fixed for hailo8 **and hailo10h**, both validated on-device; **hailo8l NOT changed — no device to test** |
| 3 | `--list-models` wrong usage text | ✅ fixed + validated |
| 4 | rhythm_royale undeclared `soundfile` | ✅ fixed (requirements.txt + importorskip) + validated (37/37, both devices) |
| 5 | C++ test harness leaked OpenCV Qt env | ✅ fixed + validated (14→0 fails) |
| 6 | C++ image test never flagged nonzero exit | ✅ fixed + validated (now correctly catches the instance_seg error) |
| 7 | `2>nul` created junk files on Linux | ✅ fixed + validated (no `nul` file) |
| 8 | C++ s3 URL used `h10h` instead of `h10` (403 on H10 s3 models) | ✅ fixed + validated on H10 (auto-download now 200) |

> **#2 remaining work for hailo8l only (no H8L device available):** default `yolov5n_seg`
> (4 outputs) is unsupported by the standalone decoder. Apply the same config pattern: a
> standalone-compatible model FIRST tagged `app_type: [standalone]`, yolov5 raw model retagged
> `[pipeline]`. h8l has no `_with_nms` variant in config — use a YOLOv8-seg model (10 outputs,
> e.g. `yolov8s_seg`). Validate on an H8L device.

> **Platform note (HailoRT, not this repo):** the HailoRT **5.3.0** PCIe driver fails to build on
> kernel **6.17** — `vdma/monitor.c` calls `del_timer_sync()`, removed in Linux 6.16. A local
> version-guarded compat shim (`del_timer_sync`→`timer_delete_sync`) in `/usr/src/...` was applied
> to proceed. (The H8 4.24.0 driver built fine on 6.17.) Report upstream to the HailoRT team.

## H10 test results (HailoRT 5.3.0)

| Suite | Result |
|---|---|
| Sanity | 39/39 pass |
| Installation | 31/31 pass |
| Pipeline (hailo10h, on-device) | 74/74 pass |
| Standalone (on-device) | 21/21 pass |
| GenAI (LLM/VLM/Whisper/voice/agent) | 16/16 pass + 1 intentional skip |
| Community | 33/33 app suites pass |
| C++ | 35 pass / 3 skip / 0 fail (after #2-h10 + #8 fixes) |

> GenAI note: the agent example needs the `agent` group model (`Qwen2.5-Coder-1.5B`) pre-downloaded;
> otherwise its 120s test timeout fires mid-download (not a code failure).

## Test results summary

| Suite | Result |
|---|---|
| Sanity | 39/39 pass |
| Installation | 31/31 pass |
| Pipeline (on-device) | 148/148 pass |
| Standalone (on-device) | 41/41 pass |
| Community | 32/33 pass (1 = missing dep, see #4) |
| GenAI | N/A on H8 (`skipif(not hailo10h)`) — validate on H10 |
| C++ | builds 10/10 pass; runtime blocked by #1/#2 |

Pipeline + standalone + most community suites are clean. All issues below are in the
C++ apps and test/packaging gaps — no defects found in the Python pipeline/standalone code.

---

## #1 — C++ model auto-download broken on HailoRT 4.24.0 (HIGH — affects all C++ apps)

**File:** `hailo_apps/cpp/common/resources_manager.cpp:498-503`

The hardcoded HailoRT→ModelZoo compat map for hailo8/8l stops at 4.23.0:

```cpp
static const std::unordered_map<std::string,std::string> compat_8 = {
    {"4.23.0","v2.18.0"}, {"4.22.0","v2.16.0"},
    {"4.21.0","v2.15.0"}, {"4.20.0","v2.14.0"},
};   // missing 4.24.0
```

On 4.24.0 the lookup misses and throws (`:516`):
```
ResourcesManager ERROR: HailoRT 4.24.0 is not compatible with hailo8. Use HailoRT 4.x.x.
```
(Misleading — 4.24.0 IS 4.x.x.) `config.yaml` already maps `4.24.0 -> v2.19.0`; this C++
table wasn't updated for the release.

**Impact:** any C++ app that needs to auto-download a (non-local) HEF fails. In this run the
other C++ apps only passed because their default HEFs were already on disk from the Python
test-resource download step.

**Fix:** add `{"4.24.0","v2.19.0"}` to `compat_8`. Better: derive the mapping from
`config.yaml` (`model_zoo_mapping` / version comments) so future releases don't regress.

---

## #2 — C++ instance_segmentation default model is unsupported by its own decoder (HIGH)

**Files:** `hailo_apps/cpp/instance_segmentation/instance_segmentation.cpp:35-49`,
model config advertising `yolov5m_seg` as default.

The decoder supports only:
- 1 output  → NMS-fused (`*_with_nms`)
- 10 outputs → YOLOv8-seg

But the **default model `yolov5m_seg` has 4 outputs**, so the app aborts at init for
image/video/camera:
```
Detected instance segmentation model with 4 outputs.
ERROR: Unsupported instance segmentation HEF. To see the supported models, run: --list-models
```
(The Python/GStreamer pipeline handles yolov5m_seg via its `.so` postprocess — only the
standalone C++ decoder lacks the 4-output path.)

**Verified working:** `--net yolov8m_seg` (10 outputs) runs full inference.
`--net yolov5m_seg_with_nms` (1 output, the apparent intended default) would work but its HEF
isn't local and the download is blocked by #1.

**Fix:** set the default to a supported model (`yolov5m_seg_with_nms` or `yolov8m_seg`), or
add a 4-output yolov5-seg decode path if that model is meant to be the default.

---

## #3 — C++ `--list-models` prints the wrong flag (LOW)

**File:** `hailo_apps/cpp/common/resources_manager.cpp:970`

Prints `Usage: --hef-path <model_name>`, but the parser actually reads `--net` / `-n`
(`hailo_apps/cpp/common/toolbox.cpp:414`). `--hef-path` and `--help` are silently ignored.

**Fix:** correct the usage text to `--net`/`-n` (and/or accept `--hef-path` as an alias).

---

## #4 — community `rhythm_royale` undeclared dependency (MEDIUM)

**Dir:** `community/apps/pipeline_apps/rhythm_royale/`

Tests `import soundfile`; the app uses `sounddevice` at runtime. Neither is declared in any
requirements file, `app.yaml`, or `run.sh`, so the suite errors at collection:
```
ModuleNotFoundError: No module named 'soundfile'
```
**Verified:** with `soundfile` installed the app passes 37/37.

**Fix:** declare `soundfile` (and `sounddevice`) as the app's deps.

---

## #5 — C++ test harness leaked OpenCV's Qt env into native subprocesses (FIXED in working tree)

**File:** `tests/test_cpp_runner.py` (edited — see git diff)

`import cv2` (via conftest) exports `QT_QPA_PLATFORM_PLUGIN_PATH` /`QT_QPA_FONTDIR` pointing at
the opencv-python wheel's Qt plugins. The native C++ apps inherited these and aborted with
`Could not load the Qt platform plugin "xcb"`. Added `_child_env()` to strip those vars before
launching the C++ subprocess (in `_run_timed` and the image `subprocess.run`). This took the
C++ suite from 14 failures → 2 (the 2 remaining are the real bug #2).

> NOTE: this edit is uncommitted in the working tree on the test machine. Re-apply or cherry-pick.

---

## #6 — C++ image test never flags nonzero exit (LOW, test quality)

**File:** `tests/test_cpp_runner.py`, `test_cpp_image` → `_Result(...)` built without
`early_exit`, so `_assert_clean`'s `bad_exit = early_exit and rc != 0` is always False for
image tests. `instance_segmentation` image "passed" even though the app exited 8 with the
"Unsupported HEF" error. Video/camera caught it only because they pass `early_exit=True`.

**Fix:** have the image test treat any nonzero return code as failure.

---

## #7 — stray `nul` files created during runs (TRIVIAL)

`hailo_apps/cpp/nul` and `./nul` appear after C++ runs — likely a `> nul` redirect (Windows-ism)
that creates a real file on Linux. Worth grepping the C++/scripts for `nul`.

---

## Environment prerequisites discovered (for docs / CI)

- C++ apps need `cmake` (>=3.20) and `libssl-dev` (for the curl submodule); OpenCV dev + g++
  were already present. Not installed by `install.sh`.
- C++ build requires submodules: `git submodule update --init --recursive` (yaml-cpp, curl).
  Without them the C++ build tests silently skip.
- C++ GUI (video/camera) tests need a display; `run_tests.sh --cpp` does not export `DISPLAY`.
  Ran with `DISPLAY=:1 XAUTHORITY=~/.Xauthority`.
- GenAI deps (`pip install -e ".[gen-ai]"`) not installed; GenAI HEFs are H10-only anyway.
