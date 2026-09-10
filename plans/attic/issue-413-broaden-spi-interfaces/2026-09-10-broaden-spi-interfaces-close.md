# Work Plan — issue-413-broaden-spi-interfaces

## State
branch: issue-413-broaden-spi-interfaces
state: closing:promoted
project-sha: 6809c1d55a4b28333b1ab24402577c4847f778e7
date: 2026-09-07
issue-repo: casehubio/casehub-pages
covers: 413
design-repo: workspace
design-section-hashes: 
flyway-next-v: none

evidence-era: true
## Queue
- [ ] casehubio/casehub-pages#413 — Issue #413 ← active
  - [ ] Batch 1: Foundation
    - [ ] Task 1: Extract ChartSettingsBase
    - [ ] Task 2: Widen PagesChartElement and refactor applyChartSettings
  - [ ] Batch 2: SPI broadening
    - [ ] Task 3: Add ChartSettingsBase shared properties
    - [ ] Task 4: Add Cartesian promotions
    - [ ] Task 5: Per-chart series promotions
  - [ ] Batch 3: Component broadening
    - [ ] Task 6: Map Graph DensityHeatmap promotions
    - [ ] Task 7: Typed escape hatch extension types
  - [ ] Batch 4: Schema validation
    - [ ] Task 8: Regenerate schemas and validate pipeline
  - [ ] Batch 5: GraphCanvas
    - [ ] Task 9: GraphCanvasProps interface and registration
    - [ ] Task 10: PagesGraphCanvas YAML bridge
