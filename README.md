# CMS architecture — validation

Design proposal for a **client-safe CMS layer** (annotated templates + typed
content) to be **validated before implementation**.

- **[cms-greenfield-architecture.md](cms-greenfield-architecture.md)** — the full
  greenfield conception: the two-layer model (structure ⟂ typed content), the
  annotation convention, the content/schema data model, the pipelines
  (onboarding import → render/merge → static export → reconciled re-ingest),
  the editor model, JS/animation handling, storage, a robustness/fragility
  analysis, prior-art mapping (Shopify OS 2.0, Instatic), an **i18n extension**,
  and an independent **conditional validation verdict** with implementation gates.

## Where to focus when validating

- **§0 — TL;DR** — the thesis in one paragraph.
- **§11 — Open questions** — concrete technical points to stress-test.
- **§12 — Limits & when to choose otherwise** — the honest scope: what the model
  does NOT answer, the sharpest internal limit (GSAP × content-reflow), and
  **§12.5 the demand question** — the most important, non-technical thing to
  validate before writing any code.
- **§14 — i18n** — how multi-language fits as a natural extension (locale-scoped
  content, fallback + staleness, AI-translation), and **§14.5 reflow-safe
  animation** — mitigation for the reflow limit: animate length-tolerant sections,
  keep motion restrained, constrain sensitive slots, and test locale/viewport states.
- **§15 — independent review** — conditional validation, P0/P1 implementation gates,
  and the smallest vertical pilot that can prove or disprove the proposal.

The document is self-contained: a reviewer (human or AI) can read it cold,
without the originating conversation.

> Note: the doc is written in Hungarian.
