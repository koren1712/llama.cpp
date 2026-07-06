# Slab Cache in the Fitted Working-Set Regime

This note records a follow-up measurement for the disk-backed expert streaming
branch. The question: does the zero-copy slab cache become a net win once the
touched expert working set fits, or does residency still lose to page-cache
competition on a 32 GB RAM machine?

## Setup

- Hardware: i7-8700K, RTX 4080 16 GB, 32 GB DDR4, NVMe
- Model: `Qwen3-Coder-30B-A3B-Instruct-Q4_K_M.gguf`
- Model file size: 18.63 GB
- Run shape: 128 generated tokens, one run each
- Common flags: `-c 4096 -n 128 --no-warmup --temp 0 -no-cnv --simple-io`
- Streaming flags: `-ngl 99 -ot "ffn_.*_exps=CPU"`
- Stock baseline flags: `-ngl 99 --n-cpu-moe 99`
- Streaming env: `LLAMA_EXPERT_STREAMING=1`, `LLAMA_EXPERT_STREAM_POOL_THREADS=12`,
  `LLAMA_EXPERT_STREAM_PREFETCH_SLOTS=16`, `LLAMA_EXPERT_STREAM_CACHE_MB=0`

## Results

| Variant | Decode speed | Streamed bytes | Cache hits | Slab used |
|---|---:|---:|---:|---:|
| Stock `--n-cpu-moe 99` mmap | 8.02 t/s | n/a | n/a | n/a |
| Streaming, no slab | 2.70 t/s | 141.5 GB | 0 | 0 GB |
| Streaming, 8 GiB slab | 8.70 t/s | 8.18 GB | 140,019 | 8.18 GB |
| Streaming, 16 GiB slab | 7.16 t/s | 8.18 GB | 140,019 | 8.18 GB |

## Interpretation

For this fitted working-set case, the slab is a net win. The 8 GiB slab cuts
streamed expert bytes from 141.5 GB to 8.18 GB and slightly beats the stock
`--n-cpu-moe 99` mmap path in this build.

The 8 GiB to 16 GiB comparison is the important caveat. Both slab sizes stream
the same 8.18 GB and report the same hit count, but the 16 GiB slab is slower.
Past working-set coverage, extra residency buys no fewer reads and becomes
deadweight on this 32 GB RAM system.

This result does not generalize a specific slab size. It supports the narrower
rule: choose the smallest residency budget that covers the active expert working
set, and keep explicit guardrails for 32 GB Windows machines.

