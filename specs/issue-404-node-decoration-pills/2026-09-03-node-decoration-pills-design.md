# Node Decoration Pills

Additive extension to `NodeDecoration` in `graph-core` and `graph-renderer`
for rendering runtime overlay labels on graph nodes.

## Motivation

`blocks-ui#150` needs trust scores on worker nodes in the case flow viewer.
`NodeDecoration` currently has `badge`, `border`, `overlay`, and `tooltip` —
no mechanism for multiple supplementary text labels. Injecting runtime data
into `GraphNode.properties` conflates definition data with runtime visual state.

## Changes

### graph-core (`model.ts`)

Add `pills` to `NodeDecoration` as an inline readonly type, matching the
existing pattern for `badge`, `border`, and `overlay`:

```typescript
export interface NodeDecoration {
  readonly badge?: { ... };   // existing
  readonly border?: { ... };  // existing
  readonly overlay?: { ... }; // existing
  readonly tooltip?: string;  // existing
  readonly pills?: readonly {
    readonly text: string;
    readonly color: string;
    readonly icon?: string;
  }[];
}
```

Each pill has:
- `text` — the label (e.g. "0.92", "45ms", "SLA: 2h")
- `color` — CSS color for the pill background
- `icon` — optional leading character/emoji

The array is ordered — pills render left-to-right in array order.

### graph-core (`model.test.ts`)

Add test cases for pills:
- Decoration with pills array (multiple pills)
- Decoration with single pill (no icon)
- Decoration with pills and other fields (badge + pills coexistence)
- Empty pills array

### graph-renderer (`stencil-wrapper.tsx`)

Add a `DecorationPills` React component following the `DecorationBadge` pattern:

```tsx
function DecorationPills({ pills }: { pills: NonNullable<NodeDecoration['pills']> }): React.JSX.Element
```

Renders a horizontal flex row of pill elements at the bottom of the
`.stencil-decoration-wrapper`, positioned absolute at `bottom: -10px`.
Each pill is a small rounded element showing optional icon + text, with
background set to the pill's `color` and white text.

Pill styling:
- `display: inline-flex`, `gap: 2px` between icon and text
- `border-radius: 8px`, `padding: 1px 6px`
- `font-size: 9px`, `font-weight: 600`
- `color: #fff` on the pill's background color
- Container: `display: flex`, `gap: 3px`, `justify-content: center`

Render order in the wrapper (matches existing structure):
1. `DecorationBadge` (top-right, absolute)
2. `DecorationOverlay` (full coverage, absolute)
3. Stencil content (`<div ref={containerRef} />`)
4. **`DecorationPills`** (bottom edge, absolute) — NEW

### graph-renderer (`stencil-wrapper.test.tsx`)

Add rendering tests:
- Pills render when decoration has pills
- Each pill shows text and optional icon
- Multiple pills render in order
- No pills element when pills array is absent or empty

### graph-renderer (`mapping.test.ts`)

Add test verifying pills pass through decoration mapping unchanged.

## Data Flow

No changes to the existing data flow. Pills travel through the same path
as all other decoration fields:

1. Consumer provides `Map<string, NodeDecoration>` with pills in entries
2. `toReactFlowNode()` packs decoration into `data._decoration`
3. `StencilNode` extracts `_decoration` and passes to `DecorationPills`

## Scope

- Additive, non-breaking — all fields optional, no existing API changes
- Two packages touched: `graph-core` (type), `graph-renderer` (rendering)
- No new dependencies
- No export changes beyond the existing `NodeDecoration` re-export

## References

- casehubio/casehub-pages#404 — this issue
- casehubio/blocks-ui#150 — trust score rendering (blocked by this)
- packages/graph-core/src/model.ts:22 — NodeDecoration interface
- packages/graph-renderer/src/stencil-wrapper.tsx:50-76 — DecorationBadge pattern
- docs/protocols/casehub/graph-core-pure-data.md — pure data constraint (pills is a data type, compliant)
