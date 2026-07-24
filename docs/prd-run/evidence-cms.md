# Evidence inventory — claude-cms codebase (feasibility / Gate B input)
Source: /Users/jarvis/Documents/projects/claude-cms/.claude/worktrees/vibor-gsap-cms (working prototype; PREDECESSOR position-anchored model of the planned greenfield rewrite).

[CMS docs/adr/ADR-003-author-js-and-editable-collections.md:30] DECISION: author JS allowed as locked, external, same-origin <script src> bundles only — inline + on* stripped, CSP script-src 'self', every captured bundle SRI hash-pinned at ingest.
[CMS docs/adr/ADR-003:42] DECISION: content ships as static HTML in the document; JS may only animate/toggle/wire existing DOM, never render content.
[CMS docs/adr/ADR-003:49] DECISION: editable collections via data-cms-collection/data-cms-item + addItem EditOp cloning vetted skeleton markup.
[CMS docs/adr/ADR-003:92] DECISION: editor Edit/Preview toggle — Edit strips bundles (stable DOM for click-to-edit), Preview injects read-only; live serving always includes bundles.
[CMS src/server/services/ingest.ts:164] Proven: external <script src> captured as same-origin /a/<id> assets, before image flood (asset-cap ordering fix).
[CMS src/server/services/ingest.ts:166 + ingest.test.ts:160] Proven: SRI integrity="sha256-…" pinned per captured bundle from exact stored bytes.
[CMS src/server/services/sanitize.ts] Proven: allowlist sanitize — inline scripts + on* dropped, incoming data-* stripped before annotation.
[CMS src/server/services/apply-ops.ts:147 + guardian.ts:47] Proven: addItem clones template subtree from stored sanitized skeleton (never client markup), mints fresh ids, Guardian-gated, allowed in safe/client mode.
[CMS src/components/editor/EditorShell.tsx:102] Proven: Edit/Preview toggle in the editor (mode=preview reload with bundles).
[CMS src/server/serving.ts:3] Proven: published CSP default-src 'self'; script-src 'self'; base-uri 'none'; frame-ancestors 'self' — SRI-pinned same-origin bundles run, inline blocked.
[CMS src/server/services/publish.ts:14,140] Proven: immutable publish snapshots (version freeze + atomic pointer flip) and one-click rollbackSite().
[CMS package.json / vitest] Proven: 19 test files, 176 tests, ALL PASSING (~3.1s); separate Playwright e2e (e2e/full-flow.spec.ts).
[CMS DESIGN.md:10-13] Invariants: ONE mutation path (applyOps behind Guardian); client edits are typed data never markup; published output immutable + pointer-flipped; MongoDB(+GridFS) single source of truth.
[CMS README.md:44] Zero-config boot: in-memory Mongo auto-seeds demo site through the REAL ingestion pipeline.
[CMS note] The vibor GSAP site is the ADR's motivating example but NOT an in-repo fixture — the proven pipeline runs against the Brew&Bloom demo fixture + unit tests; GSAP feasibility proven at capability level (external-script capture + SRI + CSP), not against the literal vibor site.

NOT-PROVEN (greenfield open unknowns — Gate B must attack these, this codebase does not):
[NP-1] Name-keyed content: ids are POSITION-anchored (t<n>/sec<n>/col<n> by document order — annotate.ts:164); no stable name/semantic keys exist.
[NP-2] Re-ingest reconciliation: unimplemented; DESIGN.md warns re-annotation would orphan the overrides.
[NP-3] GSAP-survives-content-reflow under length variance: no test/fixture exercises real GSAP under content-length change or addItem count change.
[NP-4] Composable mode: no composition/prototype-registry system.
[NP-5] Multi-locale: no locale dimension anywhere (content model, registry, overrides, snapshots).
