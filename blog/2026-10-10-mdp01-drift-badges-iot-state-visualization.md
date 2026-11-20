---
layout: post
title: "Drift badges — IoT state visualization lands in Pages"
date: 2026-10-10
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [topology, iot, lit, web-components, sse]
---

# Drift badges — IoT state visualization lands in Pages

The IoT platform now knows when a device has drifted from its desired state — and whether that drift was permitted by a trigger or genuinely unexpected. The server-side pieces shipped in casehub-iot#130: a `TriggerSource` enum, a `DriftExemptionGrantedEvent`/`DriftExemptionRevokedEvent` CDI event pair, and a REST record carrying the drift info. What was missing was a way to see it.

I created a new `topology-viewer` package rather than adding to `pages-viz`. The reasoning: `pages-viz` components extend `PagesElement`, which pulls in the full dataset binding machinery — `DataSourceController`, data requests, refresh timers. A drift badge doesn't need any of that. It receives props directly from a parent component. Putting it in `pages-viz` would have coupled it to `pages-component`, the dataset pipeline, and ECharts for no benefit.

The badge itself is a Lit component — `pages-drift-badge` — that takes a `DriftStatus` enum, a trigger source string, and an ISO timestamp for the exemption deadline. For permitted drift, it shows an amber badge with the trigger name and a live countdown: "MOTION — 24m remaining". For unexpected drift, a red badge with a pulsing indicator dot. The countdown computes remaining time directly in `render()` rather than storing it in reactive state — a `setInterval` calls `requestUpdate()` every minute to trigger re-renders, but the actual text is always derived from `Date.now()` at render time. This avoids the double-render cycle you get when `updated()` sets `@state()` properties.

The SSE integration uses `onPagesEvent` from `pages-data` with colon-separated topics — `drift:exemption:granted` and `drift:exemption:revoked` — following the pages-event contract. A `subscribeDriftEvents()` function wraps the two subscriptions and returns a single unsubscribe handle.

One thing I caught during review: I'd used `const enum` for `DriftStatus`, which was the only `const enum` in the entire codebase. Everything else uses regular `enum`. With `isolatedModules: true` in the base tsconfig, `const enum` across module boundaries doesn't get inlined by esbuild or vite — it just silently falls back to a regular enum anyway. Changed it to match the codebase convention.

The topology viewer is thin right now — types, a badge, and event wiring. The full node renderer and layout engine will follow as the IoT topology page takes shape.
