# Ternary Bonsai 2 27B, RTX 3060 12GB on a dual Xeon E5606 (native Windows 10 Pro)

All numbers from one NVIDIA GeForce RTX 3060 (12 GB, sm_86) in a Dell Precision T7500 workstation: two Intel Xeon E5606 (Westmere, 4 cores each, 2.13 GHz, no AVX), PCIe 2.0, 60 GB RAM, native Windows 10 Pro (10.0.19045), driver 581.80, power limit 170 W (the default), Windows power plan Balanced. One machine, one card, my own runs on 2026-11-11, 2026-11-13, 2026-11-14 and 2026-11-15 (the power plan check was run on 2026-11-12). The card also drives the desktop, which was in normal use (a browser open) for every run except the cold-load test. The idle memory on the card before a run ranged from 285 MiB (fresh boot, nothing open) to 984 MiB, so every table that matters lists its own idle figure.

`rtx3060-12gb-windows.md` covers the same card in a Ryzen 7 5800XT, PCIe 3.0 system (my own earlier runs); this row is the same card, installer build and method on an old no-AVX workstation, so the two can be compared. The RX 570 8GB row in pull request #8 was run on this same machine.

Two passes. The first (2026-11-11 and 2026-11-12) used the Windows installer's own launcher and llama-bench. The second (2026-11-13 and 2026-11-14) ran the repo's two 12gb `serve/` lines by hand on the same installer build, with the repo's `probe.py`, `deep_probe.py` and `verify_identity.py`. Nothing was ported from another card.

A third pass (the evening of 2026-11-14 and the early morning of 2026-11-15) repeated the main lines on PrismML's own Windows release binary (`prism-b12128`) and added the vision projector. Those results are in their own sections ("PrismML's release build against the installer's bundle" and "Vision"); the first two passes are unchanged, and every table that does not say "release" is the installer's bundle.

## Short version

- **A Westmere-era dual Xeon with no AVX runs it.** The Windows installer's CUDA build loads its SSE4.2 CPU backend (`ggml-cpu-sse42.dll`) and generates normally. No out-of-memory and no failed request in any run below.
- **PrismML's own release build (`prism-b12128`) is faster on this card than the installer's bundle and uses less VRAM.** Same files, same flags, same machine, only the binary changed. Plain file: 43.3 tok/s against 35.7 in the same window (`probe.py` means, `-c 131072`, +21%); the full 262,144 line: 43.6 against 34.9 (+25%); `llama-bench` tg128 42.85 against 37.12 (+15%) and pp512 583.1 against 560.9 (+4%); the speed head: 54.1 against 36.6 to 38.7 in the installer build's windows on other days (+40% to +48%). Those figures are with `GGML_CUDA_BATCH_INVARIANT=1` set, as the repo's line sets it; with the switch unset the release build does 59.1 with the head and 43.3 without. The process held 430 to 446 MiB less at 131,072 and 884 MiB less at 262,144. Reading a long prompt was not faster (392 s to reach 120,655 tokens head off, against 374 s). The bullets below are the installer's bundle unless they say otherwise.
- **On the release build the head is worth much more here:** +29% on `probe.py` (54.1 against 41.9, same window, switch set; +36% with the switch unset, 59.1 against 43.3), +16% to +22% at depth, and +14% on prose, where the installer's bundle was not faster than no head. Draft acceptance is unchanged (0.93 on code, 0.57 on prose, 0.78 on bash).
- **In practice (installer's bundle):** about 35 tok/s on a fresh prompt with the head off (37 to 39 with the head), about 12 tok/s (13 with the head) once 120,655 tokens are already loaded, and about 6 to 7 minutes to read a 120K-token prompt from scratch (374 s head off, 402 s head on, filled in steps; 377 s in one request). Prefill ran at 240 to 506 tok/s (the rates fall as the context grows), so a very long prompt is a wait before the first token; I would use this machine for long jobs more than for live chat over a huge document.
- **The repo's `12gb-mtp.sh` line (speed head, n-max 1, `-c 131072`) decodes 36.6, 38.3 and 38.7 tok/s (`probe.py` means, three windows) against 35.2, 35.3 and 35.5 with the head off: +4% to +9%.** By prompt, code +17% to +20%, bash +7% to +10%, prose -12% to -1% (the head is not faster than no head on prose in any window). Draft acceptance: 0.90 to 0.95 on code, 0.47 to 0.67 on prose, 0.71 to 0.83 on bash.
- **Against the same card in the Ryzen row, head-off decode is about 10% lower and head-on about 22% lower** (means of the printed figures), so the head's gain is much smaller here (+4% to +9% against +23%). The gap shrinks with depth: head off is 5.5% lower at 76K tokens and 5.7% lower at 120K. The cause is not isolated (see "Anything odd"). That is the installer's bundle: on the release build this machine's head-on figure (54.1) is above the Ryzen row's installer-build figures (48.6 and 48.8), and I did not run the release build on the Ryzen machine.
- **The greedy text is byte-identical with and without the head** on all three `verify_identity.py` prompts (747 tokens in all), with `GGML_CUDA_BATCH_INVARIANT=1`. On the release build with the switch set code and bash are identical and prose differs at one near-tie token; with the switch unset only code is identical (see "Same text with and without the head, release build"). The switch costs that build 3% (head off) to 8.5% (head on) of its speed.
- **Draft n-max 2** (one window) averaged 37.7 tok/s, inside the spread of the n-max 1 windows (36.6 to 38.7). It wins on code (45.2) and loses on prose (29.7).
- **Depth** (131,072 window, one long document filled to each depth): decode at 17,841, 38,160, 75,894 and 120,655 tokens is 27.5, 22.1, 16.1 and 12.1 tok/s with the head off and 29.4, 23.6, 18.1 and 13.2 with the head on (+6.9%, +6.5%, +12.4%, +9.8%). The plain file in the 262,144 window decodes 11.9 tok/s at 120,655 tokens and 7.2 at 242,577.
- **The repo's `12gb.sh` line (plain file, `-c 262144`) is sensitive to free VRAM**: 34.9 tok/s with 317 MiB free after load, and 16.8 with 240 MiB free (same line, same file, same machine, different desktop load). The Ryzen row saw the same kind of step as an 8 to 9% slowdown. Mechanism not shown.
- **Cold load from a hard disk, 4 minutes after boot**: 67.3 s for the head line and 50.6 s for the 262,144 line; warm loads are 12 to 19 s.
- **Vision works on the 12 GB card.** The repo's mmproj Q8_0 (629 MB), added by hand to the 12gb lines (the repo has no 12 GB vision line), read the repo's chart right in all 15 requests (five arms, three each). With the full 262,144 window loaded on the release build the process spilled 612 MiB into shared memory with 411 MiB free, read text at 41.2 tok/s, and took 74 s for the first image; the same line on the installer's bundle spilled 1,442 MiB and read text at 27.3 (see "Vision").
- **CPU threads (1 to 8, and 4 with the server pinned to either socket) and the power plan (Balanced against Ultimate Performance) did not change decode** beyond about 3%.

## System

| Item | Value |
|---|---|
| GPU | MSI GeForce RTX 3060 VENTUS 2X OC 12GB (12288 MiB; HWiNFO reports GA106-302, LHR), factory boost clock 1807 MHz (MSI's spec page), driver 581.80, power limit 170 W |
| PCIe link | Gen 2 x16 under load (Gen 1 at idle), read from nvidia-smi and HWiNFO during the first pass; the card itself supports Gen 4 x16 (HWiNFO), the host runs it at 5.0 GT/s |
| CPU | 2x Intel Xeon E5606 (Westmere, 4 cores each, 2.13 GHz, no AVX, no Hyper-Threading, 8 logical processors) |
| RAM | 60 GB installed, about 50 GB reported usable |
| Machine | Dell Precision T7500 |
| OS | Windows 10 Pro 10.0.19045, on a SATA3 SSD behind a PCIe adapter |
| Model storage | the model files sit on a partition of a 2 TB hard disk on one of the board's own storage ports (not the OS drive) |
| Power plan | Balanced (the Windows default) for every result below except the power plan check; the Ryzen row used Ultimate Performance |
| Display | the card also drives the desktop (idle memory on the card 285 to 984 MiB before runs) |

## Build and files

- Installer: the repo's root `install.ps1` (release tag `v1.2-windows`), run with `BONSAI_NO_START=1`, a custom install folder and a custom port; at the model prompt I chose option 1 (Bonsai 2 27B). It chose the RTX 30/40 CUDA build and the 12gb tier and wrote the launcher `bonsai2.ps1` into the install folder. I am not claiming this copy of the script is identical to the ones run for the other rows.
- Build: the installer's prebuilt bundle `bonsai2-small-gpu-windows-x64-cuda12.4-sm86-sm89-ff41412`; `llama-server --version` reports `0.2.0-dev (build 10759, commit de947abc3)`, MSVC 19.44. This is the installer's bundle, not a build I made from the PrismML release or the `pr-ptq1-mmv` branch. The same build string appears in the Ryzen row.
- CPU backend loaded: `ggml-cpu-sse42.dll`.
- PrismML's release, used in the third pass: `prism-b12128-d7b2896` (the Windows CUDA 12.4 zip and its separate CUDA runtime zip from the release page, extracted into one folder); `llama-server --version` reports `0.7.0-dev (build 12128, commit d7b289683)`, MSVC 19.44. On this CPU it loaded `ggml-cuda.dll`, `ggml-rpc.dll` and `ggml-cpu-sse42.dll`. The repo's `serve/README.md` names `prism-b10685-7dffb15` or newer, so this is a build the repo's lines are written for. The repo's prebuilt bundles carry its own kernel patches (`serve/prebuilt_README.md`); This release reads `GGML_CUDA_BATCH_INVARIANT` (speed and text change with it, see "Decode with `probe.py`"), so it carries at least that switch; I do not know which of the other patches it contains, and its release page has no changelog.
- `Ternary-Bonsai-2-27B-PTQ1_0-mtp.gguf` (served file), 7,012,820,512 bytes, sha256 `83a396ee218c36e5ed88205eccb940a71549d9a72a3d020cc2713f94a78f70f0`. The installer fetched it and checked it against the `SHA256SUMS` file on Hugging Face (the installer reads it); I also hashed it by hand and it matches the size and hash printed in `graft/recipe.txt`.
- `Ternary-Bonsai-2-27B-PTQ1_0.gguf` (plain file, used for llama-bench and the 262,144 line), 5,946,648,928 bytes, sha256 `53107f530aa52eb00912263ab1ee29bd199261c87cd7b4ad4ca1318c1fe33ee3`, fetched from `prism-ml/Ternary-Bonsai-2-27B-gguf`; hashed by hand, matches the size and hash in `graft/recipe.txt` and the hash the Ryzen row lists.
- `Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf` (vision projector, third pass), 629,246,976 bytes, sha256 `6807ede61d570bb86ba34b756a0fa109edc33668604de867c6ea6d8f1d631903` (the value `install.ps1` pins), from `prism-ml/Ternary-Bonsai-2-27B-gguf`; the chart is the repo's `prompts/vision_check.png` (280,212 bytes).
- Python for the probes: the embeddable Python 3.12.10 (no packages beyond the standard library). Server bind: `127.0.0.1` only.

The two repo lines as run (model path removed). Only changes from `serve/12gb-mtp.sh` and `serve/12gb.sh`: `--port 8765` instead of 8899, and `--alias bonsai-2-27b` added.

```text
GGML_CUDA_BATCH_INVARIANT=1 llama-server -m Ternary-Bonsai-2-27B-PTQ1_0-mtp.gguf -c 131072 --spec-type draft-mtp --spec-draft-n-max 1 -ngl 99 -fa on -np 1 -ctk q4_0 -ctv q4_0 --jinja --alias bonsai-2-27b --host 127.0.0.1 --port 8765 --reasoning-effort medium --temp 1.0 --top-p 0.95 --top-k 20

llama-server -m Ternary-Bonsai-2-27B-PTQ1_0.gguf -c 262144 -ngl 99 -fa on -np 1 -ctk q4_0 -ctv q4_0 --jinja --alias bonsai-2-27b --host 127.0.0.1 --port 8765 --reasoning-effort medium --temp 1.0 --top-p 0.95 --top-k 20
```

The other arms are those lines with one change each: head off = the first line without the two `--spec-*` flags (MTP file, knob set); n-max 2 = the first line with `--spec-draft-n-max 2`; plain file pinned = the second line with `-c 131072`. The thread and pinning arms add `-t 4`. `GGML_CUDA_BATCH_INVARIANT=1` was set for every MTP-file arm and not for the plain-file arms or llama-bench; I did not run the knob on its own on the installer's bundle; the release build was run both ways (see the release section). Servers were started by hand, one at a time, and killed between arms. The release-build arms are the same lines with the release's `llama-server`, and the knob was set for the same arms (every MTP-file arm) on both builds, except the four release arms in the knob-unset window (R11, R12, IR3 and IR4).

## A/B on the same card: decode with `probe.py`

The repo's `graft/probe.py`: 1 warm-up request, then 3 prompts (python function, mmap against read, bash watcher) x 3 runs, 400 tokens, thinking off. Medians of 3 per prompt, tok/s; the mean is the OVERALL line the script prints. Window 1 is 2026-11-13 09:27 (head arm only), window 2 is 09:33, window 3 is 10:01, window 4 is 20:54 (no GPU poll or counters running), window 5 is 2026-11-14 11:26 (plain 262,144 line only). "Idle" is whole-board `memory.used` before the server started, and "free" is what was left after load.

| arm | context | code | prose | bash | mean | when |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| head n-max 1 (`12gb-mtp.sh`) | 131,072 | 41.3 | 30.9 | 37.4 | **36.6** | window 1, idle 967 MiB |
| head n-max 1, repeated | 131,072 | 41.8 | 34.8 | 38.7 | **38.3** | window 3, idle 491 MiB |
| head n-max 1, repeated | 131,072 | 42.5 | 35.3 | 38.5 | **38.7** | window 4, idle 534 MiB |
| head n-max 2 | 131,072 | 45.2 | 29.7 | 37.6 | **37.7** | window 2, idle 972 MiB |
| head off | 131,072 | 35.2 | 35.2 | 35.1 | **35.2** | window 2, idle 984 MiB |
| head off, repeated | 131,072 | 35.3 | 35.5 | 35.1 | **35.3** | window 3, idle 498 MiB |
| head off, repeated | 131,072 | 35.5 | 35.7 | 35.5 | **35.5** | window 4, idle 527 MiB |
| plain file (`12gb.sh`) | 262,144 | 17.1 | 17.3 | 17.2 | **16.8** | window 2, idle 976 MiB, 240 MiB free |
| plain file, repeated | 262,144 | 34.8 | 34.9 | 34.8 | **34.9** | window 5, idle 514 MiB, 317 MiB free |
| plain file, pinned | 131,072 | 35.3 | 35.4 | 35.2 | **35.3** | window 2, idle 466 MiB |

Head off ranged 35.2 to 35.5 across three windows (under 1%). The head's gain, mean against mean: +4.0% (window 1 against window 2, six minutes apart and both at a high idle level), +8.5% (window 3) and +9.0% (window 4). Head-on means ranged 36.6 to 38.7 (5%); the spread is mostly the prose prompt (30.9 to 35.3), where acceptance moves from run to run because `probe.py` samples at temperature 1.0 without a seed. By prompt against the head-off figure of the same or nearest window: code +17.3%, +18.4%, +19.7%; bash +6.6%, +10.3%, +8.5%; prose -12.2%, -2.0%, -1.1%.

The plain-file 262,144 run in window 2 had one lower run (13.3 tok/s, the third code run); its other eight runs ranged 16.4 to 18.0.

### Draft acceptance (from the server log, per request)

Window 3 for n-max 1, window 2 for n-max 2. Acceptance is accepted over generated draft tokens; mean accepted length is the server's figure.

n-max 1 (`12gb-mtp.sh`):

| request | acceptance | mean accepted length |
| --- | ---: | ---: |
| warm-up | 0.583 | 1.58 |
| code, 3 runs | 0.944, 0.917, 0.902 (mean 0.921) | 1.94, 1.92, 1.90 |
| prose, 3 runs | 0.585, 0.512, 0.591 (mean 0.563) | 1.59, 1.51, 1.59 |
| bash, 3 runs | 0.727, 0.830, 0.773 (mean 0.777) | 1.73, 1.83, 1.77 |

n-max 2 (maximum length 3):

| request | acceptance | mean accepted length |
| --- | ---: | ---: |
| warm-up | 0.714 | 2.43 |
| code, 3 runs | 0.833, 0.928, 0.874 (mean 0.878) | 2.66, 2.85, 2.75 |
| prose, 3 runs | 0.380, 0.382, 0.429 (mean 0.397) | 1.76, 1.76, 1.86 |
| bash, 3 runs | 0.616, 0.685, 0.637 (mean 0.646) | 2.23, 2.37, 2.27 |

Across the three n-max 1 windows, per-request acceptance ranged 0.899 to 0.951 on code, 0.472 to 0.667 on prose and 0.708 to 0.834 on bash. Speed follows acceptance. In the depth runs below the head accepted 0.64 to 0.76.

## Same text with and without the head

`graft/tools/verify_identity.py`, run on 2026-11-14 at 12:26: three prompts (code, prose, bash), greedy (temperature 0, top_k 1, seed 42), 300 tokens, thinking off, once with the head off (the first line without the two `--spec-*` flags, knob set) and once with the head on (the first line as written), then the script's byte comparison.

| prompt | tokens | result | bytes | draft accepted, head on |
| --- | ---: | --- | ---: | ---: |
| code | 300 | IDENTICAL | 1054 | 143 of 156 |
| prose | 147 | IDENTICAL | 789 | 58 of 89 |
| bash | 300 | IDENTICAL | 1112 | 132 of 166 |

The three sha256 prefixes of the text matched between the two arms (`9948638b4f06bc32`, `591a3caa2c453b62`, `8e08e427b653fa1a`). Code and bash stopped at the 300-token cap; prose finished on its own. This covers three prompts at temperature 0 and says nothing about sampled output. The speeds that script prints (26.9 to 27.0 tok/s off, 35.8 to 41.0 on) come from non-streaming requests with logprobs on, so they are not comparable with the `probe.py` table.

## Draft n-max 2

One window (window 2, idle 972 MiB): mean 37.7 against 36.6 for n-max 1 in the window before it (window 1) and 38.3 and 38.7 in the two windows after; code is the one prompt where it clearly wins (45.2 against 41.3 to 42.5), prose clearly loses (29.7 against 30.9 to 35.3, acceptance 0.38 to 0.43), and bash is level (37.6 against 37.4 to 38.7). I would not call n-max 2 better or worse overall from one window. The process held 10,774 MiB dedicated against 10,624 for n-max 1.

## Depth: decode at real context depth

`deep_probe.py` with a 181,385-byte plain-text file of my own (the repo does not ship a corpus). Prompt sizes were set by tokenizing against the running server (`/tokenize`), so the depths shown are the real token counts. 2 runs per depth, 300 generated tokens, server-reported tok/s, median of the runs (run 1 prefills the new part, run 2 hits the cache). 2026-11-14, 09:13, one server per arm in the 131,072 window; the last row is the plain file in the 262,144 window (11:30).

| depth (tokens) | head off | head n-max 1 | head gain | plain file, 262,144 window |
| ---: | ---: | ---: | ---: | ---: |
| 17,841 | 27.53 | 29.43 | +6.9% | not run |
| 38,160 | 22.11 | 23.55 | +6.5% | not run |
| 75,894 | 16.08 | 18.07 | +12.4% | not run |
| 120,655 | 12.05 | 13.23 | +9.8% | 11.88 |
| 242,577 | not run | not run | | **7.15** |

Against the fresh `probe.py` means (35.3 head off, 37.9 head on, each the mean of its three windows), decode keeps 78%, 63%, 46% and 34% of its fresh speed with the head off and 78%, 62%, 48% and 35% with the head on; `probe.py` is a different prompt set from the corpus, so these shares are approximate. At 120,655 tokens the plain file decodes 11.88 in the 262,144 window against 12.05 in the 131,072 window, so the larger window costs nothing at the same depth. The plain file at 242,577 tokens decodes at 60% of its 120,655 figure.

Each depth shares its first part with the one before it, so after the first depth the server's cache reused that part. Run 1 covers only the new tokens, and its first-token time includes waiting for the first token, so these prefill rates are lower bounds:

| step | new tokens | head off | head n-max 1 |
| --- | ---: | ---: | ---: |
| to 17,841 | 17,853 | 35.3 s, 506 tok/s | 37.8 s, 472 tok/s |
| to 38,160 | 20,835 | 49.4 s, 422 tok/s | 52.9 s, 394 tok/s |
| to 75,894 | 38,250 | 114.2 s, 335 tok/s | 122.4 s, 312 tok/s |
| to 120,655 | 45,277 | 175.4 s, 258 tok/s | 188.8 s, 240 tok/s |

The head costs about 7% of prefill speed (6.6% to 7.1% per step). Filling to 120,655 tokens in steps took 374.3 s head off and 401.9 s head on. The plain file in the 262,144 window took 377.0 s to prefill 120,667 tokens from an empty cache in one request (320 tok/s) and 715.8 s for the next 122,438 tokens (171 tok/s), 1,092.8 s (18.2 minutes) in all to reach 242,577 tokens. No out-of-memory and no failed request at any depth. The plain-file run loaded at 11,984 MiB and ended at 12,010 MiB with 304 MiB free at load.

## PrismML's release build against the installer's bundle

Third pass, 2026-11-14 and 2026-11-15 (windows at 22:51, 23:39 and 00:31; the two vision windows are in their own section). The same two model files, the same repo lines, the same machine, one server at a time; the only change is the binary: the installer's bundle (`0.2.0-dev`, build 10759) or PrismML's release `prism-b12128-d7b2896` (`0.7.0-dev`, build 12128). Both are CUDA 12.4 builds and both loaded the SSE4.2 CPU backend. Window 6 (22:51) ran two llama-bench runs and four probe arms, one of them the installer's bundle as the control in the same window; window 7 (23:39) ran the 262,144 line, the two depth ladders and the identity pair; window 8 (2026-11-15, 00:31) ran the release build's head and head-off probe arms and the identity pair again with `GGML_CUDA_BATCH_INVARIANT` unset. The desktop was in normal use (idle 394 to 483 MiB).

### llama-bench (same window, plain file, 3 repeats)

| build | pp512 tok/s | tg128 tok/s |
| --- | ---: | ---: |
| installer's bundle (10759) | 560.88 ± 16.80 | 37.12 ± 0.21 |
| release (12128) | 583.14 ± 2.18 | 42.85 ± 0.33 |

pp512 +4.0%, tg128 +15.4%. The lines are verbatim in "llama-bench lines verbatim".

### Decode with `probe.py`

Same method as the A/B table (medians of 3 per prompt, the mean is the OVERALL line). "Idle" and "free" as there.

| arm | build | context | code | prose | bash | mean | idle / free |
| --- | --- | ---: | ---: | ---: | ---: | ---: | --- |
| plain file, control | installer | 131,072 | 35.7 | 35.8 | 35.6 | **35.7** | 474 / 3,097 MiB |
| plain file | release | 131,072 | 43.2 | 43.9 | 43.1 | **43.3** | 469 / 3,547 MiB |
| plain file (`12gb.sh` as written) | release | 262,144 | 43.4 | 44.1 | 43.4 | **43.6** | 419 / 1,165 MiB (window 7) |
| head off | release | 131,072 | 41.8 | 42.3 | 41.7 | **41.9** | 476 / 3,538 MiB |
| head n-max 1 (`12gb-mtp.sh`) | release | 131,072 | 59.3 | 48.2 | 54.5 | **54.1** | 469 / 1,683 MiB |
| head off, switch unset (window 8) | release | 131,072 | 43.2 | 43.8 | 43.1 | **43.3** | 428 / 3,589 MiB |
| head n-max 1, switch unset (window 8) | release | 131,072 | 64.7 | 52.2 | 60.2 | **59.1** | 431 / 1,722 MiB |

The installer build's figures from other windows are in the A/B table above (262,144 line 34.9, head off 35.2 to 35.5, head n-max 1 36.6 to 38.7). The larger window costs nothing on the release build (43.6 at 262,144 against 43.3 at 131,072). With 1,165 MiB free the 262,144 line is far above the 240 to 304 MiB where the installer build's line slowed down; I did not look for the release build's step. The MTP file with the head off decoded 41.9 with the switch set against 43.3 for the plain file (which had it unset); with the switch unset the MTP file decodes 43.3 too (window 8), so the 3% was the switch and not the file. The first five rows have the switch set on the MTP-file arms, as the repo's `12gb-mtp.sh` sets it, and unset on the plain-file arms, as `12gb.sh` has it; the last two rows have it unset.

The head's gain on the release build, mean against mean: **+29.1%** (54.1 against 41.9). By prompt: code +41.9%, prose +13.9%, bash +30.7%, against +17% to +20%, -12% to -1% and +7% to +10% on the installer's bundle. Acceptance per request (window 6): code 0.909, 0.928, 0.945 (mean 0.927), prose 0.538, 0.624, 0.538 (0.566), bash 0.777, 0.805, 0.753 (0.779), with mean accepted length 1.54 to 1.94. That is the same as the installer build's 0.921, 0.563 and 0.777, so the larger gain does not come from the head being accepted more often.

### The batch-invariant switch on the release build (set against unset)

`GGML_CUDA_BATCH_INVARIANT=1` is set by the repo's `12gb-mtp.sh`. Window 8 repeated the release build's two MTP-file arms with it unset:

| arm | switch | code | prose | bash | mean | acceptance, code / prose / bash |
| --- | --- | ---: | ---: | ---: | ---: | --- |
| head n-max 1 | set (window 6) | 59.3 | 48.2 | 54.5 | **54.1** | 0.927 / 0.566 / 0.779 |
| head n-max 1 | unset (window 8) | 64.7 | 52.2 | 60.2 | **59.1** | 0.936 / 0.545 / 0.783 |
| head off | set (window 6) | 41.8 | 42.3 | 41.7 | **41.9** | |
| head off | unset (window 8) | 43.2 | 43.8 | 43.1 | **43.3** | |

With the switch unset this build is 9.2% faster with the head and 3.3% faster without it, so the switch costs 8.5% and 3.2% of its speed here. The head's gain is +36.5% unset (code +49.8%, prose +19.2%, bash +39.7%) against +29.1% set. Acceptance is the same either way (unset, per request: code 0.930, 0.936, 0.941; prose 0.593, 0.522, 0.522; bash 0.773, 0.809, 0.765). The switch changes the numbers and not only the speed (the head-off greedy text itself differs between set and unset, see the identity table below), so this build reads it. The process memory is the same (10,182 against 10,178 MiB with the head, 8,310 MiB without). Window 8 ran about 30 minutes after window 7 ended, with the desktop as before (idle 428 to 431 MiB).

### Depth (131,072 window, same corpus and the same real token counts as above)

2 runs per depth, 300 generated tokens, server-reported tok/s, median of the two. Head n-max 1 at about 23:41, head off at about 23:50, 2026-11-14.

| depth (tokens) | head off | head n-max 1 | head gain | installer, head off | installer, head n-max 1 | release against installer (off / on) |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 17,841 | 31.87 | 37.98 | +19.2% | 27.53 | 29.43 | +15.8% / +29.1% |
| 38,160 | 25.24 | 30.89 | +22.4% | 22.11 | 23.55 | +14.2% / +31.2% |
| 75,894 | 18.12 | 21.60 | +19.2% | 16.08 | 18.07 | +12.7% / +19.5% |
| 120,655 | 13.49 | 15.71 | +16.5% | 12.05 | 13.23 | +12.0% / +18.7% |

The head accepted 0.65 to 0.76 at depth, the same range as on the installer's bundle. Prefill of the new part at each step (first-token time of run 1, so these rates are lower bounds, as above):

| step | new tokens | head off | head n-max 1 |
| --- | ---: | ---: | ---: |
| to 17,841 | 17,853 | 35.0 s, 510 tok/s | 38.1 s, 469 tok/s |
| to 38,160 | 20,835 | 50.4 s, 413 tok/s | 54.3 s, 384 tok/s |
| to 75,894 | 38,250 | 119.7 s, 320 tok/s | 129.0 s, 297 tok/s |
| to 120,655 | 45,277 | 187.3 s, 242 tok/s | 202.8 s, 223 tok/s |

Filling to 120,655 tokens took 392.4 s head off and 424.2 s head on, which is 4.8% and 5.5% longer than the installer's bundle (374.3 s and 401.9 s). The release build is level at the first step (510 against 506 tok/s head off) and slower deeper (242 against 258 at the last step); the head costs 7% to 8% of prefill speed on both builds. So the release build's gain is in decode and not in reading a long prompt. No out-of-memory and no failed request.

### Same text with and without the head, release build

`verify_identity.py` as above (three prompts, greedy, temperature 0, top_k 1, seed 42, 300 tokens, thinking off), 2026-11-14 at 23:59 to 2026-11-15 at 00:00, `GGML_CUDA_BATCH_INVARIANT=1` set for both arms, head off against the head on (n-max 1).

| prompt | tokens, off / on | result | bytes, off / on | draft accepted, head on |
| --- | ---: | --- | ---: | ---: |
| code | 300 / 300 | IDENTICAL | 1056 | 142 of 157 |
| prose | 147 / 155 | DIFFERS at byte 471 | 789 / 827 | 64 of 91 |
| bash | 300 / 300 | IDENTICAL | 1115 | 133 of 165 |

The prose texts agree for the first 471 bytes and then split at token step 93, where two tokens were close: with the head off the model chose " the" (logprob -0.7548, against -0.8508 for " a"); with the head on it chose " a" (-0.7984, against -0.8085 for " the"), a gap of 0.01. The same token has a different logprob in the two runs (-0.8508 and -0.7984) although the text before it is identical, so the logits at that step were not bit-identical between the two runs, and a near tie flipped. Both continuations are valid text. The installer's bundle gave identical text on all three prompts. Across the two builds only the prose text with the head off matched (sha256 prefix `591a3caa2c453b62` on both); the code and bash texts differ from the installer's by a few bytes (1056 and 1115 bytes against 1054 and 1112). The repo's serve notes call the head lossless with `GGML_CUDA_BATCH_INVARIANT=1` "on the fork branch", the switch is one of the repo's kernel patches (`kernel/PR_KERNEL.md`), and `graft/recipe.txt` says output without it is not identical. I ran the release build both ways (window 8, 2026-11-15 at 00:31, the same lines with the switch unset):

| prompt | switch set | switch unset |
| --- | --- | --- |
| code | IDENTICAL (1056 bytes) | IDENTICAL (1033 bytes) |
| prose | DIFFERS at byte 471 (token step 93) | DIFFERS at byte 241 (token step 48) |
| bash | IDENTICAL (1115 bytes) | DIFFERS at byte 138 (token step 35) |

Every divergence is a near tie: in both runs the two tokens were within 0.10 of each other in logprob, and within 0.02 in the head-on run (unset prose: " allocate" -1.1646 against " manage" -1.2049 with the head off, " manage" -1.1649 against " allocate" -1.1744 with it on; unset bash: a double newline -0.8438 against " [" -0.9228 off, " [" -0.8726 against -0.8913 on). The switch also changes the head-off text itself: with it unset the head-off code and prose texts have different sha256 prefixes (`abd40b975ef179e6` against `2ff5fe93d7d3568f` for code, `9889b0ee6fbd66f4` against `591a3caa2c453b62` for prose; bash is the same, `3b15237fcaf7c2e8`), which is the second sign that this build reads the switch. On this build the head matched on two of three prompts with the switch set and one of three with it unset; the installer's bundle matched on all three with it set (not run unset). The speeds the script prints (40.6 to 41.0 tok/s off, 50.5 to 56.3 on with the switch set; 42.0 to 42.4 off, 54.8 to 61.2 on unset) are from non-streaming requests with logprobs on and are not comparable with the `probe.py` table.

### Hardware readings during the window 7 runs (HWiNFO, about 2 s samples)

The HWiNFO log covers 23:38 to 23:59:58, so it stops 40 s before the last arm (the head-on identity run) ended; the other four arms are covered. Arms are matched to the log by start time and duration.

| arm | GPU core load | memory controller load | CPU1 / CPU2 busy | mean power | max temp |
| --- | ---: | ---: | ---: | ---: | ---: |
| plain file, 262,144, `probe.py` (shallow decode) | 84.7% | 86.7% | 12.6% / 26.7% | 160.7 W | 71.9 C |
| depth ladder, head n-max 1 | 94.9% | 28.1% | 26.8% / 10.2% | 160.6 W | 73.6 C |
| depth ladder, head off | 98.4% | 31.4% | 26.6% / 8.9% | 165.7 W | 73.9 C |
| identity, head off (18 s, 10 samples) | 71.3% | 65.4% | 12.8% / 25.3% | 156.8 W | 69.0 C |

The depth ladders are mostly prefill, which is compute-bound (core load 95% to 98%, memory controller about 30%); shallow decode shows a lower core load and a heavy memory controller. The link read 5.0 GT/s under load in every segment (PCIe 2.0). The CPU columns are per-socket averages over four cores: the busier socket read 25% to 27% (one core's worth) and the other 9% to 13%.

## Vision (mmproj Q8_0)

The repo's 12gb serve lines have no projector (`serve/16gb-mtp-vision.sh` is the vision line, for 16 GB). I added `--mmproj Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf` to the 12gb lines by hand to see what a 12 GB card does. The check is the installer's own: the repo's `prompts/vision_check.png` with the prompt the installer sends (the chart has a group called "fresh chat" with three bars; give the three tok/s values as JSON with keys stock, kernel, kernel_plus_head), temperature 0, top_k 1, seed 42, thinking off, non-streaming, three requests per arm, and "read right" means 25.0, 39.8 and 50.1 all appear in the answer (the installer's pass criterion). The script that sends the three requests is a small standard-library helper of mine (not in the repo). It runs first on a fresh server; `probe.py` then measures text speed with the projector loaded. V1 to V3 (installer's bundle) ran 2026-11-14 at 23:00; V4 and V5 (release) ran 2026-11-15 at 00:00.

| arm | context | board after load (free) | process dedicated / shared | first image request | decode with the image in context | `probe.py` mean |
| --- | ---: | ---: | ---: | --- | ---: | ---: |
| V1 installer, plain file | 131,072 | 10,056 MiB (2,232) | 9,834 / 218 MiB | 261.6 tok/s | 33.0 | 35.6 |
| V2 installer, head n-max 1 | 65,536 | 10,143 MiB (2,145) | 10,176 / 274 MiB | 262.8 tok/s | 39.7 | 39.5 |
| V3 installer, plain file | 262,144 | 11,995 MiB (293) | 11,684 / 1,442 MiB | 11.4 tok/s | 26.1 | 27.3 |
| V4 release, plain file | 262,144 | 11,877 MiB (411) | 11,629 / 612 MiB | 41.1 tok/s, 74.2 s | 38.9 | 41.2 |
| V5 release, head n-max 1 | 65,536 | 9,845 MiB (2,443) | 9,800 / 294 MiB | 316.7 tok/s, 10.5 s | 55.9 | 53.4 |

- **Read right in 3 of 3 requests on every arm** (15 of 15). The image is 3,002 prompt tokens.
- **Only the first request measures reading an image.** Requests 2 and 3 send the same image and reuse the cache (on V4 and V5 the server reported 2,998 of 3,002 tokens cached), so their ~10 tok/s "prefill" says nothing about image speed. V1 to V3 ran with an earlier version of my helper that did not print wall time or cached tokens, so their first-image speed is the printed prefill rate only; V3's 11.4 tok/s works out to about 263 s (calculated).
- **Text speed with the projector loaded** matches the no-projector figure when nothing spills: V1 35.6 against 35.7 for the same line without it. At 262,144 the installer build spilled 1,442 MiB and decoded 27.3 against 34.9 without the projector (-22%); the release build spilled 612 MiB and decoded 41.2 against 43.6 (-5.5%) but took 74 s for the first image, against about 10 s for the head line at 65,536.
- **Head with the projector, 65,536 window:** 53.4 on the release build against 39.5 on the installer's bundle (+35%); the head accepted 15 of 15 draft tokens on the image requests on both builds.
- I did not run the head with the projector at 131,072 or 262,144. This is one chart and checks that the projector loads, reads a chart and what it does to text speed; it says nothing about vision quality.

## Cold load

`/health` first answering after process start, 2026-11-14 at 10:32, 4 minutes after a reboot, with only the terminal and a few system apps open and the model files read from the hard disk described under System (the operating system is on a separate SSD). Idle memory on the card was 285 MiB.

| line | cold load | warm loads (earlier windows) | board after load |
| --- | ---: | ---: | ---: |
| `12gb-mtp.sh` (head, 131,072) | 67.3 s | 15.2 to 18.5 s | 10,892 MiB |
| `12gb.sh` (plain, 262,144) | 50.6 s | 12.4 to 14.2 s | 11,952 MiB |

Cold is about 3.6 to 4.4 times the warm load for the head line. The files are 7.0 GB (head line) and 5.9 GB (plain line), so those loads ran at about 104 and 118 MB/s, an ordinary sequential rate for a hard disk. The plain line loaded faster cold in part because its file is smaller; it also ran second, and I cannot say how much each reason counts.

## CPU threads, pinning and power plan

A two-socket machine with no Hyper-Threading: logical processors 0 to 3 are one socket and 4 to 7 the other. These six arms use `-t 4` so a pinned server is not oversubscribed; every other arm used the default of 8 threads. Affinity was set on the running server and read back (`0xF` and `0xF0`). 2026-11-13, 23:20, one window.

| arm | head | pinned to | code | prose | bash | mean | HWiNFO busy, CPU1 / CPU2 (sum of 4 cores, %) |
| --- | --- | --- | ---: | ---: | ---: | ---: | --- |
| P0 | off | none | 35.1 | 35.2 | 35.1 | **35.1** | 130 / 76 |
| P1 | off | processors 0 to 3 | 34.6 | 34.7 | 34.7 | **34.7** | 143 / 51 |
| P2 | off | processors 4 to 7 | 35.2 | 35.5 | 35.2 | **35.3** | 66 / 134 |
| P3 | on | none | 41.4 | 33.7 | 38.6 | **38.0** | 135 / 60 |
| P4 | on | processors 0 to 3 | 41.2 | 34.3 | 38.5 | **37.9** | 149 / 42 |
| P5 | on | processors 4 to 7 | 41.4 | 33.0 | 38.9 | **37.7** | 70 / 130 |

Pinning to either socket moved the mean by at most 1.7% head off and 0.8% head on, with the sign changing between the two head states, which I read as noise; `-t 4` against the default 8 threads moved it by under 1% (35.1 against 35.3 head off, 38.0 against 37.9 head on). The HWiNFO column (sums of per-core usage averaged over each arm's window, calculated from its CSV) shows the load following the pinned socket. The first pass tested threads with llama-bench on the plain file (`-t 1,2,4,8`): 36.18, 36.80, 36.72 and 37.11 tok/s, so with every layer on the GPU the thread count barely moves decode. HWiNFO read the busiest core at 2,128 MHz (this CPU has no Turbo), so the clock is not held low.

### Power plan (first pass, plain file, repo flags, three invocations per plan)

| Plan | tg128 tok/s, invocations 1 / 2 / 3 | tg128 mean | pp512 tok/s, invocations 1 / 2 / 3 | pp512 mean |
|---|---|---|---|---|
| Balanced | 36.98 / 37.12 / 36.83 | 36.98 | 563.73 / 560.72 / 558.26 | 560.9 |
| Ultimate Performance | 37.06 / 37.25 / 36.59 | 36.97 | 560.26 / 558.98 / 559.56 | 559.6 |

The plan was set with `powercfg /setactive` and read back with `powercfg /getactivescheme` right after the switch and after the three invocations; no server was running. Means are calculated. Decode and prompt processing did not change.

## Served VRAM

Whole-board `memory.used` from `nvidia-smi --query-gpu=memory.used` (per-process memory reads N/A under WDDM). The process columns come from the Windows counters `\GPU Process Memory(pid_<pid>_*)\Dedicated Usage` and `Shared Usage`, sampled every 2 s during the run (maximum shown).

| arm | context | idle before | board after load | board after run | process dedicated | process shared |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| head off | 131,072 | 498 MiB | 9,222 MiB | 9,250 MiB | 8,740 MiB | 218 MiB |
| head off | 131,072 | 984 MiB | 9,693 MiB | 9,714 MiB | 8,740 MiB | 218 MiB |
| plain file, pinned | 131,072 | 466 MiB | 9,194 MiB | 9,216 MiB | 8,740 MiB | 218 MiB |
| head n-max 1 | 131,072 | 491 MiB | 11,094 MiB | 11,124 MiB | 10,624 MiB | 402 MiB |
| head n-max 1 | 131,072 | 967 MiB | 11,583 MiB | 11,599 MiB | 10,624 MiB | 402 MiB |
| head n-max 2 | 131,072 | 972 MiB | 11,730 MiB | 11,752 MiB | 10,774 MiB | 404 MiB |
| plain file, first run | 262,144 | 976 MiB | 12,048 MiB | 12,044 MiB | 11,578 MiB | 452 MiB |
| plain file, repeated | 262,144 | 514 MiB | 11,971 MiB | 11,999 MiB | 11,620 MiB | 410 MiB |
| plain file, depth run | 262,144 | 444 MiB | 11,984 MiB | 12,010 MiB | 11,626 MiB | 410 MiB |
| release, plain file | 131,072 | 469 MiB | 8,741 MiB | 8,781 MiB | 8,310 MiB | 238 MiB |
| release, MTP file, head off | 131,072 | 476 MiB | 8,750 MiB | 8,786 MiB | 8,308 MiB | 238 MiB |
| release, head n-max 1 | 131,072 | 469 MiB | 10,605 MiB | 10,656 MiB | 10,178 MiB | 422 MiB |
| release, plain file | 262,144 | 419 MiB | 11,123 MiB | 11,151 MiB | 10,742 MiB | 366 MiB |

The process figures match the Ryzen row to the MiB for the arms both rows ran (8,740 / 218, 10,624 / 402, 10,774 / 404 and 11,620 / 410). The board figure moves with the desktop: after a run that filled the board, the idle level before the next run dropped (466 MiB before the pinned plain-file arm that followed the slow 262,144 run), which looks like Windows moving desktop memory out of the card. In the depth runs the process held 10,634 MiB dedicated with the head and 8,746 without, and nothing grew as the context filled. The first pass measured 10,714 MiB loaded with the head at the installer's own 110,336-token window (695 MiB idle). The release rows are the third pass: its process held 430 to 446 MiB less than the installer's bundle at 131,072 (8,310 against 8,740 MiB without the head, 10,178 against 10,624 with it) and 884 MiB less at 262,144 (10,742 against 11,626 MiB); the board figure after load on the 262,144 line was 861 MiB lower (11,123 against 11,984 MiB). Vision arms are in their own section.

## Anything odd

**On the installer's bundle, head-off decode is about 10% below the same card in the Ryzen row and the head's gain is +4% to +9% here against +23% there, and the repo's 262,144 line ran at 16.8 tok/s with 240 MiB free and 34.9 with 317 MiB free; I have not isolated the cause of either. On PrismML's newer release build the head's gain on this machine is +29% (54.1 against 41.9; +36% with the batch-invariant switch unset), so the size of the gap depends on the build; I did not run the release build on the Ryzen machine, so I cannot say how much of the gap was the machine and how much the build.**

### Against the same card in the Ryzen row

Differences are calculated from the two rows' printed figures; both rows use the installer's bundle (the same build string), the same flags, the same repo scripts and a corpus of the same size; the release build is a different binary and is not in this comparison. The Ryzen row's depths are 76,004 and 119,093 tokens, mine 75,894 and 120,655.

| measure | this row | Ryzen row | difference |
| --- | ---: | ---: | ---: |
| llama-bench tg128, plain file | 36.98 | 39.58 | -6.6% |
| llama-bench pp512 | 560.9 | 577.83 | -2.9% |
| `probe.py` mean, head off (131,072) | 35.2, 35.3, 35.5 | 39.1 to 39.8 (four windows) | -10.4% (means) |
| `probe.py` mean, head n-max 1 | 36.6, 38.3, 38.7 | 48.6, 48.8 | -22.2% (means) |
| head gain on `probe.py` | +4.0%, +8.5%, +9.0% | +23%, +23% | |
| depth about 76K, head off | 16.08 | 17.01 | -5.5% |
| depth about 76K, head n-max 1 | 18.07 | 19.85 | -9.0% |
| depth about 120K, head off | 12.05 | 12.78 | -5.7% |
| depth about 120K, head n-max 1 | 13.23 | 14.60 | -9.4% |

The gap is largest on fresh decode and shrinks with depth. Candidates I could not separate on one machine: slower single-thread speed on the old cores, the SSE4.2 CPU backend, and PCIe 2.0 latency. Ruled out on this machine: the power plan (Ultimate Performance 36.97 against Balanced 36.98 on tg128), the thread count (1 to 8, and `-t 4`) and which socket the server runs on (above). One more reading: HWiNFO's GPU Core Load averaged 72% to 73% over the three head-off arms of the pinning window and 61% to 63% over the three head-on arms, against 94.6% over the whole 262,144 depth run (long prefill and deep decode), so the card appears to wait on something during shallow decode and not at depth. The Ryzen row does not report this, so it is not a comparison, and it does not say what the card waits on. The release build shows the same split (HWiNFO, window 7: core load 71% in a short head-off decode and 85% in the plain-file probe, against 95% to 98% in the depth ladders), and in every one of those segments the busier CPU socket read 25% to 27% (one core of its four) and the other 9% to 13%, which would fit one busy thread being the limit on these cores. HWiNFO gives per-socket totals only, so that is a hint and not a finding, and I did not test it. On the release build this machine's head-on `probe.py` mean (54.1) is above the Ryzen row's installer-build figures (48.6 and 48.8), but that compares two different binaries.

### The 262,144 line against free VRAM

`serve/12gb.sh` (plain file, `-c 262144`) decoded 16.8 tok/s the first time and 34.9 on a repeat the next day. Both runs used the identical command; the second had Firefox, Mullvad Browser and Task Manager closed and Brave open.

| | 2026-11-13 09:33 | 2026-11-14 11:26 |
| --- | ---: | ---: |
| idle before start | 976 MiB | 514 MiB |
| board after load (free) | 12,048 MiB (240 MiB) | 11,971 MiB (317 MiB) |
| process dedicated / shared, max | 11,578 / 452 MiB | 11,620 / 410 MiB |
| mean decode | 16.8 tok/s | 34.9 tok/s |
| mean board power, active | 109.4 W | 159.2 W |

The slow run had its dedicated memory 42 MiB lower and its shared memory 42 MiB higher, and the card drew 109 W against 159 W, so it was waiting more than computing. The depth run (444 MiB idle, 304 MiB free) was at full speed (11.88 tok/s at 120,655 tokens, matching the 131,072-window figure), so the step falls between 240 and 304 MiB free. The same file pinned at `-c 131072` (3,094 MiB free) gave 35.3, so the window size alone is not the cause. The Ryzen row saw the same pattern: an 8 to 9% slowdown with about 260 MiB free and a small shift of the process's memory from dedicated to shared. A small spill over PCIe 2.0 is consistent with a larger penalty here, but I have not shown spill is the cause, and the CUDA Sysmem Fallback Policy was left at the driver default. On the release build the same line ran at 43.6 tok/s with 1,165 MiB free (idle 419 MiB) and its process held 884 MiB less; I did not search for its step.

## Power and temperature

`nvidia-smi` every 0.4 to 0.5 s; "active" = board power 100 W or more. Board limit 170 W. Arms are the ones in the tables above.

| step | mean board power (active) | max board power | max GPU temp | mean SM clock |
| --- | ---: | ---: | ---: | ---: |
| probe, head off (window 2) | 160.2 W | 166.8 W | 72 C | 1,872 MHz |
| probe, head off (window 3) | 160.9 W | 167.9 W | 73 C | 1,863 MHz |
| probe, head n-max 1 (window 1) | 145.0 W | 153.2 W | 70 C | 1,870 MHz |
| probe, head n-max 1 (window 3) | 147.3 W | 153.7 W | 70 C | 1,865 MHz |
| probe, head n-max 2 | 145.2 W | 151.9 W | 70 C | 1,883 MHz |
| probe, plain 262,144, slow run | 109.4 W | 116.9 W | 65 C | 1,893 MHz |
| probe, plain 262,144, repeat | 159.2 W | 167.4 W | 72 C | 1,866 MHz |
| probe, plain 131,072 | 160.0 W | 167.9 W | 73 C | 1,867 MHz |
| depth, head n-max 1 | 162.5 W | 170.1 W | 74 C | 1,799 MHz |
| depth, head off | 166.1 W | 169.9 W | 74 C | 1,797 MHz |
| depth, plain 262,144 | 164.2 W | 170.7 W | 74 C | 1,774 MHz |
| probe, release, plain 131,072 | 160.5 W | 170.5 W | 72 C | 1,847 MHz |
| probe, release, head off | 161.4 W | 170.2 W | 72 C | 1,832 MHz |
| probe, release, head n-max 1 | 158.2 W | 170.1 W | 72 C | 1,833 MHz |
| probe, release, plain 262,144 | 162.0 W | 170.2 W | 72 C | 1,847 MHz |
| probe, release, head off, switch unset | 161.3 W | 170.3 W | 72 C | 1,848 MHz |
| probe, release, head n-max 1, switch unset | 157.9 W | 169.7 W | 72 C | 1,840 MHz |
| depth, release, head n-max 1 | 161.2 W | 170.5 W | 74 C | 1,841 MHz |
| depth, release, head off | 165.1 W | 170.1 W | 74 C | 1,834 MHz |
| vision, installer, plain 262,144 | 140.9 W | 148.2 W | 69 C | 1,893 MHz |
| vision, release, plain 262,144 | 158.1 W | 169.7 W | 72 C | 1,865 MHz |
| vision, release, head n-max 1, 65,536 | 156.9 W | 170.0 W | 72 C | 1,842 MHz |

Head-off steps sit near the 170 W limit; on the installer's bundle the head arms draw about 13 to 15 W less (on the release build only 3 to 4 W less), and the clock settles lower as the temperature rises over the longer depth runs. The pinning arms read the same range (head off 159.7 to 160.8 W mean active, head on 146.3 to 148.1 W). During the first pass the card peaked at 75.5 C in the llama-bench depth run, at its 170 W limit with no thermal limit flag.

## llama-bench lines verbatim

Command shape (the repo's CONTRIBUTING flags), plain file, first pass (2026-11-11), three separate invocations, then the depth run and the thread test:
`llama-bench -m Ternary-Bonsai-2-27B-PTQ1_0.gguf -ngl 99 -fa on -ctk q4_0 -ctv q4_0 -p 512 -n 128 -r 3`. The table prints `fa` as 1. The ± is llama-bench's spread over its 3 repeats. The `load_backend` path lines are left out.

```text
(invocation 1)
| model                          |       size |     params | backend    | ngl | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           pp512 |       563.73 ± 16.78 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           tg128 |         36.98 ± 0.06 |
build: de947abc3 (10759)

(invocation 2)
| model                          |       size |     params | backend    | ngl | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           pp512 |       560.72 ± 15.93 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           tg128 |         37.12 ± 0.37 |
build: de947abc3 (10759)

(invocation 3)
| model                          |       size |     params | backend    | ngl | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           pp512 |       558.26 ± 19.16 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           tg128 |         36.83 ± 0.30 |
build: de947abc3 (10759)

(depth run, same flags plus -d 0,16384,65536,131072; 12 min 39 s)
| model                          |       size |     params | backend    | ngl | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           pp512 |       561.14 ± 17.60 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           tg128 |         37.11 ± 0.34 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |  pp512 @ d16384 |        466.23 ± 8.31 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |  tg128 @ d16384 |         29.53 ± 0.17 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |  pp512 @ d65536 |        308.88 ± 3.43 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |  tg128 @ d65536 |         17.86 ± 0.07 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 | pp512 @ d131072 |        206.50 ± 7.06 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 | tg128 @ d131072 |         11.62 ± 0.02 |
build: de947abc3 (10759)

(thread test, -p 0 -n 128 -r 3 -t 1,2,4,8)
| model                          |       size |     params | backend    | ngl | threads | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | ------: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |       1 |   q4_0 |   q4_0 |   1 |           tg128 |         36.18 ± 0.33 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |       2 |   q4_0 |   q4_0 |   1 |           tg128 |         36.80 ± 0.09 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |       4 |   q4_0 |   q4_0 |   1 |           tg128 |         36.72 ± 0.20 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |       8 |   q4_0 |   q4_0 |   1 |           tg128 |         37.11 ± 0.17 |
build: de947abc3 (10759)
```

Mean of the three tg128 invocations (calculated): 36.98 tok/s; pp512: 560.9 tok/s. Decode at depth as a share of the fresh figure: 80% at 16384, 48% at 65536 and 31% at 131072; prefill 83%, 55% and 37%. The speed head was not used in llama-bench.

Third pass (2026-11-14 at 22:51), plain file, the same command and flags, the installer's bundle and then the PrismML release in the same window (the `build:` line names the binary):

```text
(installer's bundle)
| model                          |       size |     params | backend    | ngl | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           pp512 |       560.88 ± 16.80 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           tg128 |         37.12 ± 0.21 |
build: de947abc3 (10759)

(PrismML release prism-b12128-d7b2896)
| model                          |       size |     params | backend    | ngl | type_k | type_v |  fa |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -----: | -----: | --: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           pp512 |        583.14 ± 2.18 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.53 GiB |    26.90 B | CUDA       |  99 |   q4_0 |   q4_0 |   1 |           tg128 |         42.85 ± 0.33 |
build: d7b289683 (12128)
```

## Installer speed check (first pass)

The installer's own speed check (`bench-bonsai`, release `v1.2-windows`, 2026-11-11), one prompt and the installer's fitted context, so a different method from the tables above:

```text
Bonsai 2 27B, Windows speed check (2026-11-11 16:19, v1.2-windows)
card: NVIDIA GeForce RTX 3060, 12288 MiB, driver 581.80
system: Microsoft Windows 10 Pro 10.0.19045, Intel(R) Xeon(R) CPU           E5606  @ 2.13GHz, 50 GB RAM
build: bonsai2-small-gpu-windows-x64-cuda12.4-sm86-sm89-ff41412, tier 12gb, file Ternary-Bonsai-2-27B-PTQ1_0-mtp.gguf
vram in use before start (desktop and other apps): 695 MiB
context: 110336 with the head, 197888 without
vram with the head: 10714 MiB loaded, 10731 MiB after the runs
speed with the head (mtp n-max 1): 40.1 / 40.0 / 39.7 tok/s, median 40.0 tok/s, draft 699/831 accepted
speed without the head: 35.6 / 35.4 / 35.6 tok/s, median 35.6 tok/s
head gain: +12%
same greedy text with and without the head: yes
```

## Not run

Draft n-max 3, the env knob on its own on the installer's bundle (on the release build it was run set and unset, in two arms and the identity pair), a change to the CUDA Sysmem Fallback Policy, WSL2, a source build or the repo's `bonsai2` and `pr-ptq1-mmv` branches (the two builds I ran are the installer's bundle and PrismML's release `prism-b12128`), the release build on any other machine, the release build in the cold-load, pinning and power plan tests, the full 262,144 window filled to the end (the installer build's deepest point is 242,577 tokens, and the release build was not run past 120,655), the identity check beyond three greedy prompts, vision beyond one chart and the head with the projector at 131,072 or 262,144, and pinning with the default 8 threads. Nothing here tests answer quality: every figure is speed, fit or text identity. Draft n-max 2 ran in one window only.

## Reproduce

1. Run the Windows installer for the 12gb tier into a fresh folder (the model files are in its `models` folder); check both files against the sizes and hashes in "Build and files".
2. Start `llama-server` with either line in "Build and files" (the other arms are the one-flag changes listed there).
3. `python graft/probe.py http://127.0.0.1:8765` for the A/B table; stop the server and repeat with the `--spec-*` flags removed. Note the idle `memory.used` before each start and the free MiB after load.
4. For depth, `graft/tools/deep_probe.py` with any plain-text file of about 180 KB (the repo does not ship one; mine was 181,385 bytes) against the same server; it prints the real token count of each prompt, and `--runs 0` prints only that count.
5. For identity, `python graft/tools/verify_identity.py run http://127.0.0.1:8765 <out> off` against the head-off server, `... on` against the head-on server, then `... compare <out> off on`.
6. For the release build, download the Windows CUDA 12.4 zip and its CUDA runtime zip from the PrismML release `prism-b12128-d7b2896` (github.com/PrismML-Eng/llama.cpp/releases), extract both into one folder, and repeat steps 2 to 5 with that folder's `llama-server` (and its `llama-bench`); run each MTP-file line with `GGML_CUDA_BATCH_INVARIANT=1` set and unset to see what the switch does to speed and to the identity check.
7. For vision, add `--mmproj Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf` (from `prism-ml/Ternary-Bonsai-2-27B-gguf`) to a serve line, send the repo's `prompts/vision_check.png` with the prompt and settings in the Vision section, and note the time of the first request; identical later requests reuse the cache.
