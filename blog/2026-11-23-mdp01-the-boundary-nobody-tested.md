---
layout: post
title: "The Boundary Nobody Tested"
date: 2026-11-23
entry_type: note
subtype: diary
projects: [casehubio/casehub-pages]
tags: [scanner, testing, yaml-core, match]
series: issue-741-scanner-match-guard-tests
---

# The Boundary Nobody Tested

The `scanMatchCases` function in `StepPositionScanner` handled three things: simple `when:` cases, `default:` cases, and cases with `guard:` keys or nested control flow. The first two had tests. The third had code that looked correct — it skipped metadata keys (`when`, `pattern`, `guard`), found the first action key, and recursed into it if the key was a control flow keyword like `if` or `try`.

I filed issue #741 to close the gap. Two tests: one for a match case with a `guard:` key, one for a match case containing a nested `if`/`then`/`else`.

The guard test passed immediately. The control flow test failed.

## What went wrong

The scanner computes step boundaries to know how much YAML belongs to a given step. In `scanBlock` — the function that handles normal step lists — a step is a list item (`- if: condition`), and `findNextSibling` looks for the next list item at the same indent to find where the step ends. Since `then:` and `else:` aren't list items, they're correctly included inside the step boundary.

But `scanMatchCases` doesn't work with list items in the same way. Inside a match case, the action key (`if: condition`) sits at the case's content indent, and its structural siblings (`then:`, `else:`) sit at the same indent. When the code called `findBranchEnd` to compute the step boundary, it hit `then:` immediately — same indent, boundary triggered. The step ended at line 4 instead of line 8.

The second problem cascaded from the first. `scanControlFlow` adds 2 to its `listIndent` parameter to get the indent where it should look for branch keywords like `then:` and `else:`. The code was passing `caseContentIndent` (6), so the function looked for `then:` at indent 8 — but `then:` was at indent 6.

```yaml
    - when: open         # indent 4 — case list item
      if: condition      # indent 6 — caseContentIndent
      then:              # indent 6 — same level, not deeper
        - action: yes    # indent 8
      else:              # indent 6
        - action: no     # indent 8
```

The scanner found the `if` step, recorded it as one line long, tried to recurse into it, and found nothing because the range was already exhausted.

## The fix

Two changes on the same two lines:

1. Use `caseEnd` instead of `findBranchEnd` for the step boundary. Since each case produces at most one action step (enforced by a `break`), the case boundary is the correct step boundary. This is safe for simple cases too — `findBranchEnd` was already returning `caseEnd` in those cases because there were no sibling keys to stop at.

2. Pass `caseListIndent` instead of `caseContentIndent` to `scanControlFlow`. This matches how `scanNamedBranches` handles the identical pattern for `select` branches — the indent parameter needs to be 2 less than the step key's indent so that the `+2` inside `scanControlFlow` lands at the right level.

## The pattern

The `select` scanner already handled this correctly. Its control flow dispatch at line 310 passes `nameIndent` — which is the list indent, not the content indent. The match scanner was doing the same job with the wrong parameter. The fix is alignment, not invention.

What's interesting is that the code *looked* right. It checked for control flow keywords. It called `scanControlFlow`. The recursion path existed. But the boundary computation made the recursion a no-op — `scanControlFlow` received a one-line range with no room for branches. The only way to find this was to run a test that actually exercised the path and checked the output.

Code paths without tests aren't "untested but working." They're untested and unknown.
