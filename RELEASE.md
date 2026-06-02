# v1.0.2 — Complete local log import + full request logging

## Fixes

- **Import history**: subagent JSONL files (`subagents/*.jsonl`) were not imported — all 101 missing requests now recovered
- **Streaming dedup**: same `msg_id` appears twice in subagent JSONL (input-first, then output); tokens now accumulated with `max()` per field instead of overwrite
- **Zero-token skip**: import previously skipped records with `input=0, output=0` even when `cache_read > 0` — fixed to require all four token fields to be zero before skipping
- **Synthetic records**: `<synthetic>` model entries (internal Claude Code housekeeping) now explicitly skipped during import
- **Proxy logging**: removed `if input_tok or output_tok or cr` guard in `_record()` — all API responses (including streaming, cache-only, errors) are now written to DB
- **Browser cache**: `/dashboard` now returns `Cache-Control: no-store` — stale HTML no longer shown after proxy restart
- **Onboarding sync**: step [4/8] now triggers an immediate sync after registering the daemon, so fresh data is available right after install

## Result

Local JSONL logs → DB coverage: **100%** (was ~0% for subagent requests)
Proxy request capture: **100%** (was ~72%)
