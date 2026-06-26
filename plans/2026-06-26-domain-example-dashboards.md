# Domain-Specific Example Dashboards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add four domain-specific example dashboards (Sales, IoT, People, Clinical) to the gallery, exercising underused platform features.

**Architecture:** Each dashboard is a self-contained pair: `.dash.yaml` (rendered by the gallery) + `.ts` companion (displayed as source code). All data is inline JSON. No infrastructure changes — `generate-samples.js` auto-discovers new dashboards. Each task is independently committable.

**Tech Stack:** YAML dashboards, TypeScript builder API (`@casehubio/ui`, `@casehubio/data`), Playwright (smoke tests)

## Global Constraints

- **Issue:** #14 — every commit must reference it
- **Spec:** `docs/superpowers/specs/2026-06-26-domain-example-dashboards-design.md`
- **selfApply casing:** Always `selfApply` (camelCase) — runtime reads it at `site.ts:487`. Lowercase `selfapply` is silently ignored.
- **YAML-only features:** `${var}` substitution runs in `parsePage()` only. TS companions hardcode default values.
- **TS imports:** `import { ... } from "@casehubio/ui"` for builders, `import { createLookup } from "@casehubio/data"`, `import type { DataSetId, ColumnId } from "@casehubio/data"`
- **navTree pattern:** SIDEBAR/TABS component with `navGroupId` → `navTree.root_items` GROUP with matching `id` → `children: [{page: "Name"}]`
- **No map charts** — geo data loading broken (#35)
- **No accumulate on inline data** — expression pipeline broken for content datasets (#36)
- **No PNG export** — only `csvExport: true` on tables
- **Build verification:** `yarn build:examples` (rebuilds samples.json + copies dashboards)
- **Dev server:** `yarn workspace @casehub/pages-examples run dev` (port 8080, hot reload)

---

### Task 1: Sales Dashboard

**Files:**
- Create: `examples/dashboards/Sales/Sales Dashboard.dash.yaml`
- Create: `examples/dashboards/Sales/Sales Dashboard.ts`

**Produces:** A 4-page sidebar dashboard with 2 inline datasets (~60 + ~15 rows). Exercises: donut, COLUMN_STACKED, BAR horizontal, SELECTOR_LABELS with selfApply, groupByCalendar (MONTH/QUARTER), SMOOTH line, zoom, properties/vars (`${GoalsFunction}`), sortable tables with column expressions.

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/dashboards/Sales/Sales Dashboard.dash.yaml`:

```yaml
properties:
    GoalsFunction: SUM

global:
    displayer:
        chart:
            resizable: true

datasets:
    - uuid: sales_transactions
      content: >-
          [
              [1, "2025-07-15", "EMEA", "Sarah Chen", "CloudSync Pro", "Platform", 42000, 1, "Won"],
              [2, "2025-07-22", "APAC", "James Park", "DevOps Suite", "Platform", 38500, 1, "Won"],
              [3, "2025-08-03", "Americas", "Maria Garcia", "Support Basic", "Support", 2400, 12, "Won"],
              [4, "2025-08-10", "EMEA", "Sarah Chen", "Consulting 5-day", "Services", 15000, 5, "Won"],
              [5, "2025-08-18", "ANZ", "Liam Walsh", "CloudSync Pro", "Platform", 39000, 1, "Lost"],
              [6, "2025-09-02", "Americas", "David Kim", "Support Premium", "Support", 4800, 6, "Won"],
              [7, "2025-09-14", "EMEA", "Anna Müller", "DevOps Suite", "Platform", 41000, 1, "Won"],
              [8, "2025-09-20", "APAC", "James Park", "Migration Workshop", "Services", 22000, 3, "Won"],
              [9, "2025-10-01", "Americas", "Maria Garcia", "CloudSync Pro", "Platform", 45000, 1, "Pending"],
              [10, "2025-10-08", "EMEA", "Thomas Dubois", "Support Basic", "Support", 1800, 9, "Won"],
              [11, "2025-10-15", "ANZ", "Liam Walsh", "Consulting 5-day", "Services", 15000, 5, "Lost"],
              [12, "2025-10-28", "APAC", "Yuki Tanaka", "DevOps Suite", "Platform", 37500, 1, "Won"],
              [13, "2025-11-05", "Americas", "David Kim", "CloudSync Pro", "Platform", 44000, 1, "Won"],
              [14, "2025-11-12", "EMEA", "Sarah Chen", "Support Premium", "Support", 5200, 8, "Won"],
              [15, "2025-11-19", "ANZ", "Liam Walsh", "Migration Workshop", "Services", 18000, 2, "Won"],
              [16, "2025-11-25", "APAC", "James Park", "CloudSync Pro", "Platform", 40000, 1, "Lost"],
              [17, "2025-12-03", "Americas", "Maria Garcia", "Consulting 10-day", "Services", 28000, 10, "Won"],
              [18, "2025-12-10", "EMEA", "Anna Müller", "Support Basic", "Support", 2100, 7, "Won"],
              [19, "2025-12-15", "EMEA", "Thomas Dubois", "DevOps Suite", "Platform", 43000, 1, "Pending"],
              [20, "2025-12-22", "APAC", "Yuki Tanaka", "Support Premium", "Support", 3600, 4, "Won"],
              [21, "2026-01-08", "Americas", "David Kim", "CloudSync Pro", "Platform", 46000, 1, "Won"],
              [22, "2026-01-14", "ANZ", "Liam Walsh", "DevOps Suite", "Platform", 38000, 1, "Won"],
              [23, "2026-01-20", "EMEA", "Sarah Chen", "Migration Workshop", "Services", 20000, 4, "Won"],
              [24, "2026-01-28", "APAC", "James Park", "Support Basic", "Support", 2400, 12, "Lost"],
              [25, "2026-02-04", "Americas", "Maria Garcia", "Consulting 5-day", "Services", 15000, 5, "Won"],
              [26, "2026-02-11", "EMEA", "Anna Müller", "CloudSync Pro", "Platform", 47000, 1, "Won"],
              [27, "2026-02-18", "ANZ", "Liam Walsh", "Support Premium", "Support", 4200, 6, "Won"],
              [28, "2026-02-25", "APAC", "Yuki Tanaka", "DevOps Suite", "Platform", 39500, 1, "Pending"],
              [29, "2026-03-05", "Americas", "David Kim", "Migration Workshop", "Services", 25000, 5, "Won"],
              [30, "2026-03-12", "EMEA", "Thomas Dubois", "Support Basic", "Support", 1500, 5, "Won"],
              [31, "2026-03-18", "EMEA", "Sarah Chen", "CloudSync Pro", "Platform", 48000, 1, "Won"],
              [32, "2026-03-25", "APAC", "James Park", "Consulting 10-day", "Services", 30000, 10, "Won"],
              [33, "2026-04-02", "Americas", "Maria Garcia", "DevOps Suite", "Platform", 41500, 1, "Lost"],
              [34, "2026-04-10", "ANZ", "Liam Walsh", "CloudSync Pro", "Platform", 43000, 1, "Won"],
              [35, "2026-04-15", "EMEA", "Anna Müller", "Support Premium", "Support", 5800, 10, "Won"],
              [36, "2026-04-22", "APAC", "Yuki Tanaka", "Migration Workshop", "Services", 16000, 2, "Won"],
              [37, "2026-05-01", "Americas", "David Kim", "Support Basic", "Support", 2700, 9, "Won"],
              [38, "2026-05-08", "EMEA", "Thomas Dubois", "CloudSync Pro", "Platform", 44500, 1, "Pending"],
              [39, "2026-05-14", "ANZ", "Liam Walsh", "Consulting 5-day", "Services", 15000, 5, "Lost"],
              [40, "2026-05-20", "APAC", "James Park", "DevOps Suite", "Platform", 42000, 1, "Won"],
              [41, "2026-05-28", "Americas", "Maria Garcia", "Support Premium", "Support", 4800, 8, "Won"],
              [42, "2026-06-03", "EMEA", "Sarah Chen", "DevOps Suite", "Platform", 45000, 1, "Won"],
              [43, "2026-06-10", "EMEA", "Anna Müller", "Consulting 5-day", "Services", 15000, 5, "Won"],
              [44, "2026-06-15", "APAC", "Yuki Tanaka", "CloudSync Pro", "Platform", 41000, 1, "Won"],
              [45, "2026-06-20", "Americas", "David Kim", "DevOps Suite", "Platform", 43500, 1, "Won"],
              [46, "2025-08-25", "EMEA", "Thomas Dubois", "CloudSync Pro", "Platform", 40000, 1, "Won"],
              [47, "2025-09-28", "Americas", "Maria Garcia", "DevOps Suite", "Platform", 36000, 1, "Lost"],
              [48, "2025-11-02", "APAC", "Yuki Tanaka", "Support Basic", "Support", 1200, 4, "Won"],
              [49, "2025-12-28", "ANZ", "Liam Walsh", "CloudSync Pro", "Platform", 42500, 1, "Won"],
              [50, "2026-01-30", "EMEA", "Sarah Chen", "Support Basic", "Support", 2100, 7, "Won"],
              [51, "2026-02-28", "Americas", "David Kim", "CloudSync Pro", "Platform", 47500, 1, "Pending"],
              [52, "2026-03-30", "APAC", "James Park", "Support Premium", "Support", 3900, 6, "Won"],
              [53, "2026-04-28", "EMEA", "Anna Müller", "Migration Workshop", "Services", 22000, 4, "Won"],
              [54, "2026-05-30", "ANZ", "Liam Walsh", "DevOps Suite", "Platform", 40000, 1, "Won"],
              [55, "2026-06-25", "Americas", "Maria Garcia", "CloudSync Pro", "Platform", 49000, 1, "Won"],
              [56, "2025-07-30", "APAC", "Yuki Tanaka", "Consulting 5-day", "Services", 15000, 5, "Lost"],
              [57, "2025-09-08", "ANZ", "Liam Walsh", "Support Basic", "Support", 1800, 6, "Won"],
              [58, "2025-10-20", "EMEA", "Thomas Dubois", "Migration Workshop", "Services", 20000, 4, "Won"],
              [59, "2025-12-08", "Americas", "David Kim", "Consulting 10-day", "Services", 32000, 10, "Won"],
              [60, "2026-04-05", "APAC", "James Park", "CloudSync Pro", "Platform", 38500, 1, "Pending"]
          ]
      columns:
          - id: id
            type: NUMBER
          - id: date
            type: DATE
          - id: region
            type: LABEL
          - id: rep
            type: LABEL
          - id: product
            type: LABEL
          - id: category
            type: LABEL
          - id: amount
            type: NUMBER
          - id: quantity
            type: NUMBER
          - id: status
            type: LABEL

    - uuid: pipeline
      content: >-
          [
              ["Acme Corp Migration", "Acme Corp", "Negotiation", 85000, 75, "Sarah Chen", "2026-07-15"],
              ["GlobalTech Platform", "GlobalTech Inc", "Proposal", 120000, 40, "David Kim", "2026-08-01"],
              ["StartupX DevOps", "StartupX", "Qualified", 35000, 25, "James Park", "2026-09-10"],
              ["MegaBank Support", "MegaBank", "Closed", 52000, 95, "Anna Müller", "2026-07-01"],
              ["HealthCo Cloud", "HealthCo", "Prospect", 95000, 10, "Maria Garcia", "2026-10-15"],
              ["RetailPro Consulting", "RetailPro", "Proposal", 28000, 50, "Thomas Dubois", "2026-08-20"],
              ["EduNet Platform", "EduNet", "Negotiation", 67000, 70, "Yuki Tanaka", "2026-07-30"],
              ["LogiMove Suite", "LogiMove", "Qualified", 43000, 30, "Liam Walsh", "2026-09-01"],
              ["DataVault Migration", "DataVault", "Prospect", 110000, 15, "Sarah Chen", "2026-11-01"],
              ["CityGov Support", "CityGov", "Proposal", 38000, 55, "David Kim", "2026-08-15"],
              ["MediaFlow DevOps", "MediaFlow", "Negotiation", 72000, 65, "James Park", "2026-07-25"],
              ["AgriTech Cloud", "AgriTech Co", "Closed", 48000, 90, "Maria Garcia", "2026-07-05"],
              ["FinServ Consulting", "FinServ Ltd", "Qualified", 55000, 20, "Anna Müller", "2026-09-20"],
              ["TravelEx Platform", "TravelEx", "Prospect", 88000, 5, "Yuki Tanaka", "2026-12-01"],
              ["BuildCo Support", "BuildCo", "Proposal", 31000, 45, "Liam Walsh", "2026-08-10"]
          ]
      columns:
          - id: deal
            type: TEXT
          - id: account
            type: TEXT
          - id: stage
            type: LABEL
          - id: value
            type: NUMBER
          - id: probability
            type: NUMBER
          - id: rep
            type: LABEL
          - id: closeDate
            type: DATE

pages:
    - name: index
      components:
          - type: SIDEBAR
            properties:
                navGroupId: SalesNav

    # === Page 1: Overview ===
    - name: Overview
      rows:
          - columns:
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Total Revenue
                            columns:
                                - id: amount
                                  pattern: "$#,###"
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - functions:
                                          - source: amount
                                            function: SUM
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Won Deals
                            columns:
                                - id: amount
                                  pattern: "#"
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - functions:
                                          - source: amount
                                            function: COUNT
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Avg Deal Size
                            columns:
                                - id: amount
                                  pattern: "$#,###"
                            lookup:
                                uuid: sales_transactions
                                group:
                                    - functions:
                                          - source: amount
                                            function: AVERAGE
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Total Deals
                            columns:
                                - id: amount
                                  pattern: "#"
                            lookup:
                                uuid: sales_transactions
                                group:
                                    - functions:
                                          - source: amount
                                            function: COUNT
          - columns:
                - span: 8
                  components:
                      - displayer:
                            type: BARCHART
                            subtype: COLUMN_STACKED
                            general:
                                title: Revenue by Region
                            filter:
                                listening: true
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - columnGroup:
                                          source: region
                                      functions:
                                          - source: region
                                          - source: amount
                                            function: SUM
                - span: 4
                  components:
                      - displayer:
                            type: PIECHART
                            subtype: DONUT
                            general:
                                title: Revenue by Category
                            filter:
                                listening: true
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - columnGroup:
                                          source: category
                                      functions:
                                          - source: category
                                          - source: amount
                                            function: SUM
          - columns:
                - components:
                      - displayer:
                            type: SELECTOR
                            subtype: SELECTOR_LABELS
                            filter:
                                selfApply: true
                                notification: true
                            lookup:
                                uuid: sales_transactions
                                group:
                                    - columnGroup:
                                          source: region
                                      functions:
                                          - source: region
          - columns:
                - span: 6
                  components:
                      - displayer:
                            type: LINECHART
                            general:
                                title: Monthly Revenue
                            filter:
                                listening: true
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - columnGroup:
                                          source: date
                                          strategy: fixedCalendar
                                          unit: MONTH
                                      functions:
                                          - source: date
                                          - source: amount
                                            function: ${GoalsFunction}
                - span: 6
                  components:
                      - displayer:
                            type: BARCHART
                            general:
                                title: Top Reps
                            filter:
                                listening: true
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                sort:
                                    - column: Revenue
                                      order: DESCENDING
                                group:
                                    - columnGroup:
                                          source: rep
                                      functions:
                                          - source: rep
                                          - source: amount
                                            function: SUM
                                            column: Revenue

    # === Page 2: Pipeline ===
    - name: Pipeline
      rows:
          - columns:
                - span: 4
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Pipeline Value
                            columns:
                                - id: value
                                  pattern: "$#,###"
                            lookup:
                                uuid: pipeline
                                group:
                                    - functions:
                                          - source: value
                                            function: SUM
                - span: 4
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Avg Probability
                            columns:
                                - id: probability
                                  pattern: "#"
                            lookup:
                                uuid: pipeline
                                group:
                                    - functions:
                                          - source: probability
                                            function: AVERAGE
                - span: 4
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Active Deals
                            columns:
                                - id: value
                                  pattern: "#"
                            lookup:
                                uuid: pipeline
                                group:
                                    - functions:
                                          - source: value
                                            function: COUNT
          - columns:
                - components:
                      - displayer:
                            table:
                                pageSize: 10
                                sortable: true
                            lookup:
                                uuid: pipeline
          - columns:
                - span: 6
                  components:
                      - displayer:
                            type: BARCHART
                            subtype: BAR
                            general:
                                title: Pipeline by Stage
                            chart:
                                margin:
                                    left: 100
                            lookup:
                                uuid: pipeline
                                group:
                                    - columnGroup:
                                          source: stage
                                      functions:
                                          - source: stage
                                          - source: value
                                            function: SUM
                - span: 6
                  components:
                      - displayer:
                            type: PIECHART
                            subtype: DONUT
                            general:
                                title: Pipeline by Rep
                            lookup:
                                uuid: pipeline
                                group:
                                    - columnGroup:
                                          source: rep
                                      functions:
                                          - source: rep
                                          - source: value
                                            function: SUM

    # === Page 3: Trends ===
    - name: Trends
      rows:
          - columns:
                - components:
                      - displayer:
                            type: LINECHART
                            subtype: SMOOTH
                            general:
                                title: Revenue Trend
                            chart:
                                zoom: true
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - columnGroup:
                                          source: date
                                          strategy: fixedCalendar
                                          unit: MONTH
                                      functions:
                                          - source: date
                                          - source: amount
                                            function: SUM
          - columns:
                - span: 6
                  components:
                      - displayer:
                            type: BARCHART
                            general:
                                title: Quarterly Revenue
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - columnGroup:
                                          source: date
                                          strategy: fixedCalendar
                                          unit: QUARTER
                                      functions:
                                          - source: date
                                          - source: amount
                                            function: SUM
                - span: 6
                  components:
                      - displayer:
                            type: BARCHART
                            subtype: COLUMN_STACKED
                            general:
                                title: Category Trend
                            lookup:
                                uuid: sales_transactions
                                filter:
                                    - column: status
                                      function: EQUALS_TO
                                      args:
                                          - Won
                                group:
                                    - columnGroup:
                                          source: category
                                      functions:
                                          - source: category
                                          - source: amount
                                            function: SUM

    # === Page 4: Deals ===
    - name: Deals
      components:
          - displayer:
                table:
                    pageSize: 15
                    sortable: true
                filter:
                    enabled: true
                    notification: true
                columns:
                    - id: amount
                      pattern: "$#,###"
                lookup:
                    uuid: sales_transactions

navTree:
    root_items:
        - type: GROUP
          id: SalesNav
          children:
              - page: Overview
              - page: Pipeline
              - page: Trends
              - page: Deals
```

- [ ] **Step 2: Create the TS companion**

Create `examples/dashboards/Sales/Sales Dashboard.ts`. The TS companion mirrors the YAML structure using the builder API. Import from `@casehubio/ui` and `@casehubio/data`. Use `createLookup` with `groupOp`/`filterOp` for lookups, or the newer `lookup`/`groupBy`/`filterBy`/`groupByCalendar` helpers.

Key differences from YAML:
- Properties: pass `{ GoalsFunction: "SUM" }` as first arg to `page()` — but since `${GoalsFunction}` substitution is YAML-only, hardcode `"SUM"` in lookup function references
- Subtypes: `"column-stacked"` not `COLUMN_STACKED`, `"bar"` not `BAR`, `"donut"` not `DONUT`, `"labels"` not `SELECTOR_LABELS`, `"smooth"` not `SMOOTH`
- Column IDs: cast with `as ColumnId`
- Dataset IDs: cast with `as DataSetId`

Follow the exact pattern from `FIFA 2022 Goals.ts` and `Contact Manager.ts` for structure.

- [ ] **Step 3: Build and verify**

```bash
yarn workspace @casehub/pages-examples run build
```

Expected: `samples.json` regenerated with Sales category, build succeeds.

```bash
yarn workspace @casehub/pages-examples run dev
```

Open http://localhost:8080 — navigate to Sales → Overview. Verify:
- Sidebar shows 4 entries (Overview, Pipeline, Trends, Deals)
- Metrics row renders 4 cards
- Stacked bar chart shows regions
- Donut shows categories
- Selector filters all listening charts
- All 4 pages navigate correctly

- [ ] **Step 4: Commit**

```bash
git add examples/dashboards/Sales/
git commit -m "feat: add Sales Dashboard example — sidebar, donut, groupByCalendar, selfApply

Exercises: PIECHART/DONUT, COLUMN_STACKED, BAR horizontal, SELECTOR_LABELS
with selfApply, groupByCalendar (MONTH/QUARTER), SMOOTH, zoom, properties
substitution (GoalsFunction), sortable tables with column expressions.

Refs #14"
```

---

### Task 2: IoT Fleet Monitor

**Files:**
- Create: `examples/dashboards/IoT/Fleet Monitor.dash.yaml`
- Create: `examples/dashboards/IoT/Fleet Monitor.ts`

**Produces:** A 3-page sidebar dashboard in dark mode with 2 inline datasets (~50 + ~6 rows). Exercises: AREA_STACKED, SMOOTH, meterChart with end/warning/critical, dark mode, timeseries.

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/dashboards/IoT/Fleet Monitor.dash.yaml`:

```yaml
global:
    mode: dark
    displayer:
        chart:
            resizable: true
            height: 300
            grid:
                y: false
                x: false

datasets:
    - uuid: sensor_readings
      content: >-
          [
              ["2026-06-25T00:00", "DEV-001", "Warehouse A", 21.3, 45, 1013, 98, "Online"],
              ["2026-06-25T01:00", "DEV-001", "Warehouse A", 21.1, 46, 1013, 97, "Online"],
              ["2026-06-25T02:00", "DEV-001", "Warehouse A", 20.8, 47, 1014, 96, "Online"],
              ["2026-06-25T03:00", "DEV-001", "Warehouse A", 20.5, 48, 1014, 95, "Online"],
              ["2026-06-25T04:00", "DEV-001", "Warehouse A", 20.9, 46, 1013, 94, "Online"],
              ["2026-06-25T00:00", "DEV-002", "Warehouse B", 22.5, 42, 1012, 85, "Online"],
              ["2026-06-25T01:00", "DEV-002", "Warehouse B", 22.8, 43, 1012, 84, "Online"],
              ["2026-06-25T02:00", "DEV-002", "Warehouse B", 23.1, 44, 1013, 83, "Online"],
              ["2026-06-25T03:00", "DEV-002", "Warehouse B", 23.4, 45, 1013, 82, "Online"],
              ["2026-06-25T04:00", "DEV-002", "Warehouse B", 23.0, 43, 1012, 81, "Online"],
              ["2026-06-25T00:00", "DEV-003", "Factory Floor", 28.2, 55, 1011, 72, "Warning"],
              ["2026-06-25T01:00", "DEV-003", "Factory Floor", 29.5, 57, 1010, 70, "Warning"],
              ["2026-06-25T02:00", "DEV-003", "Factory Floor", 31.0, 60, 1010, 68, "Warning"],
              ["2026-06-25T03:00", "DEV-003", "Factory Floor", 32.5, 62, 1009, 65, "Warning"],
              ["2026-06-25T04:00", "DEV-003", "Factory Floor", 30.8, 58, 1010, 63, "Warning"],
              ["2026-06-25T00:00", "DEV-004", "Cold Storage", -18.2, 30, 1015, 55, "Online"],
              ["2026-06-25T01:00", "DEV-004", "Cold Storage", -17.8, 31, 1015, 53, "Online"],
              ["2026-06-25T02:00", "DEV-004", "Cold Storage", -17.5, 32, 1016, 51, "Online"],
              ["2026-06-25T03:00", "DEV-004", "Cold Storage", -18.0, 30, 1015, 49, "Online"],
              ["2026-06-25T04:00", "DEV-004", "Cold Storage", -18.5, 29, 1015, 47, "Online"],
              ["2026-06-25T00:00", "DEV-005", "Loading Dock", 25.0, 65, 1012, 30, "Online"],
              ["2026-06-25T01:00", "DEV-005", "Loading Dock", 24.5, 67, 1012, 28, "Online"],
              ["2026-06-25T02:00", "DEV-005", "Loading Dock", 23.8, 70, 1013, 25, "Online"],
              ["2026-06-25T03:00", "DEV-005", "Loading Dock", 23.2, 72, 1013, 22, "Online"],
              ["2026-06-25T04:00", "DEV-005", "Loading Dock", 24.0, 68, 1012, 19, "Online"],
              ["2026-06-25T00:00", "DEV-006", "Server Room", 22.0, 38, 1014, 15, "Offline"],
              ["2026-06-25T01:00", "DEV-006", "Server Room", 22.3, 39, 1014, 12, "Offline"],
              ["2026-06-25T02:00", "DEV-006", "Server Room", 22.5, 40, 1013, 10, "Offline"],
              ["2026-06-25T03:00", "DEV-006", "Server Room", 22.8, 41, 1013, 8, "Offline"],
              ["2026-06-25T04:00", "DEV-006", "Server Room", 23.0, 42, 1013, 5, "Offline"],
              ["2026-06-25T05:00", "DEV-001", "Warehouse A", 21.5, 44, 1013, 93, "Online"],
              ["2026-06-25T06:00", "DEV-001", "Warehouse A", 22.0, 43, 1012, 92, "Online"],
              ["2026-06-25T05:00", "DEV-002", "Warehouse B", 23.5, 44, 1012, 80, "Online"],
              ["2026-06-25T06:00", "DEV-002", "Warehouse B", 24.0, 45, 1011, 79, "Online"],
              ["2026-06-25T05:00", "DEV-003", "Factory Floor", 33.0, 63, 1009, 60, "Warning"],
              ["2026-06-25T06:00", "DEV-003", "Factory Floor", 34.2, 65, 1008, 58, "Warning"],
              ["2026-06-25T05:00", "DEV-004", "Cold Storage", -19.0, 28, 1016, 45, "Online"],
              ["2026-06-25T06:00", "DEV-004", "Cold Storage", -18.8, 29, 1015, 43, "Online"],
              ["2026-06-25T05:00", "DEV-005", "Loading Dock", 24.5, 66, 1012, 16, "Online"],
              ["2026-06-25T06:00", "DEV-005", "Loading Dock", 25.2, 64, 1011, 14, "Online"],
              ["2026-06-25T07:00", "DEV-001", "Warehouse A", 22.5, 42, 1012, 91, "Online"],
              ["2026-06-25T08:00", "DEV-001", "Warehouse A", 23.0, 41, 1012, 90, "Online"],
              ["2026-06-25T07:00", "DEV-003", "Factory Floor", 35.0, 67, 1008, 55, "Warning"],
              ["2026-06-25T08:00", "DEV-003", "Factory Floor", 36.1, 70, 1007, 52, "Warning"],
              ["2026-06-25T07:00", "DEV-005", "Loading Dock", 25.8, 62, 1011, 11, "Online"],
              ["2026-06-25T08:00", "DEV-005", "Loading Dock", 26.3, 60, 1011, 8, "Online"]
          ]
      columns:
          - id: timestamp
            type: DATE
          - id: deviceId
            type: LABEL
          - id: location
            type: LABEL
          - id: temperature
            type: NUMBER
          - id: humidity
            type: NUMBER
          - id: pressure
            type: NUMBER
          - id: battery
            type: NUMBER
          - id: status
            type: LABEL

    - uuid: devices
      content: >-
          [
              ["DEV-001", "Environmental Sensor A1", 51.5074, -0.1278, "Environmental", "2025-01-15"],
              ["DEV-002", "Environmental Sensor B1", 51.5080, -0.1290, "Environmental", "2025-02-20"],
              ["DEV-003", "Industrial Monitor F1", 51.5060, -0.1250, "Industrial", "2025-03-10"],
              ["DEV-004", "Cold Chain Tracker C1", 51.5090, -0.1300, "Storage", "2025-04-05"],
              ["DEV-005", "Dock Sensor L1", 51.5055, -0.1240, "Environmental", "2025-05-12"],
              ["DEV-006", "Server Room Monitor S1", 51.5070, -0.1265, "Industrial", "2024-11-01"]
          ]
      columns:
          - id: deviceId
            type: LABEL
          - id: name
            type: TEXT
          - id: lat
            type: NUMBER
          - id: lon
            type: NUMBER
          - id: type
            type: LABEL
          - id: installDate
            type: DATE

pages:
    - name: index
      components:
          - type: SIDEBAR
            properties:
                navGroupId: IoTNav

    # === Page 1: Fleet Status ===
    - name: Fleet Status
      rows:
          - columns:
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Devices Online
                            columns:
                                - id: deviceId
                                  pattern: "#"
                            lookup:
                                uuid: devices
                                filter:
                                    - column: deviceId
                                      function: NOT_IN
                                      args: []
                                group:
                                    - functions:
                                          - source: deviceId
                                            function: COUNT
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Avg Temperature
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - functions:
                                          - source: temperature
                                            function: AVERAGE
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Avg Humidity
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - functions:
                                          - source: humidity
                                            function: AVERAGE
                - span: 3
                  components:
                      - displayer:
                            type: METRIC
                            general:
                                title: Low Battery
                            columns:
                                - id: battery
                                  pattern: "#"
                            lookup:
                                uuid: sensor_readings
                                filter:
                                    - column: battery
                                      function: LOWER_THAN
                                      args:
                                          - "20"
                                group:
                                    - functions:
                                          - source: battery
                                            function: COUNT
          - columns:
                - span: 8
                  components:
                      - displayer:
                            table:
                                sortable: true
                            filter:
                                listening: true
                            general:
                                title: Device Registry
                            lookup:
                                uuid: devices
                - span: 4
                  components:
                      - displayer:
                            type: METERCHART
                            general:
                                title: Temperature
                            filter:
                                listening: true
                            meter:
                                end: 60
                                warning: 30
                                critical: 40
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - functions:
                                          - source: temperature
                                            function: AVERAGE
                      - displayer:
                            type: METERCHART
                            general:
                                title: Humidity
                            filter:
                                listening: true
                            meter:
                                end: 100
                                warning: 70
                                critical: 85
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - functions:
                                          - source: humidity
                                            function: AVERAGE
                      - displayer:
                            type: METERCHART
                            general:
                                title: Pressure (hPa)
                            filter:
                                listening: true
                            meter:
                                end: 1060
                                warning: 1020
                                critical: 1040
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - functions:
                                          - source: pressure
                                            function: AVERAGE
          - columns:
                - components:
                      - displayer:
                            type: SELECTOR
                            subtype: SELECTOR_LABELS
                            filter:
                                notification: true
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - columnGroup:
                                          source: location
                                      functions:
                                          - source: location

    # === Page 2: Sensor History ===
    - name: Sensor History
      rows:
          - columns:
                - components:
                      - displayer:
                            type: AREACHART
                            subtype: AREA_STACKED
                            general:
                                title: Temperature by Device
                            chart:
                                zoom: true
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - columnGroup:
                                          source: deviceId
                                      functions:
                                          - source: deviceId
                                          - source: temperature
                                            function: AVERAGE
          - columns:
                - span: 6
                  components:
                      - displayer:
                            type: LINECHART
                            subtype: SMOOTH
                            general:
                                title: Humidity by Device
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - columnGroup:
                                          source: deviceId
                                      functions:
                                          - source: deviceId
                                          - source: humidity
                                            function: AVERAGE
                - span: 6
                  components:
                      - displayer:
                            type: AREACHART
                            general:
                                title: Pressure by Device
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - columnGroup:
                                          source: deviceId
                                      functions:
                                          - source: deviceId
                                          - source: pressure
                                            function: AVERAGE
          - columns:
                - components:
                      - displayer:
                            type: TIMESERIES
                            general:
                                title: Battery Level
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - columnGroup:
                                          source: timestamp
                                      functions:
                                          - source: timestamp
                                          - source: battery
                                            function: AVERAGE

    # === Page 3: Device Detail ===
    - name: Device Detail
      rows:
          - columns:
                - components:
                      - displayer:
                            table:
                                sortable: true
                            filter:
                                listening: true
                                notification: true
                            general:
                                title: Devices
                            lookup:
                                uuid: devices
          - columns:
                - span: 6
                  components:
                      - displayer:
                            type: BARCHART
                            general:
                                title: Readings by Device
                            filter:
                                listening: true
                            lookup:
                                uuid: sensor_readings
                                group:
                                    - columnGroup:
                                          source: deviceId
                                      functions:
                                          - source: deviceId
                                          - source: temperature
                                            function: AVERAGE
                                          - source: humidity
                                            function: AVERAGE
                - span: 6
                  components:
                      - displayer:
                            type: PIECHART
                            subtype: DONUT
                            general:
                                title: Devices by Type
                            filter:
                                listening: true
                            lookup:
                                uuid: devices
                                group:
                                    - columnGroup:
                                          source: type
                                      functions:
                                          - source: type
                                          - source: type
                                            function: COUNT
          - columns:
                - components:
                      - displayer:
                            table:
                                pageSize: 10
                                sortable: true
                            filter:
                                listening: true
                            columns:
                                - id: battery
                                  pattern: "##%"
                            general:
                                title: Sensor Readings
                            lookup:
                                uuid: sensor_readings

navTree:
    root_items:
        - type: GROUP
          id: IoTNav
          children:
              - page: Fleet Status
              - page: Sensor History
              - page: Device Detail
```

- [ ] **Step 2: Create the TS companion**

Create `examples/dashboards/IoT/Fleet Monitor.ts` mirroring the YAML. Use dark mode via `{ mode: "dark" }` in global settings. Subtypes: `"area-stacked"`, `"smooth"`, `"donut"`, `"labels"`. Include `meter()` builder with `end`, `warning`, `critical` props.

- [ ] **Step 3: Build and verify**

```bash
yarn workspace @casehub/pages-examples run build
yarn workspace @casehub/pages-examples run dev
```

Open http://localhost:8080 — navigate to IoT → Fleet Status. Verify:
- Dark mode active
- 3 meter gauges render with correct ranges (especially Pressure at hPa scale)
- Area stacked chart on Sensor History page
- Sidebar navigates all 3 pages

- [ ] **Step 4: Commit**

```bash
git add examples/dashboards/IoT/
git commit -m "feat: add IoT Fleet Monitor example — dark mode, meters, area stacked

Exercises: AREA_STACKED, SMOOTH, meterChart with end/warning/critical,
dark mode, timeseries, SELECTOR_LABELS. Static inline data (accumulate
for inline content deferred to #36).

Refs #14"
```

---

### Task 3: Workforce Analytics (People)

**Files:**
- Create: `examples/dashboards/People/Workforce Analytics.dash.yaml`
- Create: `examples/dashboards/People/Workforce Analytics.ts`

**Produces:** A single-page grid-layout dashboard with 1 inline dataset (~40 rows). Exercises: grid(), panel(), scatterChart, pieChart, COLUMN_STACKED, BAR horizontal, csvExport, SELECTOR_LABELS with filter chains.

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/dashboards/People/Workforce Analytics.dash.yaml`. This dashboard is unique — it uses `grid()` layout instead of rows/columns. However, the YAML format doesn't have a direct `grid` primitive. The grid layout is a TS-only builder. In YAML, approximate it using rows/columns with spans to achieve the 3×4 visual layout.

**Important design note:** The `grid()` and `at()` builders exist only in the TS API. The YAML parser does not support a `grid` component type — it only supports `rows` with `columns` and `span`. The YAML version uses rows/columns to approximate the same layout. The TS version uses the native `grid()` builder — this is the first example to demonstrate it.

```yaml
global:
    displayer:
        chart:
            resizable: true

datasets:
    - uuid: employees
      content: >-
          [
              [1, "Emma Wilson", "Engineering", "Senior", 5.2, 125000, 4, "London", "F", "2021-03-15"],
              [2, "James Chen", "Engineering", "Lead", 7.8, 155000, 5, "London", "M", "2018-09-01"],
              [3, "Sofia Rodriguez", "Product", "Mid", 2.1, 85000, 3, "New York", "F", "2024-05-20"],
              [4, "Amir Patel", "Engineering", "Junior", 0.8, 65000, 3, "London", "M", "2025-10-10"],
              [5, "Li Wei", "Design", "Senior", 6.3, 115000, 4, "Singapore", "M", "2020-04-01"],
              [6, "Priya Sharma", "Marketing", "Mid", 3.5, 78000, 4, "London", "F", "2023-01-15"],
              [7, "Tom Baker", "Sales", "Director", 12.0, 180000, 5, "New York", "M", "2014-06-01"],
              [8, "Yuki Tanaka", "Engineering", "Senior", 4.9, 130000, 4, "Singapore", "F", "2021-08-15"],
              [9, "Maria Santos", "HR", "Mid", 3.2, 72000, 3, "London", "F", "2023-04-20"],
              [10, "David Kim", "Engineering", "Mid", 2.5, 95000, 3, "New York", "M", "2024-01-08"],
              [11, "Anna Müller", "Finance", "Lead", 8.1, 145000, 5, "London", "F", "2018-05-15"],
              [12, "Carlos Ruiz", "Operations", "Senior", 5.5, 105000, 4, "New York", "M", "2021-01-10"],
              [13, "Sarah O'Brien", "Product", "Lead", 6.7, 140000, 4, "London", "F", "2019-11-01"],
              [14, "Raj Gupta", "Engineering", "Junior", 1.0, 62000, 2, "Singapore", "M", "2025-06-15"],
              [15, "Emily Chang", "Design", "Mid", 3.8, 88000, 4, "New York", "F", "2022-09-01"],
              [16, "Michael Brown", "Sales", "Senior", 5.0, 110000, 3, "London", "M", "2021-06-20"],
              [17, "Fatima Al-Rashid", "Engineering", "Senior", 4.2, 128000, 5, "London", "F", "2022-04-01"],
              [18, "Lucas Martin", "Marketing", "Junior", 0.5, 52000, 3, "New York", "M", "2026-01-15"],
              [19, "Mei Lin", "Engineering", "Director", 10.5, 195000, 5, "Singapore", "F", "2016-02-01"],
              [20, "Patrick Kelly", "Operations", "Mid", 2.8, 76000, 3, "London", "M", "2023-08-10"],
              [21, "Olga Ivanova", "Finance", "Senior", 4.5, 118000, 4, "London", "F", "2022-01-20"],
              [22, "Hassan Ahmed", "Engineering", "Mid", 3.0, 98000, 4, "London", "M", "2023-06-01"],
              [23, "Julia Costa", "Product", "Junior", 0.7, 58000, 3, "New York", "F", "2025-11-15"],
              [24, "Ben Thompson", "Design", "Lead", 7.2, 138000, 5, "London", "M", "2019-03-01"],
              [25, "Ling Zhang", "Engineering", "Senior", 5.8, 132000, 4, "Singapore", "NB", "2020-10-15"],
              [26, "Grace Okafor", "HR", "Senior", 6.0, 95000, 4, "New York", "F", "2020-06-01"],
              [27, "Alex Rivera", "Sales", "Mid", 2.3, 82000, 3, "London", "NB", "2024-03-10"],
              [28, "Nina Kowalski", "Marketing", "Senior", 4.8, 98000, 4, "London", "F", "2021-09-01"],
              [29, "Oscar Eriksson", "Engineering", "Lead", 8.5, 160000, 5, "London", "M", "2018-01-15"],
              [30, "Amara Diallo", "Operations", "Junior", 0.9, 55000, 2, "New York", "F", "2025-08-01"],
              [31, "Ryan Mitchell", "Finance", "Mid", 2.6, 82000, 3, "London", "M", "2023-11-15"],
              [32, "Sakura Hayashi", "Design", "Junior", 1.2, 60000, 3, "Singapore", "F", "2025-04-20"],
              [33, "Daniel Murphy", "Engineering", "Junior", 1.5, 68000, 4, "New York", "M", "2025-01-10"],
              [34, "Isabel Ferreira", "Product", "Senior", 5.3, 120000, 4, "London", "F", "2021-02-15"],
              [35, "Kevin Ng", "Sales", "Lead", 9.0, 165000, 5, "Singapore", "M", "2017-08-01"],
              [36, "Zara Hussein", "HR", "Junior", 0.4, 48000, 3, "London", "F", "2026-02-10"],
              [37, "Pierre Leclerc", "Marketing", "Lead", 7.0, 130000, 4, "New York", "M", "2019-05-15"],
              [38, "Tanya Reddy", "Engineering", "Mid", 2.9, 96000, 3, "Singapore", "F", "2023-07-01"],
              [39, "Chris Anderson", "Operations", "Lead", 8.3, 140000, 5, "London", "M", "2018-03-20"],
              [40, "Elena Popov", "Finance", "Junior", 1.1, 56000, 2, "London", "F", "2025-05-01"]
          ]
      columns:
          - id: id
            type: NUMBER
          - id: name
            type: TEXT
          - id: department
            type: LABEL
          - id: level
            type: LABEL
          - id: tenure
            type: NUMBER
          - id: salary
            type: NUMBER
          - id: rating
            type: NUMBER
          - id: location
            type: LABEL
          - id: gender
            type: LABEL
          - id: startDate
            type: DATE

pages:
    - rows:
          # Row 0: Selector (full width)
          - columns:
                - components:
                      - displayer:
                            type: SELECTOR
                            subtype: SELECTOR_LABELS
                            filter:
                                enabled: true
                                notification: true
                            lookup:
                                uuid: employees
                                group:
                                    - columnGroup:
                                          source: department
                                      functions:
                                          - source: department

          # Row 1: Three panels
          - columns:
                - span: 4
                  components:
                      - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Headcount</div><div class='pf-v5-c-card__body'>"
                      - displayer:
                            type: PIECHART
                            subtype: DONUT
                            filter:
                                listening: true
                            lookup:
                                uuid: employees
                                group:
                                    - columnGroup:
                                          source: department
                                      functions:
                                          - source: department
                                          - source: department
                                            function: COUNT
                      - html: "</div></div>"
                - span: 4
                  components:
                      - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Level Distribution</div><div class='pf-v5-c-card__body'>"
                      - displayer:
                            type: BARCHART
                            subtype: COLUMN_STACKED
                            filter:
                                listening: true
                            lookup:
                                uuid: employees
                                group:
                                    - columnGroup:
                                          source: level
                                      functions:
                                          - source: level
                                          - source: level
                                            function: COUNT
                      - html: "</div></div>"
                - span: 4
                  components:
                      - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Avg Salary by Level</div><div class='pf-v5-c-card__body'>"
                      - displayer:
                            type: BARCHART
                            subtype: BAR
                            filter:
                                listening: true
                            chart:
                                margin:
                                    left: 80
                            lookup:
                                uuid: employees
                                sort:
                                    - column: Salary
                                      order: ASCENDING
                                group:
                                    - columnGroup:
                                          source: level
                                      functions:
                                          - source: level
                                          - source: salary
                                            function: AVERAGE
                                            column: Salary
                      - html: "</div></div>"

          # Row 2: Scatter (wide) + Pie
          - columns:
                - span: 8
                  components:
                      - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Tenure vs Salary</div><div class='pf-v5-c-card__body'>"
                      - displayer:
                            type: SCATTERCHART
                            filter:
                                listening: true
                            lookup:
                                uuid: employees
                                group:
                                    - columnGroup:
                                          source: name
                                      functions:
                                          - source: name
                                          - source: tenure
                                          - source: salary
                      - html: "</div></div>"
                - span: 4
                  components:
                      - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Rating Distribution</div><div class='pf-v5-c-card__body'>"
                      - displayer:
                            type: PIECHART
                            filter:
                                listening: true
                            lookup:
                                uuid: employees
                                group:
                                    - columnGroup:
                                          source: rating
                                      functions:
                                          - source: rating
                                          - source: rating
                                            function: COUNT
                      - html: "</div></div>"

          # Row 3: Table (full width)
          - columns:
                - components:
                      - displayer:
                            table:
                                pageSize: 15
                                sortable: true
                            filter:
                                listening: true
                            csvExport: true
                            columns:
                                - id: salary
                                  pattern: "$#,###"
                            lookup:
                                uuid: employees
```

- [ ] **Step 2: Create the TS companion**

Create `examples/dashboards/People/Workforce Analytics.ts`. This is the first example to use the native `grid()` and `at()` layout builders — the key differentiator from the YAML version. Use `panel()` for the card containers instead of raw HTML.

```typescript
import {
  page, grid, at, panel, selector, pieChart, barChart,
  scatterChart, table, inlineDataset,
} from "@casehubio/ui";
import { createLookup } from "@casehubio/data";
import type { DataSetId, ColumnId } from "@casehubio/data";

const employees = "employees" as DataSetId;

// ... inline dataset definition (same data as YAML) ...

export default page(
  grid(3,
    at(0, 0, 3, 1,
      selector({
        subtype: "labels",
        filter: { enabled: true, notification: true },
        lookup: createLookup(employees, [/* groupBy department */]),
      }),
    ),
    at(0, 1, 1, 1,
      panel("Headcount",
        pieChart({
          subtype: "donut",
          filter: { listening: true },
          lookup: createLookup(employees, [/* groupBy department, COUNT */]),
        }),
      ),
    ),
    at(1, 1, 1, 1,
      panel("Level Distribution",
        barChart({
          subtype: "column-stacked",
          filter: { listening: true },
          lookup: createLookup(employees, [/* groupBy level, COUNT */]),
        }),
      ),
    ),
    at(2, 1, 1, 1,
      panel("Avg Salary by Level",
        barChart({
          subtype: "bar",
          filter: { listening: true },
          chart: { margin: { left: 80 } },
          lookup: createLookup(employees, [/* sort + groupBy level, AVG salary */]),
        }),
      ),
    ),
    at(0, 2, 2, 1,
      panel("Tenure vs Salary",
        scatterChart({
          filter: { listening: true },
          lookup: createLookup(employees, [/* groupBy name: tenure, salary */]),
        }),
      ),
    ),
    at(2, 2, 1, 1,
      panel("Rating Distribution",
        pieChart({
          filter: { listening: true },
          lookup: createLookup(employees, [/* groupBy rating, COUNT */]),
        }),
      ),
    ),
    at(0, 3, 3, 1,
      table({
        pageSize: 15,
        sortable: true,
        csvExport: true,
        filter: { listening: true },
        columns: [{ id: "salary" as ColumnId, pattern: "$#,###" }],
        lookup: createLookup(employees, []),
      }),
    ),
  ),
  { datasets: [employeeDataset] },
);
```

The TS companion is the showcase for `grid()` and `panel()` — the YAML approximates it with rows/columns + HTML cards.

- [ ] **Step 3: Build and verify**

```bash
yarn workspace @casehub/pages-examples run build
yarn workspace @casehub/pages-examples run dev
```

Open http://localhost:8080 — navigate to People → Workforce Analytics. Verify:
- Scatter chart renders with tenure on x, salary on y
- Pie chart (not donut) shows ratings
- Donut shows department breakdown
- Selector filters all listening charts
- Table has CSV export button
- Stacked bar and horizontal bar render correctly

- [ ] **Step 4: Commit**

```bash
git add examples/dashboards/People/
git commit -m "feat: add Workforce Analytics example — grid, panel, scatterChart, csvExport

First example to use grid()/at()/panel() in TS companion. Exercises:
scatterChart, pieChart (distinct from donut), COLUMN_STACKED, BAR
horizontal, SELECTOR_LABELS with filter chains, csvExport on table.

Refs #14"
```

---

### Task 4: Patient Tracker (Clinical)

**Files:**
- Create: `examples/dashboards/Clinical/Patient Tracker.dash.yaml`
- Create: `examples/dashboards/Clinical/Patient Tracker.ts`

**Produces:** A 3-page tabs dashboard with 2 inline datasets (~25 + ~60 rows). Exercises: tabs navigation, all form input types with dataScope + save, readonly fields, master-detail cross-filtering, panel(), markdown(), pie, area chart, selector dropdown with selfApply.

- [ ] **Step 1: Create the YAML dashboard**

Create `examples/dashboards/Clinical/Patient Tracker.dash.yaml`. Follow the tabs navigation pattern: an `index` page with `type: TABS` + `navGroupId`, and a `navTree` GROUP matching that ID.

The YAML follows the same structure as Sales/IoT but with tabs instead of sidebar, and adds forms on Page 3. The form page uses `dataScope` + `save` at the page level (same pattern as Contact Manager and Navigation Rebinding's Txn Form).

Build this file following the exact spec from the design document — 25 rows of patient data and 60 rows of vitals readings, with the three pages (Ward Overview, Vitals Monitor, Patient Detail) connected via tabs navigation.

Key elements:
- `type: TABS` with `navGroupId: ClinicalNav`
- Page 1: 4 metrics, selector dropdown (`subtype: SELECTOR_DROPDOWN` → maps to `"dropdown"` in TS) with `selfApply: true`, pie by diagnosis, bar by ward, donut by status, markdown component
- Page 2: areaChart of heart rate, lineCharts for blood pressure and O2, vitals table
- Page 3: panel("Patient Records") with table (filter notification + listening), panel("Edit Patient") with form inputs bound via dataScope

Form page structure (Page 3):
```yaml
    - name: Patient Detail
      dataScope:
          dataset: patients
          idColumn: id
      save:
          trigger: auto
          delay: 2000
          adapter: local
      components:
          - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Patient Records</div><div class='pf-v5-c-card__body'>"
          - displayer:
                table:
                    sortable: true
                filter:
                    listening: true
                    notification: true
                lookup:
                    uuid: patients
          - html: "</div></div>"
          - html: "<div class='pf-v5-c-card pf-m-compact'><div class='pf-v5-c-card__title'>Edit Patient</div><div class='pf-v5-c-card__body'>"
          - text-input:
                field: name
                label: Patient Name
                readonly: true
          - number-input:
                field: age
                label: Age
                readonly: true
          - dropdown:
                field: ward
                label: Ward
                options:
                    values: [ICU, General, Pediatrics, Maternity, Outpatient]
          - dropdown:
                field: status
                label: Status
                options:
                    values: [Stable, Monitoring, Critical]
          - text-input:
                field: doctor
                label: Doctor
          - textarea:
                field: notes
                label: Notes
                rows: 4
          - date-picker:
                field: admitDate
                label: Admit Date
                readonly: true
          - checkbox:
                field: flagged
                label: Flagged for Review
          - html: "</div></div>"
```

- [ ] **Step 2: Create the TS companion**

Create `examples/dashboards/Clinical/Patient Tracker.ts` mirroring the YAML. Use `tabs()` builder for navigation, `panel()` for card containers, `markdown()` for ward protocol notes, and all form builders (textInput, numberInput, dropdown, textarea, datePicker, checkbox).

- [ ] **Step 3: Build and verify**

```bash
yarn workspace @casehub/pages-examples run build
yarn workspace @casehub/pages-examples run dev
```

Open http://localhost:8080 — navigate to Clinical → Patient Tracker. Verify:
- Tabs show 3 entries (Ward Overview, Vitals Monitor, Patient Detail)
- Markdown renders on Ward Overview
- Area chart renders heart rate on Vitals Monitor
- Forms render on Patient Detail with readonly fields grayed out
- Selector dropdown filters with selfApply
- Table selection populates form fields

- [ ] **Step 4: Commit**

```bash
git add examples/dashboards/Clinical/
git commit -m "feat: add Patient Tracker example — tabs, forms, markdown, area chart

Exercises: tabs navigation, all form types (text, number, dropdown,
textarea, date-picker, checkbox) with dataScope + auto-save, readonly
fields, master-detail cross-filtering, panel(), markdown() in
non-DevOps context, pie, areaChart, selector dropdown with selfApply.

Refs #14"
```

---

### Task 5: Gallery Smoke Test and Final Verification

**Files:**
- None created — uses existing `gallery.spec.ts`

**Depends on:** Tasks 1-4

- [ ] **Step 1: Full build**

```bash
yarn install && yarn build && yarn build:examples
```

Expected: Clean build with all 4 new dashboards in samples.json.

- [ ] **Step 2: Verify samples.json**

```bash
node -e "const s = require('./examples/samples.json'); console.log('Total:', s.totalDashboards); s.categories.forEach(c => console.log(' ', c.category, '-', c.dashboards.length))"
```

Expected: Total count increased by 4. New categories: Sales (1), IoT (1), People (1), Clinical (1).

- [ ] **Step 3: Run gallery smoke tests**

```bash
yarn workspace @casehub/pages-examples run test
```

`gallery.spec.ts` iterates over `samples.json` and loads each dashboard. All new dashboards should load without errors.

- [ ] **Step 4: Visual verification in dev server**

```bash
yarn workspace @casehub/pages-examples run dev
```

Walk through all 4 dashboards:
1. **Sales** — sidebar nav, 4 pages, selector filters charts, groupByCalendar renders monthly/quarterly
2. **IoT** — dark mode, meter gauges, area stacked, sidebar nav
3. **People** — single page, all charts render, CSV export button on table
4. **Clinical** — tabs nav, markdown renders, forms with readonly fields, auto-save

- [ ] **Step 5: Commit any fixes from verification**

If any dashboard needed fixes during visual verification, commit them:

```bash
git add -A examples/dashboards/
git commit -m "fix: address rendering issues from gallery verification

Refs #14"
```
