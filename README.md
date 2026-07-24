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
  an **i18n extension**, and an independent **conditional validation verdict** with
  implementation gates.
- **[go-to-market.md](go-to-market.md)** — the business model: target audience, the
  HU-calibrated pricing (build + operation), the pricing levers, a **sourced competitor
  analysis** (HU agencies + international builders), the future self-serve direction, and
  the business validation questions.

## Where to focus when validating

- **Tech — §11** open questions · **§12** limits & when to choose otherwise (§12.5 the
  demand question) · **§14.5** reflow-safe animation · **§15** independent review and
  implementation gates. **§15.3** phases the work (v1 minimal spike → production pilot →
  Phase 2 composable); **§15.7** records the resolved v1 concurrency and isolated
  Edit/Preview decisions.
- **§16 — review round 2 (adversarial)** — discharges the three CRITICAL blockers a
  second adversarial review surfaced: **§16.1** the single canonical content schema
  (instance-scoped field identity, stable block ids, JSON Schema before the spike);
  **§16.2** the complete reconciliation/migration protocol (rename/move/transform/
  split/merge/delete precedence, name-swap no longer silently corrupts, dry-run diff);
  **§16.3** the full render sink list + import isolation (CSS-content sink, richtext
  href policy, SVG resolution, SSRF egress-allowlist, full published CSP + nosniff,
  postMessage origin/schema); **§16.4** a v1 revision-guard closing the silent
  lost-update gap; **§16.5** the store wording; **§16.6** sequencing — run the demand
  gate before the technical spike.
- **Business — §6** the validation questions (demand, real hours, recurring acceptance).

Docs are self-contained: a reviewer (human or AI) can read them cold, without the
originating conversation.

> Note: both docs are written in Hungarian.
