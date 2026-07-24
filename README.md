# CMS architecture — validation

Design proposal for a **client-safe CMS layer** (annotated templates + typed
content) to be **validated before implementation**.

- **[cms-greenfield-architecture.md](cms-greenfield-architecture.md)** — the full
  greenfield conception: the two-layer model (structure ⟂ typed content), the
  annotation convention, the content/schema data model, the pipelines
  (onboarding import → render/merge → static export → reconciled re-ingest),
  the editor model, JS/animation handling, storage, a robustness/fragility
  analysis, and prior-art mapping (Shopify OS 2.0, Instatic).

## Where to focus when validating

- **§0 — TL;DR** — the thesis in one paragraph.
- **§11 — Open questions** — concrete technical points to stress-test.
- **§12 — Limits & when to choose otherwise** — the honest scope: what the model
  does NOT answer (commerce, content-graph, DIY-no-dev, design freedom, i18n),
  the sharpest internal limit (GSAP × content-reflow), and **§12.5 the demand
  question** — the most important, non-technical thing to validate before writing
  any code.

The document is self-contained: a reviewer (human or AI) can read it cold,
without the originating conversation.

> Note: the doc is written in Hungarian.
