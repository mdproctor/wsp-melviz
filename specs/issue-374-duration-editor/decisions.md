## D1: Default field set

**Choice:** Hours, Minutes, Seconds (H/M/S)
**Alternatives:**
- Days, Hours, Minutes, Seconds — adds days for longer durations but wider layout
- All six (Y/M/D/H/M/S) — full ISO 8601 but calendar-ambiguous Y/M units, very wide
**Rationale:** Most duration use cases are elapsed time. H/M/S is the common case. Days/months/years can be added via the fields property.
**Trade-offs:** Consumers needing days or longer must set the fields property explicitly.
**Sources:** ISO 8601 duration format, pages-ui-components existing controls
**Exploration:** quick
**Status:** captured

## D2: Field configuration API

**Choice:** `fields` array property (e.g. `['hours', 'minutes', 'seconds']`)
**Alternatives:**
- Boolean per unit (showDays, showYears, etc.) — more discoverable but verbose API
- Preset string (mode='time') — simpler but inflexible for arbitrary combos
**Rationale:** Explicit, composable, matches the pattern of other array-property controls. Consumer lists exactly the units they want.
**Trade-offs:** Slightly less discoverable than boolean properties in IDE autocomplete.
**Sources:** pages-tag-editor value array pattern, pages-slider min/max pattern
**Exploration:** quick
**Status:** captured

## D3: Zero-value serialization

**Choice:** Omit zero-valued units from the ISO 8601 string
**Alternatives:**
- Include all visible fields — more predictable round-tripping but noisier output
**Rationale:** Standard ISO 8601 practice. `PT1H30M` is cleaner than `PT1H30M0S`. Empty duration serializes as `PT0S`.
**Trade-offs:** If a consumer round-trips through the control, zero-valued fields won't appear in the string even if they were visible in the UI.
**Sources:** ISO 8601 duration specification
**Exploration:** quick
**Status:** captured

## D4: Fractional seconds

**Choice:** Integer-only seconds
**Alternatives:**
- Allow fractional seconds (step=0.1 or similar) — adds precision but uncommon in property editing
**Rationale:** Fractional seconds are rare in property-panel contexts. All number inputs use integer step=1 by default.
**Trade-offs:** Cannot represent sub-second precision. The custom resolver mechanism allows consumers to provide a domain-specific high-precision duration editor.
**Sources:** Issue #374 spec, property palette custom resolver docs
**Exploration:** quick
**Status:** captured
