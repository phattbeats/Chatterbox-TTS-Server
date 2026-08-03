# Chatterbox TTS on AMD ROCm — The Complete Story

*A narrative reconstruction of the full conversation, spanning April 19 – August 2026, told from the chat's own record rather than the Paperclip board. Where the two sources disagree, that's noted explicitly.*

---

## Part 1 — The Ask

**Sunday, April 19, 2026, 7:12 AM EDT.** Coffee, clear mind, opening message, verbatim:

> "let's do a deep research dive this morning with some fresh coffee and a clear mind.
>
> trying to solve my age old issue: finding ANY TTS provider (alltalk,chatterbox etc.) that will work with my desktop stack: windows 11 + AMD Radeon RX 6750 XT
>
> i've tried running WSL - it seems to be not a solved issue. (hopefully thats fixed now, or someone has found a workaround)
>
> my podcast-gen is going great, except for the fact that it takes an hour to do a 20 min podcast because its in cpu mode. i use chatterbox-tts-serer-1.0.0, it has good stability, custom voices.
>
> i treid alltalk in the past but it seems to be abandoned.
>
> use as many tool calls and searches as you need."

Plain statement of the problem: a working podcast pipeline, bottlenecked by a TTS engine running on CPU because the official ecosystem said the hardware wasn't supported. `chatterbox-tts-server-1.0.0`'s own README, dated December, was explicit: **"ROCm not working on Windows: ROCm only supports Linux — use CPU mode on Windows with AMD GPUs."** You asked whether that was outdated. It was, but nobody had proven it yet.

---

## Part 2 — The Research That Changed the Picture

The first pass through search results confirmed everything discouraging you already knew. AMD's ROCm 7.2 official support matrix covered RDNA3 and RDNA4 on Windows — nothing on your card. The RX 6750 XT is gfx1031 (Navi 22, RDNA2), and AMD's own `TheRock` GitHub issue #1719 confirmed the standard `HSA_OVERRIDE_GFX_VERSION` workaround (spoofing gfx1031 as gfx1030) **does not work on native Windows**, only on Linux and WSL2 — and WSL2 for your generation of card had its own open bugs (PCIe atomics failures, an ROCm 6.4.3 regression breaking the override entirely). Your WSL instinct not to trust it was correct.

Then a different thread surfaced. AMD had quietly stood up **TheRock**, an open build system generating per-architecture ROCm + PyTorch wheels, publishing nightlies at `rocm.nightlies.amd.com/v2-staging/` under a `gfx103X-dgpu` index — covering gfx1030, gfx1031, and gfx1032 together. A community member, **guinmoon**, was mirroring and repackaging these Windows-native builds at `github.com/guinmoon/rocm7_builds`, with an explicit line: *"gfx1031: AMD RX 6700 / XT."* Another user, **o0LINNY0o**, had a documented working local AI stack — llama.cpp, Whisper, Kokoro TTS — on the exact same card, using exactly this path. Proof of life.

The plan: TheRock/guinmoon wheels for PyTorch, `devnen/Chatterbox-TTS-Server` installed with `--no-deps` to stop pip's resolver from silently downgrading torch back to CPU, and however many patches it took after that.

---

## Part 3 — Install Hell

The first PowerShell setup script failed on execution policy, then on a locked directory. Once past that:

**The typo.** The wheel folder lived at `C:\Users\PHATT\Downloads\guimoon-rocm` — misspelled, missing the second `n` — which cost real time before the path was found.

**`hipErrorInvalidImage`.** The first wheel set grabbed wasn't built specifically enough for gfx103X; guinmoon's local `gfx103x_all` variant fixed it. Roughly 10–20GB of downloads and installs later, `torch.cuda.is_available()` returned `True` with `Device: AMD Radeon RX 6750 XT`. Progress — briefly.

**The crash that took down the display driver.** Running the model for the first time crashed the GPU driver hard enough to require a manual restart:

```
Display driver amduw23g-198974-eda2a421 stopped responding and has successfully recovered.
```

Root cause: MIOpen's convolution kernels were JIT-compiling against a tuning database with no entries for gfx1031, and the resulting garbage kernel launches were segfaulting the driver. The fix — later formalized as the fork's single most important patch — was one line at the top of `engine.py`:

```python
torch.backends.cudnn.enabled = False
```

On ROCm, disabling cudnn also disables MIOpen, since MIOpen impersonates cudnn at the PyTorch API layer. Native HIP ops on gfx103X are stable; MIOpen's Windows port is not.

**The watermarker crash.** Next failure: `TypeError: 'NoneType' object is not callable` on `perth.PerthImplicitWatermarker()`. Resemble AI's `perth` library ships the watermarker as `None` when its native binary isn't present, but exports a working `DummyWatermarker` as a fallback. Two-line patch in `chatterbox/tts.py`.

You said, mid-crisis: **"you would be instructed to panic :D lmao."** Correctly diagnosed the mood.

---

## Part 4 — First Light

Late morning, with both patches applied, a clean test run:

```
Sampling:  81%|########  | 809/1000 [00:23<00:05, 33.12it/s]
Text length:    682 chars
Audio length:   32.4s
Generation:     33.4s
RTF:            1.03x (1.0x realtime)
Podcast est:    20.6 min to generate 20 min of audio
```

Real-time factor 1.03. A 20-minute podcast, generated in roughly 20 minutes — down from the CPU baseline's one hour. Your response, characteristically dry: **"no glitches, that is a stable output."**

A 5-chunk loop test followed, averaging RTF 0.88x. Then, escalating: **"WE MUST PRESS ON LADS. VICTORY IS NEAR."**

---

## Part 5 — Production Validation

Late April 19, we cloned `devnen/Chatterbox-TTS-Server` proper and built it into a real server rather than a bare test script. A third patch emerged here: `torchaudio.save()` was failing on Windows because torchaudio 2.9+ wants torchcodec, which wants FFmpeg DLLs absent from a standard Python install. Fallback to `soundfile.write()` directly, in `utils.py`.

You wrote a fictional three-host podcast script — DEV, MARA, and PRIYA arguing about the week's tech news — as a real-world stress test: 140 dialogue turns, three distinct cloned voices (Thomas, Emily, Olivia), 15,832 characters. Full run through the production server with chunking and voice cloning:

```
Generated: 140 chunks
Gen time:  1289.1s (21.5 min)
Audio length: 994.7s (16.6 min)
```

**RTF 1.30x.** Multi-voice, chunked, cloned — a real approximation of the daily podcast pipeline, running end to end on hardware the official documentation said couldn't do it. This is the moment you said: **"this is my daily podcast-gen pretty much."**

---

## Part 6 — Standing Up the Fork

That night, the fork plan got written for handoff to your Paperclip agent org — Mr. House, Van Dam, Jenkins, and others — as a structured issue tree (`plan.md`, later PHA-257 and descendants). Four tracks: infrastructure/wheels, code patches, documentation, QA. `phattbeats/Chatterbox-TTS-Server` was forked from `devnen/Chatterbox-TTS-Server`, with an `upstream` remote kept live for future pulls.

*(The detailed issue-by-issue engineering from here — PHA-259 through PHA-720 — happened on the Paperclip board, not in this chat directly, and is documented exhaustively in `PROJECT_NOTES.md`, which lives in the repo. That thread runs in parallel with what follows below, and the two accounts overlap on April 30.)*

---

## Part 7 — v1.0.0 and the Four-Layer Bug (in this chat)

By April 30, `v1.0.0` was live on GitHub. A downstream user (via SillyTavern, at `C:\SillyTavern\Chatterbox-TTS-Server-1.0.0-amd\`) installed it and it silently ran on CPU — no error, just slow. Debugging this directly in chat surfaced a chain of four nested failures:

1. **`config.py`** wrote `device: cpu` to `config.yaml` whenever auto-detection failed, with no visible warning.
2. **`start.py`** called `install_requirements(venv_pip, "requirements.txt", root_dir)` *after* the ROCm wheels were already installed. The default `requirements.txt` pointed at PyTorch's CPU-only package index, silently downgrading the working ROCm torch build to `torch-2.5.1+cpu`.
3. **A fake wheel.** `_create_rocm_stub_wheel()` was generating a stub `rocm-7.1.1` package that satisfied torch's metadata dependency pin but contained none of the actual `rocm_sdk` Python module.
4. Fixing 1–3 exposed the real problem: `ModuleNotFoundError: No module named 'rocm_sdk'`, because the *real* `rocm-7.1.1.tar.gz` — the one that actually provides that module — only existed on guinmoon's Google Drive, not anywhere the installer could reach automatically.

We diagnosed all four live, wrote a recovery procedure, and verified the fix: `torch: 2.9.1+rocmsdk20251207`, `cuda: True`, `device: AMD Radeon RX 6750 XT` — confirmed on the affected machine.

---

## Part 8 — The Performance Mystery

While fixing the v1.0.0 install bug, you re-ran the original April 19 benchmark on your *own* machine as a sanity check. Same venv, same script, same wheels.

**RTF 6.69x.** Sampling at 5.79–7.27 it/s, against an original of 33 it/s. A five-to-six-times regression on a setup that should have been byte-identical.

We checked everything measurable: AMD driver dated weeks before the working build, DLLs unchanged, JIT cache identical hot and cold, chatterbox force-reinstalled. Nothing explained it. I spent roughly two hours offering theories about why this might just be the real ceiling and the April 19 numbers a fluke, while you kept saying, correctly: **"we had multiple tests that ran great. super fast."** I was wrong and you were right, repeatedly, in real time.

By early afternoon I told you to stop — pet Moses, eat something, ship on CPU tomorrow if you had to. Ten minutes later:

> "there was a windows update that required a restart this week."

That was it. Windows had staged a kernel-mode driver update and was waiting on a reboot the whole time. The on-disk DLL was the new version; the driver actually loaded in memory was the old one. `WMI`'s `DriverDate` reports what's on disk, not what's running — invisible to every diagnostic we'd run. You rebooted. The next message, verbatim:

> "CLAUDE..... IT JUST WORKED AFTER A RESTART BRO."

Post-reboot: Original Chatterbox sustained 36–37 it/s — *better* than the April 19 baseline of 33. Turbo hit 66–70 it/s, RTF roughly 0.5x. The fork had been correct the entire time. Your machine, not the code, was the variable. The lesson, stated plainly at the time and worth repeating: **when every diagnostic comes back clean and the person swears the behavior used to be different, suspect system state you can't observe — and check for a pending reboot far earlier in the troubleshooting order than I did.**

---

## Part 9 — The v1.0.1 Handoff

That same evening, we packaged a full release handoff for your coding agent: a master document with the bug chain and all five required patches, an execution checklist, release notes, and a README troubleshooting snippet (including "if performance suddenly craters, check for a pending Windows Update reboot"). Patches specified: remove the stub-wheel function entirely; put the real `rocm-7.1.1.tar.gz` first in the wheel asset list; swap the CPU-index `requirements.txt` call for a ROCm-safe one with no torch pins; patch the watermarker in *both* `tts.py` and `tts_turbo.py` (a second instance of the same bug, found only when you hot-swapped to Turbo); and add a `verify_torch_rocm()` post-install check that aborts the install outright if it detects a CPU wheel, a missing `rocm_sdk`, or `cuda_available == False`.

---

## Part 10 — Three Weeks Later

**May 9.** You returned with:

> "i basically got it working! it has been running my daily podcast for a couple weeks now."

I read back through the compacted summary and all the source transcripts and reconstructed the full arc above, correctly as far as the chat record went — but working entirely from the chat's own memory, with no visibility into what had happened on the Paperclip board in parallel.

---

## Part 11 — Where the First Retelling Was Wrong

You then supplied `PROJECT_NOTES.md`, an uploaded document describing the Paperclip-side engineering. I used it uncritically to write a second narrative, and it was carrying real errors forward — not caught until the doc's *own* corrected version was found later in the actual repo:

- **The wheel-hosting story was wrong.** I described wheels being served from `wheels.phatt.vip` over an nginx pip index. That track was proposed, then explicitly killed by you on the board: *"no need for hosting on an ip. it will only call locally. just run chatterbox tts server as is."* After a real attempt to just link the ~2.1GB wheel files through Nextcloud failed (file-size cap), they landed on a **GitHub release, `v0.1.0-rocm-wheels`**, pulled at install time. This pivot — index to release — was, per the corrected board record, "the single most important architecture decision in the project."
- **The "four-layer bug" from Part 7 was a real, verified fix — but it undersold the full v1.0.0 debugging saga on the board.** The board's parallel thread (`PHA-276`) ran to 119 comments and surfaced at least three more distinct failures never mentioned in my first retelling: a **phantom PyPI dependency** (`torch`'s metadata claimed `rocm==7.1.1`, but no such package exists on PyPI — pip found an unrelated, real `rocm==0.1.0` package and collapsed), a **Python version mismatch** (the wheels are cp312-only; the portable launcher's embedded Python was 3.10), and a **Linux-only GPU detection path** that had to be rewritten for Windows using `vulkaninfo`, then registry scan, then WMIC as fallbacks.
- **The 7-day validation soak never happened.** I implied it had. It didn't — `validation-report.md` is an honest framework with the soak-data rows still marked TODO. No numbers were fabricated to fill it.

---

## Part 12 — The Corrected Record, and Auditing the Audit

`PROJECT_NOTES.md` itself had already been corrected once, on June 14, when — on your own instruction in the board issue that closed the project — an agent read all 22 tracked issues end to end and rewrote the document against what had actually happened, catching the errors above before I ever saw an old copy of it. That corrected file is real, accurate, and lives permanently in the repo.

But asked directly whether *that* corrected doc told the whole story end to end, the honest answer was no. Auditing it against the board turned up three more gaps:

1. **Everything in Parts 1–5 of this document** — the entire solo discovery arc, from the original ask through the first RTF 1.03x — happened in this chat, before any Paperclip ticket existed. The board's project starts from "Brandon had a working `EXECUTION_LOG`" as a given precondition. It was never captured as its own chapter anywhere on the board.
2. **The reboot mystery (Part 8) is one clinical sentence** in the board's closing record: *"Brandon worked it out on his machine on April 30."* The actual sequence — a real 6.69x regression, a dozen clean diagnostics, two hours of me being wrong, you correctly refusing to accept it, and the fix landing ten minutes after I told you to give up for the night — isn't preserved anywhere on the board.
3. **A genuine post-completion gap, not a narrative one.** On July 6, three weeks after the project's official close, you opened a fresh review issue (`PHA-1319`) asking simply whether the fork could be improved. The review found that upstream `devnen/Chatterbox-TTS-Server` had shipped 14 commits since the fork point, including a fix for a **live path-traversal vulnerability (CWE-22)** in the exact endpoints the fork exposes — `/tts` and `/v1/audio/speech` accepted unsanitized `voice`, `predefined_voice_id`, and `reference_audio_filename` parameters, meaning `../../` sequences could read arbitrary files off the server. That fix, plus two chunking bug fixes and a language-parameter improvement, were cherry-picked and pushed to `main` on July 6. **`PROJECT_NOTES.md`'s last edit predates this by three weeks** — the file has never been updated to record that this vulnerability existed or that it's closed. Anyone reading the permanent doc today has no way to know the exposure ever existed, or to confirm they're running a commit that fixes it.

---

## Where Things Stand

- The fork works, is running your daily podcast pipeline, and has been stable for weeks.
- `PROJECT_NOTES.md` in the repo is accurate for everything it covers, but doesn't cover this chat's own discovery arc, doesn't preserve the reboot-night debugging lesson, and — the one that matters operationally — is silently out of date on a security fix that landed after the doc was last touched.
- This document is the first place all three of those gaps, plus the full chat-originated story, live together in one place.

## Open Item

`PROJECT_NOTES.md` should get an addendum recording the July 6 security patch at minimum — that's not a storytelling nicety, it's a currently-missing fact about what commit is safe to run. Whether the pre-history and reboot-night material also belong folded into the permanent repo doc, or are better preserved here as a standalone companion piece, is your call.

---

*Sources: full chat transcript history (April 19 – August 2026, across all compactions), `PROJECT_NOTES.md` as committed in `phattbeats/Chatterbox-TTS-amd` at commit `a4e9047`, and the live Paperclip board (project `Chatterbox-TTS-Server- AMD Fork`, issues PHA-257 through PHA-1319).*
