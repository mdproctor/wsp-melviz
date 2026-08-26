---
layout: post
title: "Six Units, Three Inputs, One String"
date: 2026-08-26
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [form-controls, iso-8601, lit, pages-ui-components]
series: issue-374-duration-editor
---

# Six Units, Three Inputs, One String

ISO 8601 durations look simple until you try to edit them. `PT1H30M` is fine as a string — but asking a user to type `PT1H30M` into a text input is asking for syntax errors. The property palette's default resolver needed a proper control for `format: "duration"` schemas, and the parent spec (#373) explicitly deferred it because a multi-field compound UI is a different beast from the single-native-input controls like date and number.

The interesting design choice was the field set. ISO 8601 durations span six units — years, months, days, hours, minutes, seconds — but years and months are calendar-ambiguous (a month is 28–31 days depending on context). I defaulted to hours/minutes/seconds with a `fields` array property for consumers who need more. The array determines which sub-inputs render and in what order, and hidden units are silently dropped on round-trip. If you set `value="P1YT2H"` but only show `['hours']`, the year vanishes when the value serializes back — the control edits only what it shows.

The implementation reused the "plain `<input>` to avoid side-effect registration" pattern from `pages-slider`. Each sub-input is a raw `<input type="number" min="0">` with a short unit label below — `h`, `m`, `s`. Not a `<pages-number-input>` custom element, because importing the slider shouldn't force-register `pages-number-input` as a side effect. The same reasoning applies here.

Serialization omits zero-valued units — `PT1H30M` not `PT1H30M0S` — with the canonical empty duration being `PT0S`. The regex parser handles the full ISO 8601 duration grammar but only populates the fields the consumer asked for.

The resolver change was a single line: `format: "duration"` maps to `pages-duration-input`, alongside the existing `date` and `date-time` mappings. The palette's barrel import picks up the side-effect registration. 19 tests cover parsing, serialization, field configuration, ARIA, and readonly/disabled states.

What this opens up: the palette now handles every editor type from the original #373 spec. The remaining follow-ups are #375 (migrating `PagesSchemaForm` to embed the palette) and the blocks-ui migration (which needs the palette published to `.casehub-packages` first).
