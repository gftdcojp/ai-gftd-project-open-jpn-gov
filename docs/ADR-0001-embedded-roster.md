# ADR-0001: Embedded Roster, Proxied Laws

**Status**: Accepted (2026-04-15)

## Decision

- **Roster (ministries/agencies) is embedded as TS** (`worker/src/roster.ts`), not persisted in a DB.
- **Law data is proxied live** to e-Gov 法令API v2 with 1h edge cache, not mirrored.

## Rationale

- The roster changes ~annually (reorganizations, new agencies). A PR is the right change vector — diffable, reviewable, no ops.
- Laws update frequently (amendments, new promulgations). Mirroring would require a sync pipeline + staleness tracking. e-Gov is authoritative; proxy through.
- No DB = zero ops, zero cost, trivially reproducible.

## Consequences

- **Zero state.** Worker is stateless; any replica gives identical answers.
- **Edge cache = 1h.** Law amendments propagate within an hour.
- **Upstream outage = 502.** `getLaw` / `searchLaws` surface `UpstreamError` when e-Gov is down. Roster endpoints remain fully available.
- **Search is client-side filter over the full law list.** e-Gov API v2 returns the full index; we filter by `query` in-process. For exact semantic search, a later ADR will add a vector index.
