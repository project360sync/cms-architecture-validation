# CMS architecture — validation

Design + business proposal for a **client-safe CMS layer** (annotated templates +
typed content), used by *us as the agency* to build bespoke animated sites clients
can self-edit. **To be validated before implementation.**

## Documents

- **[cms-greenfield-architecture.md](cms-greenfield-architecture.md)** — the technical
  conception: the two-layer model (structure ⟂ typed content), the annotation
  convention, the content/schema data model, the pipelines (onboarding import →
  render/merge → static export → reconciled re-ingest), the editor model, JS/animation
  handling, storage, robustness analysis, prior-art mapping (Shopify OS 2.0, Instatic),
  and an **i18n extension**.
- **[go-to-market.md](go-to-market.md)** — the business model: target audience, the
  HU-calibrated pricing (build + operation), the pricing levers, a **sourced competitor
  analysis** (HU agencies + international builders), the future self-serve direction, and
  the business validation questions.

## Where to focus when validating

- **Tech — §11** open questions · **§12** limits & when to choose otherwise (§12.5 the
  demand question) · **§14.5** reflow-safe animation.
- **Business — §6** the validation questions (demand, real hours, recurring acceptance).

Docs are self-contained: a reviewer (human or AI) can read them cold, without the
originating conversation.

> Note: both docs are written in Hungarian.
