# Chatterbox-TTS-amd — Project Notes

**Repo:** phattbeats/Chatterbox-TTS-amd · **Upstream:** devnen/Chatterbox-TTS-Server
**What it is:** an AMD Windows fork — native ROCm for the Radeon RX 6000 series (gfx1030/1031/1032) on Windows 11, which upstream says it doesn't support.
**Built:** April–June 2026, across 22 tracked issues (PHA-257 → PHA-720).

---

## The short version

Upstream's stance is "ROCm is Linux-only." This fork disproves that for one specific
slice of hardware — RX 6000 series on Windows — by skipping AMD's official 5+ GB ROCm
installer and using **guinmoon's pre-built ROCm-SDK wheels** (compiled with AMD's
TheRock build system) instead. The actual source change is small. Making it install
itself reliably on a stranger's Windows box was the hard part, and most of the project's
hours went there, not into the patches.

### The four code changes

| File | Change | Why |
|---|---|---|
| `engine.py` | Disable cudnn/MIOpen on gfx103X + Windows before model load | MIOpen kernels crash with HIP `0xC0000005` on RX 6000 / Windows |
| `utils.py` | `soundfile` fallback around `torchaudio.save()` | torchaudio 2.9+ needs FFmpeg/torchcodec DLLs absent on stock Windows Python |
| `start.py` | `--rocm-windows` install path | GPU detection, wheel download, local install, experimental banner |
| `requirements-rocm-windows.txt` | ROCm wheel reference manifest | documents the wheel set (install is driven by `start.py`) |

### The wheel set (Python 3.12 / win_amd64 only)

`rocm`, `rocm-sdk-core`, `rocm-sdk-devel`, `rocm-sdk-libraries-gfx103x-all` (all `7.1.1`),
plus `torch 2.9.1`, `torchaudio 2.9.0`, `torchvision 0.24.0` (all `+rocmsdk20251207`).
These exist nowhere on PyPI or the PyTorch index — they are served as assets on the
**GitHub release `v0.1.0-rocm-wheels`** and pulled at install time. They are **cp312-only**;
Python 3.10 cannot use them (this caused a multi-day detour — see Phase 4).

---

## How we actually wrote it

The fork was built by several agents over two months — Mr. House / Jenkins and Van Dam
did the engineering, with Brandon driving every test on the real RX 6750 XT machine and a
final cleanup pass at the end. The plan was clean. Reality was a grind of Windows
packaging edge cases discovered one console paste at a time.

### Phase 0 — Setup (PHA-257)

Brandon had a working `EXECUTION_LOG` from getting Chatterbox running by hand on his
Windows 11 / RX 6750 XT (gfx1031) box — ~10 hours of trial and error, ending with bare
chatterbox generating at **~0.88x RTF**. PHA-257 turned `plan.md` + that log into 16
sub-issues across four tracks: engineering (T-3xx), wheel infra (T-2xx), docs (T-4xx),
QA (T-1xx). Brandon's standing note: *"you have github_token — almost no local work,
fork and push first."* That set the tone: do it in the repo, not in a sandbox.

### Phase 1 — Fork + wheel hosting (PHA-259, 266–269)

- **PHA-259 (T-301):** Forked to `phattbeats/Chatterbox-TTS-Server`, branch
  `feature/rocm-windows-support`, `upstream` remote → devnen.
- **PHA-266 (T-201) — cancelled.** The original plan was an nginx PEP-503 index at
  `wheels.phatt.vip`. Brandon killed it: *"no need for hosting on an IP, it will only
  call locally."* The whole hosted-index track was abandoned.
- **PHA-267 (T-202):** The wheels are ~2.1 GB. Brandon asked "can't we just link them in
  the repo?" — no (10 GB repo cap, LFS limits, every clone pays). Landing spot:
  **GitHub release `v0.1.0-rocm-wheels`**, assets pulled on demand. This pivot — index →
  GitHub release — is the single most important architecture decision in the project,
  and the source of nearly every stale `wheels.phatt.vip` reference cleaned up later.
- **PHA-268 (T-203):** MIT LICENSE + NOTICE crediting AMD TheRock / guinmoon / devnen.
- **PHA-269 (T-204):** `smoke_test.py` written; real execution can only happen on
  Brandon's hardware.

### Phase 2 — The four patches (PHA-260, 261, 262, 263, 264)

- **PHA-260 (T-302):** Perth watermarker fallback — already present upstream in
  `start.py`'s `_patch_chatterbox_watermarker`, idempotent. Inherited, no new PR needed.
- **PHA-261 (T-303):** the cudnn disable in `engine.py` — the critical stability fix.
- **PHA-262 (T-304):** the `soundfile` fallback in `utils.py`.
- **PHA-263 (T-305):** `requirements-rocm-windows.txt`.
- **PHA-264 (T-306):** the `--rocm-windows` flag, pre-flight GPU check, experimental banner.

### Phase 3 — Docs (PHA-270, 271, 272, 273, 274)

README header (PHA-270), a fork doc (PHA-271), a longform `docs/ROCM_WINDOWS.md`
(PHA-272), and a softened upstream "don't use ROCm on Windows" warning that adds an
RX-6000 escape hatch (PHA-273, also opened as upstream PR devnen#137). **PHA-274
(T-103)** is the honest one: the validation report was written as a *framework* with the
soak rows left as TODO, because the 7-day soak (T-102) never formally ran. No fabricated
numbers. (See "Performance, honestly" below.) Several of these docs were later collapsed
into the README in PHA-277.

### Phase 4 — Release + the debugging marathon (PHA-275, 276, 283)

**PHA-275** shipped `v1.0.0` — goal: download a zip, double-click `start.bat`, done. The
release tag and `main` got tangled (a push clobbered the upstream tag; the ROCm work was
on the feature branch, not `main`) before being reconciled so `main` and `v1.0.0` carried
the full fork.

**PHA-276** is the real story — 119 comments of Brandon pasting console output and agents
fixing one wall at a time:

- `torch==2.5.1` from `download.pytorch.org/whl/rocm6.1` **doesn't exist** → install from
  the GitHub-release wheels on local disk instead of any pip index.
- **`rocm==7.1.1` is a phantom dependency.** guinmoon's torch wheel lists `rocm==7.1.1`
  in its metadata, but AMD never published a `rocm` package — pip finds an unrelated
  `rocm==0.1.0` and dies. Fix: generate a local **stub `rocm` wheel** that satisfies the
  pin and does nothing (`_create_rocm_stub_wheel`). Brandon pushed for this over a
  constraints file, correctly — constraints can't resolve a package that isn't there.
- **Python version mismatch.** Wheels are cp312; the portable launcher shipped Python
  3.10.11. `not a supported wheel on this platform`. No cp310 ROCm wheels exist anywhere
  → switch the embeddable Python to **3.12.8** and add a version guard. (Stale 3.10
  folders had to be manually deleted to take effect.)
- **`--no-index` was too aggressive** — it blocked PyPI for genuine deps like `filelock`.
  Switched to `--find-links` so local wheels win but PyPI is still reachable.
- **Embedded vs system Python.** `server.py` spawned via PATH picked up system Python 3.10
  and crashed on `yaml`/`librosa`. Fix: launch `server.py` with the explicit embedded
  interpreter path.
- **Missing server deps** the ROCm path skipped — `librosa`, `PyYAML`, `uvicorn`,
  `python-multipart` — resolved by installing the full `requirements.txt` after the wheels.
- **WinError 1314** (HuggingFace symlink in cache) → set `HF_HUB_ENABLE_SYMLINKS=0`, no
  Developer Mode needed.
- **GPU detection was Linux-only** (`rocm-smi`). On Windows the RX 6750 XT read as "Not
  detected," forcing the wrong menu choice. Rewritten to Windows-native detection:
  `vulkaninfo` (JSON, then plain text) → registry scan → WMIC, picking up AMD vendor ID
  `0x1002`.

**PHA-283** was Paperclip's auto-recovery wrapper for PHA-276 when runs kept timing out.
Its recorded resolution is the honest endpoint: the server **runs** (loads
ChatterboxTurboTTS and serves TTS), the remaining GPU-detection gap was a host-side
ROCm-driver matter, and Brandon worked it out on his machine on **April 30**. By PHA-720
he reports it *"working perfectly."*

### Phase 5 — Consolidation (PHA-277)

After v1.0.0, the added `.md` files overlapped ~90%. Removed `ROCM_WINDOWS_INSTALL.md` and
`docs/ROCM_WINDOWS.md`, folded the essentials into the README quick-start, and purged the
contradictory `wheels.phatt.vip` references Brandon kept hitting.

### Phase 6 — Final cleanup (PHA-720, this issue)

Repo renamed `Chatterbox-TTS-Server` → **`Chatterbox-TTS-amd`**. Fork commits re-authored
to `phattbeats <obiwouldjablowme@protonmail.com>`. "PHATT TECH" branding stripped from
docs, the `v1.0.0` release title/body, and the leftover spots in code — the runtime
`--rocm-windows` banner, `engine.py`'s patch comment, and a dead `phatt-tech` repo URL
now read "AMD Windows fork" and point at `phattbeats/Chatterbox-TTS-amd`. This document
was rewritten from a full read of all 22 issues. (Upstream's 144 inherited commits keep
their original authors — that's the fork relationship, not our work to rewrite.)

---

## Technical notes

**Why guinmoon's wheels.** AMD's official Windows ROCm is a 5+ GB SDK install. guinmoon's
TheRock-based rebuild is pip-installable and pulls only the runtime PyTorch needs (~2.1 GB).
Trade-off: unofficial, tested only on gfx103X (RX 6000). gfx1100 (RX 7000) is untested.

**Why Python 3.12.** The wheels are cp312-only and no cp310 ROCm build exists publicly, so
the `--rocm-windows` path mandates 3.12 — even though upstream prefers 3.10 for ONNX wheels.

**The cudnn disable** (`torch.backends.cudnn.enabled = False`) is blunt but cheap: gfx103X's
native HIP ops are stable while cudnn's Windows port crashes. Negligible inference cost.

## Performance, honestly

- Brandon's `EXECUTION_LOG` measured **bare chatterbox at ~0.88x RTF** (5-chunk loops) on
  the RX 6750 XT — memory stable, known-good baseline.
- The README's headline claim is **"~2.3x over CPU"** — deliberately *not* worded as
  "real-time," since RTF on short turns is sub-real-time.
- The formal 7-day soak (T-102) that would have produced a clean validated table was never
  run, so `docs/validation-report.md` stayed a framework. We didn't invent numbers to fill it.

## Credits

devnen (upstream server) · Resemble AI (Chatterbox model) · guinmoon (Windows ROCm wheels)
· AMD TheRock (the build system behind them).
