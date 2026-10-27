# Release notes: v0.2.2

Date: 2026-10-27

This is an editorial/restructuring release of the README. No benchmark data,
raw traces, CSVs, or figures were regenerated or changed.

## What's new

- **README restructured around the MTP result.** The page now leads with the
  headline question and answer: enabling MTP more than doubles deep-context
  decode at 64k on an RTX 3060 12GB (14.79 → 38.56 t/s, +160.7%) with
  explicit memory cost (+993 MiB VRAM, ~+1.7 GiB host RAM) and a clear note
  that prefill is unchanged (-1.19%, below the ~4% noise threshold).
- **At-a-glance table.** Added a compact MTP comparison table and a "How much
  extra VRAM does MTP use?" section to make tradeoffs easy to scan.
- **Clearer configuration.** Added a "What hardware, model, and settings were
  used?" table and a "What does MTP improve—and what does it not?" section.
- **Headline findings restored.** Reintroduced the numbered headline findings
  table with evidence labels (MEASURED/REGENERATED) to satisfy repository
  documentation checks.
- **Better traceability.** Added provenance footers to all figures in README
  and fixed the title/anchor to match CI expectations.

## Documentation checks
All `tests/test_docs.py` checks pass (8/8).

## Artifacts
- Package: `dist/local-inference-study-v0.2.2.tar.gz`
- SHA256: `87968abe48f693ad90ee7402dece09c47c67f1ec2c281b51f54b72e794c2e540`

## Notes
- The experiment remains **not reproducible** from this repository alone (the
  original runner and coding harness are private). The tables are regenerated
  from committed traces only.
- All results remain configuration-specific to the documented 29B MTP profile
  on RTX 3060 12GB, 64k context.
