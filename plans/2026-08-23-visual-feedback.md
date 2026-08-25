# Visual Feedback Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use executing-plans to
> implement this plan task-by-task. Each task follows TDD.

**Focal issue:** #351 — Visual feedback for automated UI interactions
**Issue group:** #351 (part of epic casehubio/parent#408)

**Goal:** Add highlights, focus rings, and typing animation to ARIA
executor commands so audiences can see what's happening during scenarios.

**Architecture:** New `visual-feedback.ts` module provides `injectStyles()`,
`highlightElement()`, and `typeText()`. scenario-handler.ts calls these
before each command. Executor stays pure.

**Tech Stack:** TypeScript, DOM API, CSS animations

## Global Constraints

- Zero new dependencies
- Executor (`command-executor.ts`) must not change
- CSS injected into document head, not shadow DOM
- Typing speed: ~40ms per character default

---

## Batch 1: Visual feedback module + integration

### Task 1: visual-feedback.ts — style injection, highlight, typeText

**Files:**
- Create: `packages/pages-aria/src/executor/visual-feedback.ts`
- Test: `packages/pages-aria/src/executor/visual-feedback.test.ts`

**Interfaces:**
- Produces:
  - `injectStyles(): void`
  - `highlightElement(el: Element, type: 'click' | 'fill' | 'select'): void`
  - `typeText(el: HTMLInputElement | HTMLTextAreaElement, value: string, speed?: number): Promise<void>`

### Task 2: Wire visual feedback into scenario-handler

**Files:**
- Modify: `packages/pages-aria/src/server/scenario-handler.ts`

**Interfaces:**
- Consumes: `injectStyles`, `highlightElement`, `typeText` from visual-feedback
- Consumes: `resolveTarget` from command-executor (already exported)

## References

- [2026-08-23-visual-feedback-design.md] — design spec
- `packages/pages-aria/src/executor/command-executor.ts` — pure executor
- `packages/pages-aria/src/server/scenario-handler.ts` — orchestration
- D26, D27 in decisions.md
- casehubio/casehub-pages#351
