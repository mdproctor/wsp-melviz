# Design Journal — issue-22-server-data-providers

## 2026-09-25 — Architecture design complete

Designed holistic server-side data provider architecture. Prometheus is
the first provider; the design accommodates any backend (InfluxDB, ES, SQL)
via the existing `DataProvider` CDI SPI.

**Key decisions:**
- Extend `DataSetLookup` with `QueryResult` (data + `remainingOps`) instead
  of capability pre-negotiation — providers return what they couldn't handle
- Reuse existing `FilterOp` + `TIME_FRAME` for time ranges — no new op type
- One Maven module per provider, depending only on `data/` core SPI
- `DataQueryException` + `ExceptionMapper` for structured error handling

**Research:** Parallel agents researched Grafana, Metabase, Superset, Cube.js,
Arrow Flight, and TSDB query abstractions. Synthesis: one query model + many
translators + unified response format is the industry consensus.

**Spec:** 3-round standard review, 14 issues raised and resolved.
**Plan:** 3 batches, 7 tasks — ready for execution.
