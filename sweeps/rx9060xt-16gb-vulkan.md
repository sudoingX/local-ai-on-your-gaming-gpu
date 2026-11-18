# Ternary Bonsai 2 27B, AMD Radeon RX 9060 XT 16GB (native Windows 11, Vulkan beta)

All numbers from one AMD Radeon RX 9060 XT 16GB (16,304 MiB) on native Windows 11 Professionnel 10.0.26200,
13th Gen Intel Core i5-13400F, 32 GB RAM, driver 26.10.1, one machine, 2026-11-18. The card also drives the
desktop: 2,993 MiB in use before the runs.

This is the first Radeon row (the repo had no AMD sweep yet), run on the Windows installer's own Vulkan beta path.

## Build and files

- Installer: the repo's root `install.ps1` (release `v1.2-windows`), run from a local copy, `BONSAI_MODEL=base`.
- Build: `llama-prism-b10772-e8fd209-bin-win-vulkan-x64`, tier 16gb, Vulkan device Vulkan0.
- `Ternary-Bonsai-2-27B-PTQ1_0-mtp.gguf`, 7,012,820,512 bytes + `Ternary-Bonsai-2-27B-mmproj-Q8_0.gguf`, 629,246,976 bytes.

## Results (the installer's own `bench-bonsai.bat` speed check, 2026-11-18 10:12)

- context: **257,024 with the MTP head, 262,144 without** — the full window fits in 16 GB on Vulkan
- VRAM with the head: 15,046 MiB loaded, 14,970 MiB after the runs
- **speed with the head (mtp n-max 2): 50.6 / 51.0 / 51.1 tok/s, median 51.0 tok/s** (draft 879/1308 accepted)
- **speed without the head: 37.0 / 37.1 / 37.3 tok/s, median 37.1 tok/s**
- head gain: **+38%**
- same greedy text with and without the head: yes
- vision check (fresh chat bars 25.0 / 39.8 / 50.1): read right

## Anything odd

- Greedy text differs with and without the head on Vulkan: no batch-invariant mode on this backend, expected per the installer.
- A follow-up manual timing on the long-lived chat server (port 8899) measured ~16 tok/s wall-clock with the same
  model; the 51.0 tok/s figure above is the installer's own bench (`predicted_per_second`, temp 0, top_k 1, seed 42,
  thinking off) run on a fresh server, matching the method of the other rows.

Short line for X:

```
bonsai 2 27b on my amd radeon rx 9060 xt 16gb (windows, vulkan): 51.0 tok/s with the mtp head, 37.1 without, 257k context @sudoingX
```
