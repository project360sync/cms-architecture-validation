# CMS architecture — validation

Design proposal for a **client-safe CMS layer** (annotated templates + typed
content) to be **validated before implementation**.

- **[cms-greenfield-architecture.md](cms-greenfield-architecture.md)** — the full
  greenfield conception: the two-layer model (structure ⟂ typed content),
  the annotation convention, the content/schema data model, the pipelines
  (onboarding import → render/merge → static export → reconciled re-ingest),
  the editor model, JS/animation handling, storage, a robustness/fragility
  analysis, prior-art mapping (Shopify OS 2.0, Instatic), the diff from the
  current implementation, and **§11 — concrete open questions to stress-test.**

The document is self-contained: a reviewer (human or AI) can read it cold,
without the originating conversation. Start at §0 (TL;DR) and §11 (open questions).

> Note: the doc is written in Hungarian.
