# Provenance

`scripts/bench.py` comes from
[sybil-solutions/inference-optimisation-skill](https://github.com/sybil-solutions/inference-optimisation-skill)
commit `fda968e7fc1996aa23c67139c755fe541c9d4be0`, MIT license.

`scripts/bench.py` has one local change: if the config sets `api_key_env`, the harness sends
`Authorization: Bearer` from that environment variable. The value is not stored in this repo.

`scripts/mem_engine.sh` prints the host RSS of `strata-inkling --serve` or `strata --serve`.

Everything else in this repo is the log of one hill-climb on one machine.
The Qwen3.8-Flash-Next log is a separate repo:
[andyholst/strata-qwen-flash-hillclimb](https://github.com/andyholst/strata-qwen-flash-hillclimb).
