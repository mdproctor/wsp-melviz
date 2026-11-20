---
title: "Server-Side Data Providers — From SPI to Prometheus Showcase"
date: 2026-09-25
author: mdp
tags: [data-pipeline, prometheus, server-query, typescript, java]
entry_type: note
subtype: diary
---

# Server-Side Data Providers — From SPI to Prometheus Showcase

The Pages data pipeline has supported two query paths since early on:
client-side fetch (URL → extract → filter/group/sort) and server-side
query (`DataSetLookup` → `DataResource` → `DataProvider`). The server
path had a provider SPI with CDI auto-discovery and one implementation
(`SqlDataProvider`). Today we extended it to handle backends that can
only translate *some* operations natively — Prometheus being the first.

## The design problem

SQL can handle every filter, group, and sort we throw at it. Prometheus
can't. It speaks PromQL — label matchers for filters, `sum by (label)`
for aggregation, but no concept of sort. A naive approach would require
the client to know what each backend supports and split the operation
list accordingly. That's capability negotiation, and it pushes
complexity onto every page author.

The alternative: let the provider decide. It translates what it can,
returns the rest as `remainingOps`, and the client applies those
client-side via the existing `applyOps()` pipeline. No capability
negotiation needed — the provider is the single source of truth.

## What we built

**Core SPI changes** — `QueryResult` wraps `DataSetResult` plus
`List<DataSetOp> remainingOps`. `DataProvider.query()` now returns
`QueryResult` instead of raw `DataSetResult`. Existing providers
(`SqlDataProvider`, `NoOpDataProvider`) use `QueryResult.complete(result)`
— zero remaining ops. `DataQueryException` maps provider errors to
structured HTTP responses: `INVALID_QUERY` → 400, `FETCH_FAILED` → 502,
`RESULT_TOO_LARGE` → 413.

**Prometheus module** — `backend/data-prometheus/` depends only on
`backend/data/` plus `java.net.http`. No Agroal, no SQL, no other
provider's types. The `PromQLBuilder` translates `FilterOp` label
matchers (`EQUALS_TO` → `{mode="idle"}`, `LIKE_TO` → regex), `GroupOp`
aggregations (`SUM`/`AVG`/`MIN`/`MAX`/`COUNT` with `Distinct` strategy),
and `TIME_FRAME` to `start`/`end` query parameters. Everything else —
`Or`, `Not`, `Numeric`, `SortOp`, non-Distinct groups — goes to
`remainingOps`.

**TypeScript client** — `resolveOps` extracted from `manager.ts` to a
shared `ops-resolve.ts`. `ServerQueryClient.query()` now returns
`{ dataset, remainingOps }`. The resolver applies remaining ops
client-side: `remainingOps.length > 0 ? applyOps(dataset,
resolveOps(remainingOps, dataset.columns)) : dataset`. Backward
compatible — `remainingOps ?? []` handles older servers.

**Showcase** — a "Prometheus Metrics" page in the Server examples tab
with three line charts (CPU idle, HTTP request rates, memory usage) and
a data table. A `SyntheticMetricsProvider` in the examples server
generates realistic time-series data so the showcase works without
actual Prometheus infrastructure.

## What surprised me

The `serverQuery: true` and `url` fields are mutually exclusive in the
dataset schema — but the `def-to-binding.ts` code checks
`def.serverQuery && def.url`. The schema validation rejects what the
runtime code appears to expect. The showcase initially used both and
hit `CONFIG_MISSING` errors. The fix: `serverQuery: true` datasets get
their endpoint from `providerConfig.serverQuery.endpoint` (configured
at site init time), not from a per-dataset URL.

## What this opens up

The `remainingOps` pattern makes adding new backends mechanical. An
InfluxDB provider, an Elasticsearch provider — each implements
`DataProvider`, translates what it can natively, and the rest
falls through. Page YAML doesn't change when the backing provider
changes. Dataset ID routing (`canHandle`) separates page content
from deployment topology.
