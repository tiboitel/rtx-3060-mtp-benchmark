# RTX 3060 12GB MTP Benchmark: +160.7% LLM Decode Speedup at 64k Context

**Does multi-token prediction (MTP) make local LLM inference faster on an NVIDIA RTX 3060 12GB?** In one measured long-context configuration, yes: enabling MTP increased deep-context decode throughput from **14.79 to 38.56 tokens/s (+160.7%)** when serving a 29B-class mixture-of-experts model at a 64k context limit.

The improvement came with a memory trade-off: MTP increased observed peak GPU memory usage by **993 MiB** (from 10,595 MiB to 11,588 MiB). Prefill throughput changed from 645.1 to 637.4 tokens/s (−1.19%), a difference below this study's approximately 4% operational noise threshold. Peak host RAM also increased by roughly **+1.7 GiB** between observed maxima.

This benchmark documents a **specific configuration**, not a universal MTP speedup for every RTX 3060, LLM, or inference runtime.

## MTP benchmark results at a glance

| Metric | MTP off | MTP on | Change |
| --- | --- | --- | --- |
| Deep-context decode throughput | 14.79 tokens/s | 38.56 tokens/s | **+160.7%** |
| Prefill throughput | 645.1 tokens/s | 637.4 tokens/s | −1.19% *(< 4% noise)* |
| Observed peak VRAM | 10,595 MiB | 11,588 MiB | **+993 MiB** |
| Observed peak host RAM | Baseline | Higher | **~+1.7 GiB** *(difference of maxima)* |
| Per-request draft-token acceptance | — | 0.838 (SD 0.070) | — |

**Test setup:** NVIDIA RTX 3060 12GB, Intel i7-9700K, 32GB system RAM, `Qwen3.6-29B-REAP-Opus-Reasoning-Distill-MTP` (Q4_K_M, ~29B-class MoE), 65,536-token context limit, ~57.9k-token prompts, and `llama.cpp` build `bcb85fc`. MTP configured with `--spec-type draft-mtp --spec-draft-n-max 2`. See [full configuration details](#what-hardware-model-and-settings-were-used) below.

![MTP on/off at 64k](figures/fig_mtp_ab.png)

*Data: [mtp_ab.csv](results/mtp_ab.csv) · [mtp_comparison.csv](results/mtp_comparison.csv) · Raw traces: [data/raw/mtp/](data/raw/mtp/) · Methodology: [METHODOLOGY.md](METHODOLOGY.md). 10 kept runs per arm, interleaved off→on→on→off, cold prefill (`cache_n == 0`). Request-level unpaired nominal 95% CI for decode gain [19.19, 28.34] t/s; not paired due to unique nonces per request.*

**[Read the full research article →](ARTICLE.md)**

## Headline findings

| # | Finding | Configuration | Cost / caveat | Evidence |
|---|---|---|---|---|
| 1 | **MTP more than doubles deep-context decode at 64k in the documented shared configuration: 14.79 → 38.56 t/s (+160.7%)**, while prefill changes by −1.19%, below the ~4% operational noise threshold. | Live 29B MTP profile, 10 runs/arm, ~57.9k prompt; identical shared serving settings, different speculative-decoding configuration | **+993 MiB peak VRAM**, ~1.7 GiB peak host RAM (difference of observed maxima); not REPRODUCIBLE (private runner) | **MEASURED + REGENERATED** — [mtp_ab.csv](results/mtp_ab.csv) |
| 2 | **Depth-0 `llama-bench` measures 46.0 t/s, but only ~15 t/s once a long prompt in a 64k-context configuration is in front of it (~3× in this comparison).** Depth-0 is a valid baseline for model-only generation; it does not capture the long-context request path. | Both rows are 28B bench-sibling measurements; the deep row is a server request, not the live 29B profile | Different configs within the sibling; this is a measured gap, not a universal ratio | **MEASURED (sibling)** — [deep_vs_depth0.csv](results/deep_vs_depth0.csv) |
| 3 | **The MoE expert cache is the prefill lever: 308 → 776 t/s (~2.5×) in the short-context comparison.** At 64k, cache-off prefill collapses below 73 t/s, versus ~677–712 t/s with the cache on. | 28B bench sibling, 1k comparison; prompt/sampling not recorded; 64k rows are sibling measurements | Spends spare VRAM; deep decode stays ~14–15 t/s; 64k no-cache is a timeout upper bound | **MEASURED (sibling) + REGENERATED** — [moe_cache.csv](results/moe_cache.csv) |

## How much extra VRAM does MTP use?

MTP adds **993 MiB of peak GPU memory** on this configuration (10,595 MiB → 11,588 MiB). For a 12 GB card with ~12,288 MiB total, this leaves meaningful headroom in this test, but it's a real cost to plan for. Host RAM also increases by roughly 1.7 GiB at peak (measured as difference of observed maxima).

![Tested serving path and memory split](figures/fig_setup.svg)

*Schematic: VRAM is the binding constraint for 64k context on this hardware — the KV cache is VRAM-resident, so host RAM does not scale with context length.*

## What hardware, model, and settings were used?

| Component | Value |
| --- | --- |
| GPU | NVIDIA RTX 3060, **12 GB** (12,288 MiB) |
| CPU / RAM | Intel i7-9700K, 8 cores / 8 threads / 32 GB |
| OS | Arch Linux, kernel 7.2.8 |
| GPU driver / power | 615.78.08; 170 W limit *(post-hoc capture, not the run-time record)* |
| Model | `Qwen3.6-29B-REAP-Opus-Reasoning-Distill-MTP`, Q4_K_M |
| Architecture | Sparse Mixture-of-Experts, ~29B total |
| Context | 64k (`-c 65536`) |
| Expert offload | `--n-cpu-moe 20` |
| MTP off arm | `--spec-type none` |
| MTP on arm | `--spec-type draft-mtp --spec-draft-n-max 2` |
| KV cache | `turbo4` (K) / `turbo3` (V) |
| Expert cache | `--moe-cache on` (both arms) |
| Build | `llama.cpp` (KV-cache fork, `turbo4`/`turbo3`) @ `bcb85fc3ae85efa0f5f392c6c880dfc524923860` |

Full hardware/runtime details: [HARDWARE-AND-RUNTIME.md](HARDWARE-AND-RUNTIME.md).

## What does MTP improve—and what does it not?

- **Decode gains, not prefill.** MTP is most effective during token generation (decode). In this test, prefill was essentially unchanged (−1.19%), below the ~4% operational noise threshold.
- **Long-context benefits matter.** Depth-0 `llama-bench` measures ~46.0 t/s, but only ~15 t/s once a long prompt (~57.9k tokens) is in front of the server in a 64k context configuration (~3× drop). This benchmark measures the real long-context request path, not synthetic depth-0 throughput.
- **Configuration-specific.** Results depend on model architecture, quantization, context length, draft budget (`--spec-draft-n-max`), build, and hardware. Acceptance rate (0.838 here) is content-driven and will vary.
- **Throughput ≠ latency or quality.** Tokens/s measures generation speed; it does not directly predict end-to-end latency for all use cases or model correctness. See coding results below for one quality signal under fixed conditions.

## Other measured findings

| Finding | Configuration | Notes | Evidence |
| --- | --- | --- | --- |
| **Expert cache prefill lever** | 28B bench sibling | 308 → 776 t/s (~2.5×) at short context; at 64k cache-off collapses below 73 t/s vs ~677–712 t/s with cache on. | [moe_cache.csv](results/moe_cache.csv) |
| **Depth-0 vs long-context gap** | 28B bench sibling | Depth-0 ~46.0 t/s vs ~15 t/s in deep long-context configuration (~3×). | [deep_vs_depth0.csv](results/deep_vs_depth0.csv) |
| **ncmoe optimum doesn't transfer** | 29B live profile | `ncmoe=17` OOMs at load, `ncmoe=20` fits. | [ncmoe_probe_29b.csv](results/ncmoe_probe_29b.csv) |
| **End-to-end coding (fixed suite)** | 29B live profile, shipped sampling | 19/33 runs = 57.6% (run-level Wilson 95% CI 40.8–72.8; task-clustered bootstrap 27.3–84.9%). 6/11 tasks passed all 3 runs, 4/11 never passed. 84.0% of individual test methods passed. | [coding_overall.csv](results/coding_overall.csv), [coding_summary.csv](results/coding_summary.csv) |

![Three measured levers](figures/fig_overview.png)

*Only the MTP panel is the live 29B profile; expert cache and depth-0 comparisons use the 28B bench sibling.*

## How reliable and generalizable is this?

**Observed in this configuration (MEASURED + REGENERATED):**
- Magnitudes for 64k context, `ncmoe=20`, MTP with two-token draft budget (`spec-draft-n-max 2`), `turbo4`/`turbo3`, build `bcb85fc`, this prompt family.
- MTP-on: 14.79 → 38.56 t/s decode (+160.7%), prefill −1.19% (below ~4% noise). +993 MiB VRAM, ~+1.7 GiB host RAM.
- 10 kept runs per arm, interleaved, cold prefill per request (unique nonce). Per-request draft acceptance 0.838.

**Not established / hypotheses only:**
- Other GPUs, VRAM tiers, multi-GPU, other stacks (vLLM, TensorRT-LLM, MLX), quantizations, or model families.
- 128k on the live profile (128k figures are bench-sibling estimates).
- Optimal draft count (two tokens not proven optimal).
- Throughput predicts correctness in general.
- Results transfer to unseen coding tasks or different corpora.

Every number links to committed traces/tables. Evidence labels and definitions: [EVIDENCE.md](EVIDENCE.md). Full boundaries: [LIMITATIONS.md](LIMITATIONS.md). Methodology: [METHODOLOGY.md](METHODOLOGY.md).

## Inspect or reproduce the results

The raw traces are committed for audit, but the original benchmark runner and coding harness are **private** — the experiment cannot be independently repeated from this repository alone. The tables are regenerated from committed traces.

```bash
# Audit committed traces and regenerate tables/figures
pip install -r requirements-dev.txt  # scripts + pytest
python scripts/build_results.py --raw data/raw --out results   # raw traces -> results/*.csv
python scripts/make_figures.py                                 # results/*.csv -> figures/*.png
python scripts/check_traceability.py --report results/traceability.txt
python scripts/manifest.py --check
python -m pytest -q
bash scripts/check_hygiene.sh
```

**Reproduce the procedure against running servers:** `scripts/reproduce_mtp_ab.py` publishes the client-side MTP on/off procedure against two already-running servers. Compare directions and intervals, not the published magnitudes. See [REPRODUCTION.md](REPRODUCTION.md) for details.

**Release artifacts:** Version `v0.2.1` is the current evidence release (editorial revision of `v0.2.0`, no data/result changes). Build checksummed artifact with `python scripts/package_release.py --version 0.2.1`. See [RELEASE-NOTES-v0.2.1.md](RELEASE-NOTES-v0.2.1.md).

## Links & resources

| Resource | Where |
| --- | --- |
| **Full article (flagship)** | [ARTICLE.md](ARTICLE.md) |
| Methodology & protocol | [METHODOLOGY.md](METHODOLOGY.md) |
| Hardware, model, serving flags | [HARDWARE-AND-RUNTIME.md](HARDWARE-AND-RUNTIME.md) |
| Evidence labels, matrix, release checklist | [EVIDENCE.md](EVIDENCE.md) |
| Limitations | [LIMITATIONS.md](LIMITATIONS.md) |
| Reproduction & boundaries | [REPRODUCTION.md](REPRODUCTION.md) |
| Prompt provenance | [PROMPT-PROVENANCE.md](PROMPT-PROVENANCE.md) |
| Data hygiene checklist | [DATA-HYGIENE.md](DATA-HYGIENE.md) |
| Traceability map | [data/traceability.md](data/traceability.md) · [results/traceability.txt](results/traceability.txt) |
| Raw traces | [data/raw/](data/raw/) |
| Result tables | [results/](results/) |
| Figures | [figures/](figures/) |
| Citation metadata | [CITATION.cff](CITATION.cff) |
| License | [LICENSE](LICENSE) |

## License

Text and figures: CC BY 4.0. Scripts: MIT. See [LICENSE](LICENSE).
