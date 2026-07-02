# Design Journal — issue-21-data-module-backend

## §1 Data Module Architecture (2026-07-02)

**Decision: two modules mirroring layout/layout-sqlite split.**

`backend/data/` holds the SPI (`DataProvider` interface), REST resource, relay proxy, and all DTOs. No JDBC dependency — a deployer who only wants relay includes this module alone. `backend/data-sql/` implements the SPI using Quarkus Agroal named datasources with full SQL push-down.

**Key design choices:**

- **Two endpoints, not one.** `POST /api/dataset/fetch` is the relay proxy (forwards DataRequest to external URLs). `POST /api/dataset/query` dispatches to a DataProvider for server-side operations. Clean separation — relay doesn't need the SPI, query doesn't need HTTP forwarding.

- **Base queries configured server-side, not user-supplied.** Frontend sends `dataSetId: "sales-summary"`. Backend resolves to a registered base query via `casehub.pages.data.sql.queries.<name>.query` config property. SQL injection surface reduced to zero for the base query path.

- **SQL injection prevention at three layers.** Filter values are bind parameters. Column identifiers validated against allowlist from base query metadata. Column names ANSI-quoted in generated SQL.

- **SSRF protection on relay.** Scheme restriction (http/https only), optional host allowlist, private IP block (loopback/site-local/link-local rejected after DNS resolution).

- **Filter expression tree preserved.** The frontend's recursive AND/OR/NOT tree with typed leaves (numeric, string, date, unresolved) is modelled faithfully in Java sealed interfaces. SqlQueryBuilder walks it recursively to produce parameterized WHERE clauses.

**What's deferred:** Frontend `RemoteDataService` integration (#89), server-side caching (#90), named datasource resolution (MVP uses default datasource only).
