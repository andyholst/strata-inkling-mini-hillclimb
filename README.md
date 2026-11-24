# Strata hill-climb: Inkling-2-Mini SQ-10, SQ-15, and SQ-20 on an RTX 4090 Laptop

This repo is the log of one speed hill-climb for the Inkling-2-Mini packs. The method is
[0xSero's inference-optimisation skill](https://github.com/sybil-solutions/inference-optimisation-skill)
(`fda968e7fc1996aa23c67139c755fe541c9d4be0`): measure the machine, change one thing,
compare it with a control, and keep a number only when a long prompt still retrieves.

The Qwen3.8-Flash-Next IQ3_XXS log, including the Hermes NV16 preset, is a separate repo:
[andyholst/strata-qwen-flash-hillclimb](https://github.com/andyholst/strata-qwen-flash-hillclimb).

No API keys, tokens, or account files belong in this repo. The server key stays in
the environment of the process that calls the API.

## Machine and models

| | |
|---|---|
| GPU | NVIDIA GeForce RTX 4090 Laptop, 16 GiB, sm 8.9, driver 615.78.08, 175 W cap |
| CPU | Intel i9-13980HX, governor `powersave`, 62 GiB RAM |
| Engine | Strata 0.1.67 CUDA 13, `strata-inkling` |
| Endpoint | `http://127.0.0.1:8080/v1` |
| Context served | 131072 tokens |
| Packs | SQ-10, SQ-15, SQ-20 under the Inkling-2-Mini family |

UD-IQ3_XXS is an incomplete download and is not in this log.

Inkling does not use the Qwen prefill switch `STRATA_NV_CHUNK`. The engine starts at
prefill chunk 512 and the flag that changes it is `--chunk N`.

Measured machine ceilings (VRAM read 563.45 GB/s, pinned host-to-device 12.3 GB/s,
sustained FP16 GEMM 86.49 TFLOPS) are in the Qwen repo's `hardware/` directory. They
were taken on this same machine with the server stopped. This repo does not repeat that run.

## The loop

1. Start the existing 128K launch preset. Record the engine line it actually printed.
2. Run the harness against that live server. Token counts come from the server.
3. Change exactly one flag. Here that flag is `--chunk`.
4. Keep the change only if the long-prompt speed clears the control and the needle still matches.
5. Leave the original preset file untouched. A kept change would be a new preset, same as Hermes NV16.

## Reproduce a measured preset

`STRATA_API_KEY` must already be exported. The command does not print the key.

```bash
cd ~/.local/share/strata/components/control/0.1.67
python3.12 strata.py launch \
  --preset "SQ-10 128K" \
  --api-key-env STRATA_API_KEY \
  --no-tui --yes --output json
```

From this repo, with the same variable exported:

```bash
python3 scripts/bench.py run configs/sq-10-128k-ctl-a.json --long 150000 --skip-cN --reps 1
```

`--long 150000` is the word-budget that this tokenizer counts as 117169 tokens.
`--long 185000` is the Qwen document. On Inkling that shape is 144469 tokens and the
server rejects it, because 131072 tokens then have no room left to answer.
The three `REQ_FAIL` rows in `ledger.jsonl` are that rejection.

## Files

| Path | What it is |
|---|---|
| `PLAN.md` | What was run, what was kept, what was not tried |
| `configs/preset-sq-10-128k.json` | Public copy of the SQ-10 128K launch preset |
| `configs/preset-sq-15-128k.json` | Public copy of the SQ-15 128K launch preset |
| `configs/preset-sq-20-128k.json` | Public copy of the SQ-20 128K launch preset |
| `configs/sq-*.json` | Harness configs. They name `api_key_env` and do not contain the secret |
| `ledger.jsonl` | One JSON object per run |
| `scripts/bench.py` | Upstream harness plus bearer auth from the environment |
| `NOTICE.md` | Where the script came from |

## Presets

The launch presets on this machine are the existing ones: `SQ-10 128K`, `SQ-15 128K`,
and `SQ-20 128K`. Each public file has the same fields as `preset-hermes-nv16.json`
in the Qwen repo: name, profile, served id, context, the engine arguments the process
actually used, and the engine's own chunk line. None of them contains an API key.

`--chunk 2048` on SQ-10, and `--chunk 1024` on SQ-15 and SQ-20, did not beat the
automatic chunk of 512. Those preset files were not copied and were not edited.
The 117169-token needle matched 3/3 on every measured row. The raw rows are
`ledger.jsonl`. Prompt tok/s is `prompt_tokens / time-to-first-token`. Decode
tok/s is the harness figure, `(completion_tokens - 1)` from the first streamed
token to the end of the stream. Each speed cell below is one sample.

## SQ-10 (2026-11-24)

Preset `SQ-10 128K`, chunk 512, `--vram-experts-gb 8.60`, DSpark. The candidate
was `--chunk 2048` only. It was not kept.

| Run | Short decode | Prefill at 117169 | TTFT | Decode at 117169 | Retrieval |
|---|---:|---:|---:|---:|---|
| Control, chunk 512 | 122.60 tok/s | 1335.5 tok/s | 87.7 s | 9.58 tok/s | 3/3 |
| Rejected, `--chunk 2048` | 120.43 tok/s | 1336.8 tok/s | 87.7 s | 8.43 tok/s | 3/3 |

Host RSS peak was 3.48 GiB on the control and 3.65 GiB on the candidate. A
144469-token document (the Qwen `--long 185000` shape) was rejected with HTTP 400
before this pair. That failed row still recorded a short decode of 122.46 tok/s.
Short TTFT on the control was 91.3 ms.

## SQ-15 (2026-11-24)

Preset `SQ-15 128K`, chunk 512, `--vram-experts-gb 11.49`, DSpark. KV was 3072
tokens in VRAM and 128000 in host memory. The candidate was `--chunk 1024` only.
It was not kept.

| Run | Short decode | Prefill at 117169 | TTFT | Decode at 117169 | Retrieval |
|---|---:|---:|---:|---:|---|
| Control, chunk 512 | 98.72 tok/s | 837.4 tok/s | 139.9 s | 3.17 tok/s | 3/3 |
| Rejected, `--chunk 1024` | 95.80 tok/s | 833.8 tok/s | 140.5 s | 3.21 tok/s | 3/3 |

Host RSS peak on the control was 5.9 GiB. The candidate row recorded 0 because
that watchdog still matched only `strata --serve`, and this binary is
`strata-inkling`. The rejected 144469-token prompt recorded a short decode of
98.60 tok/s before the HTTP 400. Short TTFT on the control was 97.3 ms.

## SQ-20 (2026-11-24)

Preset `SQ-20 128K`, chunk 512, `--vram-experts-gb 10.23 --ram-experts-gb 5.45`,
DSpark. Cold experts stay in pinned host memory. KV was 11520 tokens in VRAM and
119552 in host memory. The candidate was `--chunk 1024` only. It was not kept.
Prefill got slower.

| Run | Short decode | Prefill at 117169 | TTFT | Decode at 117169 | Retrieval |
|---|---:|---:|---:|---:|---|
| Control, chunk 512 | 51.64 tok/s | 489.2 tok/s | 239.5 s | 3.37 tok/s | 3/3 |
| Rejected, `--chunk 1024` | 54.24 tok/s | 465.4 tok/s | 251.7 s | 3.16 tok/s | 3/3 |

Host RSS was recorded as 0 on both rows, for the same watchdog mismatch. The
rejected 144469-token prompt recorded a short decode of 52.59 tok/s before the
HTTP 400. Short TTFT on the control was 633.7 ms.
