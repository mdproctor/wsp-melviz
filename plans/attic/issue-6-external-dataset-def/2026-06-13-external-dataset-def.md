# ExternalDataSetDef Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implement the external dataset definition model, pluggable data providers, extraction pipeline, presets, CSV/metrics parsing, join, and accumulate — enabling melviz dashboards to fetch and transform data from any REST API, file, or inline source.

**Architecture:** The `ExternalDataSetDef` describes what data to get (url/content/join) and how to transform it (dataPath → type preset → expression pipeline). A `DataProvider` SPI handles the fetch (browser, CORS proxy, server relay, inline, postMessage). The extraction pipeline parses (JSON/CSV/Prometheus text), navigates/extracts/transforms via JSONata, tabulates into a `DataSet` wire format, then calls `toTypedDataSet()` for the typed runtime format. Join concatenates existing datasets. Accumulate appends new-data-first with a rolling cap.

**Tech Stack:** TypeScript 5.6+, Vitest, Zod 3.23+, JSONata 2.2+ (existing), `@melviz/core`

**Spec:** `docs/superpowers/specs/2026-06-13-external-dataset-def-design.md`

---

## Task 1: Extend DataSetErrorCode and add accumulate() to DataSetManager

**Files:**
- Modify: `packages/core/src/dataset/errors.ts`
- Modify: `packages/core/src/dataset/manager.ts`
- Modify: `packages/core/src/dataset/manager.test.ts`

- [ ] **Step 1: Write failing tests for new error codes and accumulate**

Add to `packages/core/src/dataset/manager.test.ts`:

```typescript
describe("DataSetManager — accumulate", () => {
  it("accumulate on empty registry behaves like register", () => {
    const mgr = createDataSetManager();
    const ds = testDataSet([["Alice", "100"]]);
    mgr.accumulate(ID_A, ds);
    expect(mgr.get(ID_A)).toBeDefined();
    expect(mgr.get(ID_A)!.rows).toHaveLength(1);
  });

  it("accumulate puts new rows first, then appends old rows", () => {
    const mgr = createDataSetManager();
    const existing = testDataSet([["Alice", "100"], ["Bob", "200"]]);
    mgr.register(ID_A, existing);
    const fresh = testDataSet([["Charlie", "300"]]);
    mgr.accumulate(ID_A, fresh);
    const result = mgr.get(ID_A)!;
    expect(result.rows).toHaveLength(3);
    expect(result.rows[0]!.text("name" as ColumnId)).toBe("Charlie");
    expect(result.rows[1]!.text("name" as ColumnId)).toBe("Alice");
    expect(result.rows[2]!.text("name" as ColumnId)).toBe("Bob");
  });

  it("accumulate trims oldest rows when maxRows exceeded", () => {
    const mgr = createDataSetManager();
    const existing = testDataSet([["Alice", "100"], ["Bob", "200"], ["Charlie", "300"]]);
    mgr.register(ID_A, existing);
    const fresh = testDataSet([["Diana", "400"]]);
    mgr.accumulate(ID_A, fresh, 3);
    const result = mgr.get(ID_A)!;
    expect(result.rows).toHaveLength(3);
    expect(result.rows[0]!.text("name" as ColumnId)).toBe("Diana");
    expect(result.rows[1]!.text("name" as ColumnId)).toBe("Alice");
    expect(result.rows[2]!.text("name" as ColumnId)).toBe("Bob");
  });

  it("accumulate with zero new rows preserves existing dataset", () => {
    const mgr = createDataSetManager();
    const existing = testDataSet([["Alice", "100"]]);
    mgr.register(ID_A, existing);
    const empty = testDataSet([]);
    mgr.accumulate(ID_A, empty);
    expect(mgr.get(ID_A)!.rows).toHaveLength(1);
    expect(mgr.get(ID_A)!.rows[0]!.text("name" as ColumnId)).toBe("Alice");
  });

  it("accumulate with no maxRows appends all rows", () => {
    const mgr = createDataSetManager();
    const existing = testDataSet([["Alice", "100"], ["Bob", "200"]]);
    mgr.register(ID_A, existing);
    const fresh = testDataSet([["Charlie", "300"]]);
    mgr.accumulate(ID_A, fresh);
    expect(mgr.get(ID_A)!.rows).toHaveLength(3);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- src/dataset/manager.test.ts`
Expected: FAIL — `accumulate` does not exist on `DataSetManager`

- [ ] **Step 3: Add new error codes to DataSetErrorCode**

In `packages/core/src/dataset/errors.ts`, add the 4 new codes:

```typescript
export type DataSetErrorCode =
  | "FETCH_FAILED"
  | "PARSE_FAILED"
  | "SCHEMA_MISMATCH"
  | "TRANSFORM_FAILED"
  | "TIMEOUT"
  | "INVALID_REF"
  | "UNKNOWN_COLUMN"
  | "TYPE_MISMATCH"
  | "UNKNOWN_PROVIDER"
  | "INVALID_OPERATION"
  | "RESOLUTION_FAILED"
  | "UNKNOWN_PRESET"
  | "EXTRACTION_ERROR"
  | "INVALID_DEFINITION"
  | "EMPTY_RESULT";
```

- [ ] **Step 4: Add accumulate() to DataSetManager interface and implementation**

In `packages/core/src/dataset/manager.ts`:

Add to the `DataSetManager` interface:
```typescript
accumulate(id: DataSetId, dataset: TypedDataSet, maxRows?: number): void;
```

Add to `DataSetManagerImpl`:
```typescript
accumulate(id: DataSetId, dataset: TypedDataSet, maxRows?: number): void {
  if (dataset.rows.length === 0) {
    if (!this.datasets.has(id)) {
      this.datasets.set(id, dataset);
    }
    return;
  }
  const existing = this.datasets.get(id);
  if (!existing) {
    this.datasets.set(id, dataset);
    return;
  }
  const combined = [...dataset.rows, ...existing.rows];
  const rows = maxRows !== undefined && maxRows >= 0
    ? combined.slice(0, maxRows)
    : combined;
  this.datasets.set(id, { columns: dataset.columns, rows });
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- src/dataset/manager.test.ts`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```
feat: add accumulate() to DataSetManager and extend DataSetErrorCode  Refs #6
```

---

## Task 2: Types — ExternalDataSetDef, DataRequest, FetchResult, HttpMethod

**Files:**
- Create: `packages/core/src/dataset/external/types.ts`

- [ ] **Step 1: Create the types file**

```typescript
import type { ColumnId, ColumnType, DataSetId } from "../types.js";

export enum HttpMethod {
  GET = "GET",
  POST = "POST",
  PUT = "PUT",
  DELETE = "DELETE",
}

export interface ExternalColumnDef {
  readonly id: ColumnId;
  readonly name?: string;
  readonly type: ColumnType;
}

export interface ExternalDataSetDef {
  readonly uuid: DataSetId;
  readonly name?: string;

  readonly url?: string;
  readonly content?: string;
  readonly join?: readonly DataSetId[];

  readonly method?: HttpMethod;
  readonly headers?: Readonly<Record<string, string>>;
  readonly query?: Readonly<Record<string, string>>;
  readonly form?: Readonly<Record<string, string>>;
  readonly body?: string;

  readonly dataPath?: string;
  readonly type?: string;
  readonly expression?: string;

  readonly columns?: readonly ExternalColumnDef[];

  readonly cacheEnabled?: boolean;
  readonly cacheMaxRows?: number;
  readonly refreshTime?: string;
  readonly accumulate?: boolean;
}

export interface DataRequest {
  readonly url: string;
  readonly method: HttpMethod;
  readonly headers: Readonly<Record<string, string>>;
  readonly query: Readonly<Record<string, string>>;
  readonly form?: Readonly<Record<string, string>>;
  readonly body?: string;
}

export interface FetchResult {
  readonly data: unknown;
  readonly contentType?: string;
}

export interface MelvizDataMessage {
  readonly type: "melviz-dataset";
  readonly dataSetId: string;
  readonly data: unknown;
  readonly contentType?: string;
}

export interface ExtractionPreset {
  readonly id: string;
  readonly expression: string;
}

export interface PresetRegistry {
  get(id: string): ExtractionPreset | undefined;
  has(id: string): boolean;
}

export interface DataProvider {
  fetch(request: DataRequest): Promise<FetchResult>;
}

export interface DataProviderConfig {
  readonly defaultProvider?: "browser" | "server-relay";
  readonly corsProxy?: {
    readonly url: string;
    readonly enabled: boolean;
  };
  readonly serverRelay?: {
    readonly endpoint: string;
  };
}

export interface ExtractionResult {
  readonly dataset: import("../types.js").TypedDataSet;
  readonly inferredColumns: boolean;
}

export interface ResolveResult {
  readonly dataset: import("../types.js").TypedDataSet;
  readonly inferredColumns: boolean;
  readonly source: "url" | "content" | "join";
}
```

- [ ] **Step 2: Verify it compiles**

Run: `yarn workspace @melviz/core run build`
Expected: PASS (no errors)

- [ ] **Step 3: Commit**

```
feat: add ExternalDataSetDef types and interfaces  Refs #6
```

---

## Task 3: Zod validation schema for ExternalDataSetDef

**Files:**
- Create: `packages/core/src/dataset/external/schema.ts`
- Create: `packages/core/src/dataset/external/schema.test.ts`

- [ ] **Step 1: Write failing tests for validation rules**

Create `packages/core/src/dataset/external/schema.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { parseExternalDataSetDef } from "./schema.js";

const valid = (overrides: Record<string, unknown>) =>
  parseExternalDataSetDef({ uuid: "test-id", url: "https://example.com/api", ...overrides });

describe("ExternalDataSetDef schema", () => {
  it("accepts minimal url-based definition", () => {
    const result = valid({});
    expect(result.uuid).toBe("test-id");
    expect(result.url).toBe("https://example.com/api");
  });

  it("accepts content-based definition", () => {
    const result = parseExternalDataSetDef({ uuid: "x", content: '[{"a":1}]' });
    expect(result.content).toBe('[{"a":1}]');
  });

  it("accepts join-based definition", () => {
    const result = parseExternalDataSetDef({ uuid: "x", join: ["ds-a", "ds-b"] });
    expect(result.join).toEqual(["ds-a", "ds-b"]);
  });

  it("rejects missing uuid", () => {
    expect(() => parseExternalDataSetDef({ url: "https://x.com" })).toThrow();
  });

  it("rejects no data source (no url, content, or join)", () => {
    expect(() => parseExternalDataSetDef({ uuid: "x" })).toThrow(/Exactly one/);
  });

  it("rejects multiple data sources (url + content)", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", url: "https://x.com", content: "[]",
    })).toThrow(/Exactly one/);
  });

  it("rejects form + body together", () => {
    expect(() => valid({ form: { a: "1" }, body: '{"a":1}' })).toThrow(/mutually exclusive/);
  });

  it("rejects method without url", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", content: "[]", method: "POST",
    })).toThrow(/only valid when url/);
  });

  it("rejects headers without url", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", content: "[]", headers: { "X-Key": "v" },
    })).toThrow(/only valid when url/);
  });

  it("rejects extraction fields on join", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", join: ["a"], expression: "$.data",
    })).toThrow(/not valid with join/);
  });

  it("rejects extraction fields (type) on join", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", join: ["a"], type: "prometheus",
    })).toThrow(/not valid with join/);
  });

  it("rejects extraction fields (dataPath) on join", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", join: ["a"], dataPath: "data.items",
    })).toThrow(/not valid with join/);
  });

  it("rejects accumulate without url", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", content: "[]", accumulate: true,
    })).toThrow(/only valid when url/);
  });

  it("rejects refreshTime without url", () => {
    expect(() => parseExternalDataSetDef({
      uuid: "x", content: "[]", refreshTime: "10minute",
    })).toThrow(/only valid when url/);
  });

  it("validates refreshTime format", () => {
    expect(() => valid({ refreshTime: "10min" })).toThrow();
    expect(() => valid({ refreshTime: "abc" })).toThrow();
    const result = valid({ refreshTime: "30second" });
    expect(result.refreshTime).toBe("30second");
  });

  it("accepts all valid refreshTime units", () => {
    for (const unit of ["millisecond", "second", "minute", "hour", "day", "week", "month", "quarter", "year"]) {
      expect(valid({ refreshTime: `5${unit}` }).refreshTime).toBe(`5${unit}`);
    }
  });

  it("allows type and expression together (composable pipeline)", () => {
    const result = valid({ type: "prometheus", expression: "$[value > 100]" });
    expect(result.type).toBe("prometheus");
    expect(result.expression).toBe("$[value > 100]");
  });

  it("allows dataPath with type and expression", () => {
    const result = valid({
      dataPath: "data.items",
      type: "prometheus",
      expression: "$[value > 0]",
    });
    expect(result.dataPath).toBe("data.items");
  });

  it("accepts columns with optional name", () => {
    const result = valid({
      columns: [
        { id: "col1", type: "NUMBER" },
        { id: "col2", name: "Column Two", type: "LABEL" },
      ],
    });
    expect(result.columns).toHaveLength(2);
    expect(result.columns![0]!.name).toBeUndefined();
    expect(result.columns![1]!.name).toBe("Column Two");
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/schema.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the schema**

Create `packages/core/src/dataset/external/schema.ts`:

```typescript
import { z } from "zod";
import { ColumnType } from "../types.js";
import { HttpMethod } from "./types.js";

const externalColumnDefSchema = z.object({
  id: z.string().min(1),
  name: z.string().optional(),
  type: z.nativeEnum(ColumnType),
});

const externalDataSetDefSchema = z.object({
  uuid: z.string().min(1),
  name: z.string().optional(),

  url: z.string().optional(),
  content: z.string().optional(),
  join: z.array(z.string().min(1)).min(1).optional(),

  method: z.nativeEnum(HttpMethod).optional(),
  headers: z.record(z.string()).optional(),
  query: z.record(z.string()).optional(),
  form: z.record(z.string()).optional(),
  body: z.string().optional(),

  dataPath: z.string().optional(),
  expression: z.string().optional(),
  type: z.string().optional(),

  columns: z.array(externalColumnDefSchema).optional(),

  cacheEnabled: z.boolean().optional(),
  cacheMaxRows: z.number().int().positive().optional(),
  refreshTime: z.string().regex(
    /^\d+(millisecond|second|minute|hour|day|week|month|quarter|year)$/,
    "Must be a number followed by a time unit (e.g. '10minute', '30second')",
  ).optional(),
  accumulate: z.boolean().optional(),
}).refine(
  d => [d.url, d.content, d.join].filter(Boolean).length === 1,
  { message: "Exactly one of url, content, or join is required" },
).refine(
  d => !(d.form && d.body),
  { message: "form and body are mutually exclusive" },
).refine(
  d => d.url !== undefined || [d.method, d.headers, d.query, d.form, d.body]
    .every(v => v === undefined),
  { message: "method, headers, query, form, body are only valid when url is set" },
).refine(
  d => !d.join || [d.dataPath, d.type, d.expression]
    .every(v => v === undefined),
  { message: "dataPath, type, expression are not valid with join (nothing to extract)" },
).refine(
  d => !d.accumulate || d.url !== undefined,
  { message: "accumulate is only valid when url is set" },
).refine(
  d => !d.refreshTime || d.url !== undefined,
  { message: "refreshTime is only valid when url is set" },
);

export type ParsedExternalDataSetDef = z.output<typeof externalDataSetDefSchema>;

export function parseExternalDataSetDef(input: unknown): ParsedExternalDataSetDef {
  return externalDataSetDefSchema.parse(input);
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/schema.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat: add Zod validation schema for ExternalDataSetDef  Refs #6
```

---

## Task 4: CSV parser

**Files:**
- Create: `packages/core/src/dataset/external/csv.ts`
- Create: `packages/core/src/dataset/external/csv.test.ts`

- [ ] **Step 1: Write failing tests**

Create `packages/core/src/dataset/external/csv.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { parseCsv } from "./csv.js";

describe("parseCsv", () => {
  it("parses simple CSV with header row", () => {
    const result = parseCsv("name,age\nAlice,30\nBob,25");
    expect(result.headers).toEqual(["name", "age"]);
    expect(result.rows).toEqual([["Alice", "30"], ["Bob", "25"]]);
  });

  it("parses CSV without header row", () => {
    const result = parseCsv("Alice,30\nBob,25", { hasHeader: false });
    expect(result.headers).toEqual(["Column 0", "Column 1"]);
    expect(result.rows).toEqual([["Alice", "30"], ["Bob", "25"]]);
  });

  it("handles quoted fields containing commas", () => {
    const result = parseCsv('name,address\nAlice,"123 Main St, Apt 4"\nBob,"456 Oak Ave"');
    expect(result.rows[0]).toEqual(["Alice", "123 Main St, Apt 4"]);
  });

  it("handles quoted fields containing newlines", () => {
    const result = parseCsv('name,note\nAlice,"line1\nline2"\nBob,simple');
    expect(result.rows[0]).toEqual(["Alice", "line1\nline2"]);
    expect(result.rows[1]).toEqual(["Bob", "simple"]);
  });

  it("handles escaped quotes (doubled quotes)", () => {
    const result = parseCsv('name,quote\nAlice,"She said ""hello"""\nBob,plain');
    expect(result.rows[0]).toEqual(["Alice", 'She said "hello"']);
  });

  it("skips empty lines", () => {
    const result = parseCsv("name,age\nAlice,30\n\nBob,25\n");
    expect(result.rows).toHaveLength(2);
  });

  it("handles \\r\\n line endings", () => {
    const result = parseCsv("name,age\r\nAlice,30\r\nBob,25");
    expect(result.headers).toEqual(["name", "age"]);
    expect(result.rows).toEqual([["Alice", "30"], ["Bob", "25"]]);
  });

  it("handles trailing delimiter producing empty final field", () => {
    const result = parseCsv("a,b,\n1,2,", { hasHeader: false });
    expect(result.rows[0]).toEqual(["1", "2", ""]);
  });

  it("supports custom delimiter", () => {
    const result = parseCsv("name\tage\nAlice\t30", { delimiter: "\t" });
    expect(result.headers).toEqual(["name", "age"]);
    expect(result.rows[0]).toEqual(["Alice", "30"]);
  });

  it("handles single-row CSV (header only, no data)", () => {
    const result = parseCsv("name,age");
    expect(result.headers).toEqual(["name", "age"]);
    expect(result.rows).toEqual([]);
  });

  it("handles whitespace-only fields", () => {
    const result = parseCsv("a,b\n , ");
    expect(result.rows[0]).toEqual([" ", " "]);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/csv.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the CSV parser**

Create `packages/core/src/dataset/external/csv.ts`:

```typescript
export interface CsvParseOptions {
  readonly delimiter?: string;
  readonly hasHeader?: boolean;
  readonly quote?: string;
}

export interface CsvParseResult {
  readonly headers: string[];
  readonly rows: string[][];
}

export function parseCsv(
  raw: string,
  options?: CsvParseOptions,
): CsvParseResult {
  const delimiter = options?.delimiter ?? ",";
  const hasHeader = options?.hasHeader ?? true;
  const quote = options?.quote ?? '"';

  const rows: string[][] = [];
  let i = 0;
  const len = raw.length;

  while (i < len) {
    const row: string[] = [];
    while (i < len) {
      let field = "";
      if (raw[i] === quote) {
        i++;
        while (i < len) {
          if (raw[i] === quote) {
            if (i + 1 < len && raw[i + 1] === quote) {
              field += quote;
              i += 2;
            } else {
              i++;
              break;
            }
          } else {
            field += raw[i]!;
            i++;
          }
        }
      } else {
        while (i < len && raw[i] !== delimiter && raw[i] !== "\n" && raw[i] !== "\r") {
          field += raw[i]!;
          i++;
        }
      }
      row.push(field);
      if (i < len && raw[i] === delimiter) {
        i++;
        continue;
      }
      break;
    }
    if (i < len && raw[i] === "\r") i++;
    if (i < len && raw[i] === "\n") i++;

    if (row.length === 1 && row[0] === "") continue;
    rows.push(row);
  }

  if (hasHeader) {
    const headers = rows.shift() ?? [];
    return { headers, rows };
  }

  const colCount = rows[0]?.length ?? 0;
  const headers = Array.from({ length: colCount }, (_, idx) => `Column ${idx}`);
  return { headers, rows };
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/csv.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat: add RFC 4180 CSV parser  Refs #6
```

---

## Task 5: Prometheus text exposition format parser

**Files:**
- Create: `packages/core/src/dataset/external/metrics-parser.ts`
- Create: `packages/core/src/dataset/external/metrics-parser.test.ts`

- [ ] **Step 1: Write failing tests**

Create `packages/core/src/dataset/external/metrics-parser.test.ts`:

```typescript
import { describe, it, expect } from "vitest";
import { parseMetrics } from "./metrics-parser.js";

describe("parseMetrics", () => {
  it("parses simple metric line", () => {
    const result = parseMetrics('up{instance="localhost:9090"} 1');
    expect(result).toEqual([["up", 'instance="localhost:9090"', "1"]]);
  });

  it("parses metric without labels", () => {
    const result = parseMetrics("process_cpu_seconds_total 42.5");
    expect(result).toEqual([["process_cpu_seconds_total", "", "42.5"]]);
  });

  it("skips comment lines", () => {
    const result = parseMetrics(
      '# HELP up Whether the target is up\n# TYPE up gauge\nup{instance="a"} 1',
    );
    expect(result).toHaveLength(1);
    expect(result[0]![0]).toBe("up");
  });

  it("replaces NaN values with -1", () => {
    const result = parseMetrics("some_metric{} NaN");
    expect(result[0]![2]).toBe("-1");
  });

  it("handles multiple metrics", () => {
    const result = parseMetrics(
      'node_cpu{cpu="0"} 100\nnode_cpu{cpu="1"} 200',
    );
    expect(result).toHaveLength(2);
    expect(result[0]![2]).toBe("100");
    expect(result[1]![2]).toBe("200");
  });

  it("skips empty lines", () => {
    const result = parseMetrics("up 1\n\ndown 0\n");
    expect(result).toHaveLength(2);
  });

  it("handles metric with multiple labels", () => {
    const result = parseMetrics('http_requests{method="GET",code="200"} 42');
    expect(result[0]![1]).toBe('method="GET",code="200"');
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/metrics-parser.test.ts`
Expected: FAIL — module not found

- [ ] **Step 3: Implement the metrics parser**

Create `packages/core/src/dataset/external/metrics-parser.ts`:

```typescript
export function parseMetrics(input: string): string[][] {
  const rows: string[][] = [];
  const lines = input.split("\n");

  for (const line of lines) {
    if (line.startsWith("#") || line.trim() === "") continue;

    let metric = "";
    let labels = "";
    let value = "";

    const chars = line;
    let pos = 0;
    let target: "metric" | "labels" | "value" = "metric";

    while (pos < chars.length) {
      const ch = chars[pos]!;
      if (ch === "{" && target === "metric") {
        target = "labels";
        pos++;
        continue;
      }
      if (ch === "}" && target === "labels") {
        target = "value";
        pos++;
        continue;
      }
      if (ch === " " && target === "metric") {
        target = "value";
        pos++;
        continue;
      }
      if (target === "metric") metric += ch;
      else if (target === "labels") labels += ch;
      else value += ch;
      pos++;
    }

    value = value.trim();
    if (metric === "" || value === "") continue;
    if (value === "NaN") value = "-1";

    rows.push([metric, labels, value]);
  }

  return rows;
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/metrics-parser.test.ts`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```
feat: add Prometheus text exposition format parser  Refs #6
```

---

## Task 6: Preset Registry and built-in presets

**Files:**
- Create: `packages/core/src/dataset/external/presets/registry.ts`
- Create: `packages/core/src/dataset/external/presets/registry.test.ts`
- Create: `packages/core/src/dataset/external/presets/prometheus.ts`
- Create: `packages/core/src/dataset/external/presets/prometheus.test.ts`
- Create: `packages/core/src/dataset/external/presets/elasticsearch.ts`
- Create: `packages/core/src/dataset/external/presets/elasticsearch.test.ts`
- Create: `packages/core/src/dataset/external/presets/graphql-relay.ts`
- Create: `packages/core/src/dataset/external/presets/graphql-relay.test.ts`
- Create: `packages/core/src/dataset/external/presets/jsonapi.ts`
- Create: `packages/core/src/dataset/external/presets/jsonapi.test.ts`
- Create: `packages/core/src/dataset/external/presets/odata.ts`
- Create: `packages/core/src/dataset/external/presets/odata.test.ts`
- Create: `packages/core/src/dataset/external/presets/kubernetes.ts`
- Create: `packages/core/src/dataset/external/presets/kubernetes.test.ts`

This task is large. Implement each preset as a sub-step using TDD — write the test with the example fixture from the spec (§4), then implement the JSONata expression. Each preset is a single export with an `ExtractionPreset` object. The registry test verifies the factory loads all built-ins and the read-only contract.

The spec (§4.1–4.6) defines the exact input → output for each preset. Each test uses that fixture data verbatim. The JSONata expression is the implementation — the test validates the transformation matches the spec's documented output table.

**Note:** The preset expressions evaluate via the existing `compileOrCached()` bridge from `expression/jsonata-bridge.ts`. Tests call `compile(preset.expression).evaluate(inputFixture)` and assert the output matches the spec's expected shape. Preset tests are async (JSONata is async).

- [ ] **Step 1: Implement registry + tests (TDD)**
- [ ] **Step 2: Implement prometheus preset + tests (TDD, using spec §4.1 fixtures for vector/matrix/scalar)**
- [ ] **Step 3: Implement elasticsearch preset + tests (TDD, using spec §4.2 fixture)**
- [ ] **Step 4: Implement graphql-relay preset + tests (TDD, using spec §4.3 fixture)**
- [ ] **Step 5: Implement jsonapi preset + tests (TDD, using spec §4.4 fixture)**
- [ ] **Step 6: Implement odata preset + tests (TDD, using spec §4.5 fixture)**
- [ ] **Step 7: Implement kubernetes preset + tests (TDD, using spec §4.6 fixture)**
- [ ] **Step 8: Run all preset tests**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/presets/`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat: add PresetRegistry with 6 built-in extraction presets  Refs #6
```

---

## Task 7: Extraction pipeline — parse, navigate, extract, tabulate, convert

**Files:**
- Create: `packages/core/src/dataset/external/extraction.ts`
- Create: `packages/core/src/dataset/external/extraction.test.ts`

This is the core pipeline: `FetchResult` → parse (JSON/CSV/metrics) → navigate (dataPath) → extract (type/expression) → tabulate (shape → DataSet) → convert (`toTypedDataSet`).

Tests cover:
- Shape A (explicit columns+values), Shape B (array of objects), Shape C (array of arrays)
- Content type detection (JSON, CSV, metrics via URL suffix)
- `dataPath` navigation
- `type` preset extraction
- `expression` custom JSONata
- Pipeline composition: dataPath + type + expression together
- Explicit columns vs inferred columns
- Column inference gate: infer from pipeline output, not raw response
- Inline content auto-detection (JSON first, CSV fallback)
- Error cases: PARSE_FAILED, EXTRACTION_ERROR, SCHEMA_MISMATCH, EMPTY_RESULT

- [ ] **Step 1: Write failing tests for shape detection and basic extraction**
- [ ] **Step 2: Implement extractDataSet with shape detection + tabulation + conversion**
- [ ] **Step 3: Add tests for dataPath navigation**
- [ ] **Step 4: Implement dataPath navigation**
- [ ] **Step 5: Add tests for preset extraction (type)**
- [ ] **Step 6: Implement type preset extraction**
- [ ] **Step 7: Add tests for custom expression extraction**
- [ ] **Step 8: Implement custom expression extraction**
- [ ] **Step 9: Add tests for pipeline composition (dataPath + type + expression)**
- [ ] **Step 10: Verify pipeline composition works (should pass from existing impl)**
- [ ] **Step 11: Add tests for content type detection (CSV, metrics)**
- [ ] **Step 12: Implement content type routing in parse stage**
- [ ] **Step 13: Add tests for column inference gate**
- [ ] **Step 14: Implement column inference with pipeline-output gate**
- [ ] **Step 15: Add tests for error cases**
- [ ] **Step 16: Run all extraction tests**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/extraction.test.ts`
Expected: ALL PASS

- [ ] **Step 17: Commit**

```
feat: implement extraction pipeline — parse, navigate, extract, tabulate, convert  Refs #6
```

---

## Task 8: DataProvider implementations

**Files:**
- Create: `packages/core/src/dataset/external/providers/inline.ts`
- Create: `packages/core/src/dataset/external/providers/inline.test.ts`
- Create: `packages/core/src/dataset/external/providers/cors-proxy.ts`
- Create: `packages/core/src/dataset/external/providers/cors-proxy.test.ts`
- Create: `packages/core/src/dataset/external/providers/browser-fetch.ts`
- Create: `packages/core/src/dataset/external/providers/server-relay.ts`
- Create: `packages/core/src/dataset/external/providers/post-message.ts`
- Create: `packages/core/src/dataset/external/provider-factory.ts`
- Create: `packages/core/src/dataset/external/provider-factory.test.ts`

**InlineProvider** and **CorsProxyProvider** are fully testable without browser APIs.

**BrowserFetchProvider**, **ServerRelayProvider**, and **PostMessageProvider** depend on browser globals (`fetch`, `window.postMessage`). These are typed stubs with the correct interface — their real behavior is tested in integration/browser tests, not unit tests. The provider factory is unit-testable.

- [ ] **Step 1: Write tests for InlineProvider (TDD)**
- [ ] **Step 2: Implement InlineProvider**
- [ ] **Step 3: Write tests for CorsProxyProvider (TDD — decorator wrapping a mock)**
- [ ] **Step 4: Implement CorsProxyProvider**
- [ ] **Step 5: Create BrowserFetchProvider, ServerRelayProvider, PostMessageProvider stubs**
- [ ] **Step 6: Write tests for DataProviderFactory (TDD — resolution logic)**
- [ ] **Step 7: Implement DataProviderFactory**
- [ ] **Step 8: Run all provider tests**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/providers/ src/dataset/external/provider-factory.test.ts`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```
feat: add DataProvider implementations and factory  Refs #6
```

---

## Task 9: Join

**Files:**
- Create: `packages/core/src/dataset/external/join.ts`
- Create: `packages/core/src/dataset/external/join.test.ts`

- [ ] **Step 1: Write failing tests**

Test cases:
- Two datasets with matching schemas → rows concatenated in order
- Three datasets → all rows concatenated
- Missing dataset → `DataSetError("UNKNOWN_PROVIDER")`
- Schema mismatch (different column types) → `DataSetError("SCHEMA_MISMATCH")`
- Schema mismatch (different column IDs) → `DataSetError("SCHEMA_MISMATCH")`
- Schema mismatch (different column count) → `DataSetError("SCHEMA_MISMATCH")`
- Single dataset join → returns its rows unchanged

- [ ] **Step 2: Run tests to verify they fail**
- [ ] **Step 3: Implement joinDataSets**
- [ ] **Step 4: Run tests to verify they pass**
- [ ] **Step 5: Commit**

```
feat: implement joinDataSets — vertical concatenation with schema validation  Refs #6
```

---

## Task 10: ExternalDataSetResolver — the orchestrator

**Files:**
- Create: `packages/core/src/dataset/external/resolver.ts`
- Create: `packages/core/src/dataset/external/resolver.test.ts`

The resolver ties everything together: validate → route (url/content/join) → fetch → extract → register/accumulate.

Tests use mock DataProviders (InlineProvider for content, a mock for url) and the real extraction pipeline + PresetRegistry.

- [ ] **Step 1: Write failing tests for url-based resolution**
- [ ] **Step 2: Write failing tests for content-based resolution**
- [ ] **Step 3: Write failing tests for join-based resolution**
- [ ] **Step 4: Write failing tests for accumulate path**
- [ ] **Step 5: Write failing tests for validation errors**
- [ ] **Step 6: Implement resolveExternalDataSet**
- [ ] **Step 7: Run all resolver tests**

Run: `yarn workspace @melviz/core run test -- src/dataset/external/resolver.test.ts`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```
feat: implement ExternalDataSetResolver orchestrator  Refs #6
```

---

## Task 11: Public API index and full test suite

**Files:**
- Create: `packages/core/src/dataset/external/index.ts`

- [ ] **Step 1: Create the public API barrel export**

```typescript
export type {
  ExternalDataSetDef,
  ExternalColumnDef,
  DataRequest,
  FetchResult,
  DataProvider,
  DataProviderConfig,
  ExtractionPreset,
  PresetRegistry,
  ExtractionResult,
  ResolveResult,
  MelvizDataMessage,
} from "./types.js";

export { HttpMethod } from "./types.js";
export { parseExternalDataSetDef } from "./schema.js";
export { parseCsv } from "./csv.js";
export { parseMetrics } from "./metrics-parser.js";
export { createPresetRegistry } from "./presets/registry.js";
export { extractDataSet } from "./extraction.js";
export { joinDataSets } from "./join.js";
export { resolveExternalDataSet } from "./resolver.js";
export { InlineProvider } from "./providers/inline.js";
export { CorsProxyProvider } from "./providers/cors-proxy.js";
export { createDataProviderFactory } from "./provider-factory.js";
```

- [ ] **Step 2: Run the full test suite**

Run: `yarn workspace @melviz/core run test`
Expected: ALL PASS — no regressions in existing tests, all new tests pass

- [ ] **Step 3: Run the build**

Run: `yarn workspace @melviz/core run build`
Expected: PASS — no type errors

- [ ] **Step 4: Commit**

```
feat: public API exports for external dataset module  Refs #6
```

---

## Task 12: Final integration verification and issue closure

- [ ] **Step 1: Run full test suite one final time**

Run: `yarn workspace @melviz/core run test`
Expected: ALL PASS

- [ ] **Step 2: Review git log for issue references**

All commits should reference `Refs #6`. The final commit should use `Closes #6`.

- [ ] **Step 3: Squash and push to fork**

Squash implementation commits, push to fork remote, close issue #6.
