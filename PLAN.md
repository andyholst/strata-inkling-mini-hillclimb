# Plan

One machine, one model at a time, one change at a time. Failures stay in this file.

## Target

- Models: Inkling-2-Mini SQ-10, SQ-15, and SQ-20, served as `inkling-2-mini-sq-10`,
  `inkling-2-mini-sq-15`, and `inkling-2-mini-sq-20`.
- Engine: Strata 0.1.67 CUDA 13, binary `strata-inkling`
  `0.1.67-linux-x86_64-cuda-cuda13`.
- Launch presets: `SQ-10 128K`, `SQ-15 128K`, `SQ-20 128K`. Context 131072.
  `http://127.0.0.1:8080`.
- The launch preset files store `drafting: null` and `extra: {}`. The engine still
  starts `--draft dspark --draft-beam 1 --draft-adapt` from its own Inkling defaults.
- Quality bar: the quant and KV layout the preset already loads. A KV format change
  is lossy and has to be labelled. None has been tried.
- Metric: one short single-stream decode, then one 117169-token document with three
  codes. Retrieval has to stay 3/3.
- Concurrency under test: 1. `--skip-cN` is set because one 128K stream fills the card.

## Why the long prompt is `--long 150000`

The harness sizes a document by words, and the server reports the real token count.
On these Inkling tokenizers, `--long 185000` (the Qwen 121530-token shape) comes back
as 144469 tokens. The server answers HTTP 400: the prompt leaves no room to answer
inside 131072. `--long 150000` comes back as 117169 tokens and leaves room for the
256-token reply. Those three 400s are the first three ledger rows. They are not speeds.

## SQ-10

Engine, preset unchanged: `--max-context 131072 --vram-experts-gb 8.60 --draft dspark
--draft-beam 1 --draft-adapt`. Log: prefill chunk 512, 5342 MiB free after the experts,
KV 82176 VRAM tokens plus 48896 host tokens, 946 MiB free after DSpark (588 MiB).

| Run | Chunk | Short c1 | Prefill | TTFT | Decode at 117169 | Retrieval | Kept? |
|---|---:|---:|---:|---:|---:|---|---|
| sq-10-128k-ctl-a | 512 auto | 122.60 | 1335.5 tok/s | 87.7 s | 9.58 tok/s | 3/3 | control |
| sq-10-128k-chunk2048 | `--chunk 2048` | 120.43 | 1336.8 tok/s | 87.7 s | 8.43 tok/s | 3/3 | no |

Prefill did not move. Long decode got slower. The preset file `SQ-10 128K` is unchanged.
`STRATA_NV_CHUNK` was not set. That variable is the Qwen engine's chunk switch, and
this process is `strata-inkling`.

The short decode is about 123 tok/s. The same model at 117169 tokens is about 10 tok/s.
Most of the KV at that length is mapped host memory, and the link on this laptop reads
about 12.3 GB/s. A bigger prefill chunk does not put that KV back on the card.

## SQ-15

Engine, preset unchanged: `--max-context 131072 --vram-experts-gb 11.49 --draft dspark
--draft-beam 1 --draft-adapt`. Log: prefill chunk 512, 1644 MiB free after the experts,
KV 3072 VRAM tokens plus 128000 host tokens, 248 MiB free after DSpark.

| Run | Chunk | Short c1 | Prefill | TTFT | Decode at 117169 | Retrieval | Kept? |
|---|---:|---:|---:|---:|---:|---|---|
| sq-15-128k-ctl-a | 512 auto | 98.72 | 837.4 tok/s | 139.9 s | 3.17 tok/s | 3/3 | control |
| sq-15-128k-chunk1024 | `--chunk 1024` | 95.80 | 833.8 tok/s | 140.5 s | 3.21 tok/s | 3/3 | no |

Prefill and long decode both sat on the control. The preset file `SQ-15 128K` is unchanged.

## SQ-20

Engine from the 0.1.67 start of `SQ-20 128K`: `--max-context 131072
--vram-experts-gb 10.23 --ram-experts-gb 5.45 --draft dspark --draft-beam 1
--draft-adapt`. Log: prefill chunk 512, 2042 MiB free after the experts, KV 11520 VRAM
tokens plus 119552 host tokens, 356 MiB free after DSpark. Cold experts sit in a pinned
host arena and the card reads them itself.

| Run | Chunk | Short c1 | Prefill | TTFT | Decode at 117169 | Retrieval | Kept? |
|---|---:|---:|---:|---:|---:|---|---|
| sq-20-128k-ctl-a | 512 auto | 51.64 | 489.2 tok/s | 239.5 s | 3.37 tok/s | 3/3 | control |
| sq-20-128k-chunk1024 | `--chunk 1024` | 54.24 | 465.4 tok/s | 251.7 s | 3.16 tok/s | 3/3 | no |

Prefill got slower and long decode did not move. The preset file `SQ-20 128K` is unchanged.

`mem_peak_gib` is 0 on the SQ-15 candidate and on both SQ-20 rows. The watchdog that wrote those rows still matched only `strata --serve`, and this engine binary is `strata-inkling`. SQ-10 recorded 3.48 and 3.65 GiB, and the SQ-15 control recorded 5.9 GiB. `scripts/mem_engine.sh` in this repo matches both binaries.

## Not done

- A second control after a candidate that already lost the first. All three candidates
  lost on the first control, so there is no control B.
- KV fp8, fp16, k8v8, or k8v4. That changes the numbers. It is the lever most likely to
  move the 3 to 10 tok/s long decode, and it is labelled lossy until someone measures it.
- `strata tune`. Its default score is a short decode and a 4K prompt.
- UD-IQ3_XXS. The file on disk is a partial download.
- CPU governor. It stayed on `powersave`.
