# Ternary Bonsai 2 27B (Offrail 27B), RX 7600 8GB (native Windows 11)

All numbers from one AMD Radeon RX 7600 (Navi 33, 32 CU, 8 GB GDDR6, 288 GB/s, PCIe 4.0 x8) on native
Windows 11 Enterprise build 26300, no WSL2, Adrenalin 32.0.32015.7029. CPU is an AMD Ryzen 5 5500
(6 cores / 12 threads, Zen 3, 4.1 GHz boost) on a Gigabyte B450 AORUS ELITE, with 32 GB of DDR4-3200
CL16 in four 8 GB sticks, all running at 3200 MT/s. One machine, 2026-11-18 and 2026-11-19. Single slot,
`-np 1`, q4_0 K/V cache, flash attention on, `-ngl 99`, ubatch 256, head off.

The card also drives the desktop. Idle dedicated VRAM with a browser and a mail client open: **2,471
MiB**. With nothing but the desktop shell: **about 600 to 900 MiB**. That difference matters, see
"Anything odd".

This is a native Windows row on PrismML's Vulkan build. There is no ROCm here and no custom kernel: the
PTQ1_0 mat-vec kernel in this repo is CUDA, so a Radeon runs PrismML's own PTQ1_0 shader.

## Short version

- **Prefill falls off a cliff between a 16,384 and a 32,768 window, and the 8 GB tier's 64K step is on
  the slow side of it.** A 1,500-token prompt prefills at **151 tok/s at `-c 16384`**, then at **13.0,
  12.3 and 11.6 tok/s** at 32,768, 65,536 and 131,072. Three runs at each size, same prompt, same
  flags, only `-c` changed.
- **Decode halves across the same step**: 12.72 tok/s at 16,384 against 6.72 at 32,768 and above
  (greedy, 64 tokens, two runs each; llama-bench `tg128` independently gives 6.72).
- **The KV cache is not why.** At 32,768 the K/V is 576 MiB and the whole footprint about 6,196 MiB
  of the 8,176 MiB the driver reports, which leaves 1.9 GB spare. Nothing is paging and the adapter
  counter barely moves between 32,768 and 131,072 (6.608 to 6.790 GB) while prefill falls 12x. It is
  not memory pressure; it looks like an allocation or kernel-path change above 16,384. Mechanism not
  established, see "Anything odd".
- **131,072 serves and generates**, with load in 5.9 s and 6.6 tok/s decode, so the tier's 64K ceiling
  is not a hard limit on this card. It is just a slow ceiling.
- **The MTP head is slower here**: 4 to 11% below head off on the same prompts with 72 to 82% draft
  acceptance, where the NVIDIA rows gain 23 to 26%. This card runs head off.
- **`-nkvo` (K/V in system RAM) works and is worse**: 131,072 served at 4.61 tok/s against 6.61 with the
  K/V on the card, and `graph splits` goes from 2 to 322.

## Build and files

- Installer: the repo's root `install.ps1`, Radeon path, which picks the Vulkan bundle and the lean MTP
  file. Scripts are SHA256-checked by the installer.
- Build: PrismML `prism-b10772-e8fd209`, Windows x64 Vulkan. `llama-bench` in this bundle reports
  `build: e8fd209c9 (10772)`.
- `Offrail-27B-8G.gguf`, 6,297,658,848 bytes, sha256
  `be2b5ce27bd4bf6125f571e994489a325b711b092315c4b874837e4649fc04ed` (checked against the repo's
  `SHA256SUMS` by the installer). The standard build's lean file is the same size with a different hash
  (`1e33c571a5ce7a9a3e42474d66192923d5a6d77da7fb3a22986dc809522b5685`); both were run, speed is the same.
- Architecture, from the load banner: `n_layer 64` with only 16 full-attention layers, `n_head_kv 4`,
  `n_embd 5120`, head 256. K/V is **18 KiB per token** (288 MiB at 16K, 576 at 32K, 1,152 at 64K, 2,304
  at 128K) plus a context-independent 149.62 MiB recurrent state, 5,395.33 MiB of weights and a 75 to
  115 MiB compute buffer. `n_ctx_train` 262,144.
- Vulkan device as reported: `uma: 0 | fp16: 1 | bf16: 1 | fp4: 0 | warp size: 64 | shared memory:
  32768 | int dot: 1 | matrix cores: KHR_coopmat`.
- Server bind: `0.0.0.0` with an API key, reached over a tailnet.

The serve line as run (model path removed), which is `serve/8gb.sh` with the window and the port
changed:

```text
llama-server -m Offrail-27B-8G.gguf -c 131072 -ngl 99 -fa on -np 1 -ctk q4_0 -ctv q4_0 --jinja --alias offrail-27b --host 0.0.0.0 --port 8899 --device Vulkan0 --fit off -ub 256 --reasoning-effort medium --temp 1.0 --top-p 0.95 --top-k 20
```

`--fit off` is load-bearing. Left at its default, llama.cpp rewrites the requested context to whatever
it decides fits, and a sweep that forgets this measures the same fallback repeatedly while printing
different requested values.

## Decode and prefill by window

Prefill: a 1,500-token prompt (`cache_prompt: false`, so every request really prefills), `n_predict 1`,
three runs per size, `/completion` timings. Decode: greedy, 64 tokens, two runs per size, same prompt.
Only `-c` changed between rows. Server restarted for each row and the window confirmed from `/props`.

| context | prefill (3 runs) | decode (2 runs) |
| ---: | ---: | ---: |
| 16,384 | 152.18, 151.17, 151.13 | 12.72, 12.70 |
| 32,768 | 13.04, 13.04, 13.06 | 6.72, 6.72 |
| 65,536 | 12.28, 12.25, 12.28 | 6.73¹ |
| 131,072 | 11.56, 11.55, 11.59 | 6.61¹ |

¹ Those two came from an earlier pass with a different prompt; `llama-bench tg128` on the same build
gives 6.72 ± 0.00, which matches the 32,768 row exactly. Everything else in the table is one method.

Spread is under 1% at every size, so the step is the configuration and not run-to-run noise. With a
desktop holding 2,471 MiB on the card, 16,384 measured 8.32 tok/s decode instead of 12.72, and a
7,050-token prompt took about 9 minutes to prefill at 131,072.

What this means in practice: at 16,384 a 1,500-token prompt is ready in 10 seconds; at 64K it is two
minutes of silence before the first token. Chat is unaffected because llama.cpp reuses the prompt cache
and only prefills the new turn, but anything that stuffs a document into the context pays the 12x.

Load time 5.2 to 7.1 s at every window from 64K up, no clamping, no out-of-memory, no failed request.

## Served VRAM

Whole adapter, `\GPU Adapter Memory(*)\Dedicated Usage`, read with the server up and nothing else on the
card. The process column is the server's own working set.

| context | adapter dedicated | server working set |
| ---: | ---: | ---: |
| 32,768 | 6.608 GB | 6.072 GB |
| 65,536 | 6.656 GB | 6.621 GB |
| 131,072 | 6.790 GB | not read |

The adapter counter moves 182 MB across a 1.7 GiB change in K/V, so on this driver it is not tracking
the cache. The server's allocation banner is the trustworthy figure: 5,395.33 MiB weights + K/V +
149.62 MiB recurrent state + 75 to 115 MiB compute. At 131,072 that is about 7,964 MiB of 8,176, so
eighty percent of the card is in use before the desktop asks for anything.

## llama-bench lines verbatim

Command as run, the repo's CONTRIBUTING flags plus `-fitt`/`-fitc` (this bundle has no `--ctx-size`, so
without them llama-bench sizes the K/V to the model's 262,144 training ceiling and pages):

`llama-bench -m Offrail-27B-8G.gguf -ngl 99 -fa 1 -ctk q4_0 -ctv q4_0 -p 512 -n 128 -r 3 -ub 256 -dev Vulkan0 -fitt 512 -fitc 131072 -o md`

```text
| model                          |       size |     params | backend    | ngl | n_ubatch | type_k | type_v |  fa | dev          |       fitt |        fitc |            test |                  t/s |
| ------------------------------ | ---------: | ---------: | ---------- | --: | -------: | -----: | -----: | --: | ------------ | ---------: | ----------: | --------------: | -------------------: |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.85 GiB |    27.32 B | Vulkan     |  99 |      256 |   q4_0 |   q4_0 |   1 | Vulkan0      |        512 |      131072 |           pp512 |        159.66 ± 0.39 |
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.85 GiB |    27.32 B | Vulkan     |  99 |      256 |   q4_0 |   q4_0 |   1 | Vulkan0      |        512 |      131072 |           tg128 |          6.72 ± 0.00 |

build: e8fd209c9 (10772)
```

The same command without `-fitt`/`-fitc`, i.e. the plain CONTRIBUTING line, sizes the context to
262,144, allocates 16x too much K/V for an 8 GB card and pages:

```text
| qwen35 27B PTQ1_0 - 1.75 bpw ternary (group 128) |   5.85 GiB |    27.32 B | Vulkan     |  99 |   q4_0 |   q4_0 |   1 |           tg128 |          5.54 |
```

`pp512` at 159.66 agrees with the 151 tok/s the served path gives at 16,384, which is the fast side of
the step: `-fitt 512` chose a context that fits the card with margin, not the 131,072 in `-fitc`. That
`-fitc` value is a floor for the fitter, not a guarantee of the size it ends up with. Anyone comparing
these numbers with a served 64K run should expect the served path to be 12x slower on prefill, because
the two are on different sides of the step.

## Anything odd

- **The step between 16,384 and 32,768 is the thing to look at.** Same binary, same flags, same prompt,
  same card with the same ~600 MB of desktop on it, and prefill drops from 151 to 13 while decode drops
  from 12.72 to 6.72. The K/V at 32,768 is 576 MiB and the footprint 6,196 MiB, which fits with 1.9 GB
  to spare, so this is not the card running out. Both axes fall together and then stay flat to 131,072,
  which looks like a change of allocation or attention path rather than a slope. I could not establish
  which. `fp4: 0` says there is no ternary fast path here, but that is constant across the step and so
  cannot explain it. This is the number I would most like a second Radeon owner to reproduce.
- **`think` eats the token budget.** With `--reasoning-effort medium`, `max_tokens` of 120 returns an
  empty `content` with about 1,100 reasoning tokens. Budget 700 and up.
- **`AdapterRAM` lies**: 4,294,967,040 B, the 32-bit wraparound. The real size is
  `HardwareInformation.qwMemorySize` = 8,573,157,376 B (8,176 MiB). Free-VRAM decisions on this card
  need the registry value, not WMI.
- **Windows writes inbound Block rules for `llama-server.exe`** the first time it binds, and a block
  beats an allow. Reaching it from another machine needs the block rules removed and an explicit allow
  added.
- **A server started from a console dies with `0xC000013A`** when that window closes, which reads like a
  crash and is not one. Launch it detached or from a task.
- **Vision is not available with these weights.** `sudoingx/Offrail-27B-Uncensored` ships two GGUFs and
  no projector; `/props` reports `modalities {vision:false, video:false, audio:false}` and an image part
  returns `500 image input is not supported`.

## Not run

`graft/probe.py` (the numbers above are llama-bench plus `/completion` timings, three runs at each
size), the `-d 0,16384,65536,131072` depth curve, `llama-bench` on the standard build, any other build
than the installer's, ROCm, WSL2, and power or temperature logging (the Windows counters did not expose
board power for this card).

## Reproduce

1. Run the Windows installer on the Radeon path, take the uncensored build when it asks.
2. Start `llama-server` with the line above, changing only `-c`. Confirm the window from `/props`
   rather than trusting the flag, and keep the desktop to a known state: on an 8 GB card the shell's own
   memory changes the answer.
3. Prefill: `POST /completion` with a prompt of known size (`/tokenize` it first), `n_predict: 1`,
   `cache_prompt: false`, three times, read `timings.prompt_per_second`.
4. Decode: same endpoint, `n_predict: 64`, `temperature: 0`, read `timings.predicted_per_second`.
5. For llama-bench, pass `-fitt 512 -fitc 131072` or it will size the K/V to 262,144 and give you a
   paging number.
