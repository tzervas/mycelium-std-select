# Spec — `std.select` (the reified selection-policy surface + the mandatory EXPLAIN record)

| Field | Value |
|---|---|
| **Status** | **Accepted** (2026-06-20, maintainer-ratified per DN-07 — guarantee matrix asserted in tests; open §7/§8 questions are design/scope calls, not contract violations; was *Implemented (Rust-first) — pending ratification* 2026-06-18, Draft/needs-design 2026-06-17) — the Rust-first code landed as `mycelium-std-select` (M-519, Batch P5-A; guarantee matrix asserted in tests). The Mycelium-lang migration (M-502-gated) remains. |
| **Module / Ring** | `std.select` (with `explain`) · Ring `1` (RFC-0016 §4.2 — capability surface) · Tier `A` |
| **Tracks** | `M-519` (#161) — the Phase-5 task this spec delivers (`select` / `explain`: selection DSL + mandatory EXPLAIN). |
| **Scope** | The ergonomic library home of the **total, non-learned, content-addressed** selection policy and the **mandatory EXPLAIN record**: build/inspect a `SelectionPolicy`, run a selection over a finite candidate set against queryable `Meta`, and obtain the reified `Explanation` for *every* selection — the one mechanism many sites (swap-target, packing-schedule) share. |
| **Boundary** | This module owns *deciding among a finite candidate set and explaining the decision*. It does **not** own *performing* the chosen action: the representation change is `std.swap` (M-516); the packing-schedule consumption is the M-250 path. Content-addressing primitives are `std.content` (M-523); diagnostic presentation of an EXPLAIN record is `std.diag` (M-510). |
| **Depends on** | **RFC-0005 (Accepted)** — the total cost-based policy + mandatory EXPLAIN this module is the library form of; **ADR-006** (reified, no-black-box policies); RFC-0016 §4.1 (the C1–C6 contract); RFC-0001 (`SelectionPolicy`/`PolicyRef`/`Meta.policy_used`, the value model, content-addressing §4.6). |
| **Grounds on** | the landed `mycelium-select` crate (M-220 `SelectionPolicy` + closed `Predicate` language + `CostModel` in real storage **bits** + `policy_ref()`; M-221 the `Explanation` emitted on *every* `select`; M-222 the single `select(policy, candidates, meta)` adapter behind both sites). KC-3: this module is **above** the kernel — an EXPLAIN **consumer/re-exporter**, no new trusted code. |

---

## 1. Summary

`std.select` is the standard-library surface over the landed `mycelium-select` capability crate (M-220/221/222): a **total, terminating, non-learned, content-addressed** decision-table policy and — the crux — the **mandatory `Explanation` record** every selection emits. The user-facing surface is `select(policy, candidates, meta) → (choice, explanation)`, `explain(policy, meta) → trace`, `build`/validate a policy, and a first-class `override`. Its **honesty crux** is C3 / SC-3 (no black boxes): there is **no selection without an EXPLAIN** — a selection that returned a choice but no inspectable record is structurally impossible in this surface, and the policy itself is a content-addressed value one can always inspect to answer *"which policy chose this, and what does it do?"* (RFC-0005 §3, G2). This is a Ring-1 capability surface; it adds **no trusted code** (KC-3) — it consumes and re-exports the landed crate.

## 2. Scope & module boundary

- **In scope:** the exported `SelectionPolicy` type (a finite ordered decision table over a closed predicate language) and its validating constructor; the `CostModel` expressed in **real declared units** (storage bits — *never* "arbitrary internal units", RFC-0005 §2 black-box-mode (1)); the `select` op (choice + mandatory `Explanation`); the `explain(policy, meta) → trace` capability (RFC-0005 §4); the first-class `Override` (a forced choice that is *itself* recorded); and `PolicyRef`, the policy's content hash (RFC-0001 §4.6, ADR-003).
- **Out of scope (and who owns it):** *performing* the swap selected — `std.swap` (M-516); *consuming* a packing schedule — the M-250 packing path (RFC-0005 §4 names the site; this module decides, it does not pack); content-addressing machinery as a first-class library — `std.content` (M-523); rendering an `Explanation` as a human diagnostic — `std.diag` (M-510, additive presentation over the explicit record, I1). The policy language is deliberately **not** Turing-complete; a general scripting surface is explicitly *not* offered (RFC-0005 §2 — the expressiveness ceiling is the feature).
- **Ring & layering:** Ring 1 (RFC-0016 §4.2). It is an EXPLAIN **consumer** that re-exports / wraps `mycelium-select`; it builds no new trusted code and confines nothing to `wild` (no FFI). KC-3 holds: the policy lattice and the `Explanation` schema are owned by the landed crate, not enlarged here.

## 3. Exported-op surface (design sketch)

A signature sketch — value-semantic, immutable-by-default. `select`/`build` are fallible (`Result`); `explain` is total over a *valid* policy. Effects are none (pure functions of `policy` + `meta` + `candidates`). This is a DESIGN sketch to fix the surface and feed §4, not a committed grammar; the concrete `Explanation` field names are owned by the landed `mycelium-select` crate (M-221) and are described abstractly here — see (Q1).

```
// illustrative signatures (not a committed surface)

// A policy is a finite, ordered decision table over a closed predicate language,
// carrying a CostModel in REAL declared units (storage bits). Content-addressed.
type SelectionPolicy            // value; identity = PolicyRef (content hash)
type PolicyRef = ContentHash    // RFC-0001 §4.6 / ADR-003 — policy identity
type CostModel                  // candidate -> cost in storage BITS (declared units)

// Build is fallible: a malformed table (e.g. non-total — no default arm) is refused.
fn build(rules, cost: CostModel) -> Result<SelectionPolicy, PolicyError>
fn policy_ref(p: &SelectionPolicy) -> PolicyRef

// The one mechanism, many sites. EVERY select emits an Explanation — not optional.
fn select(p: &SelectionPolicy, candidates: &[Candidate], meta: &Meta)
        -> Result<(Choice, Explanation), SelectError>

// The explain capability (RFC-0005 §4): total + deterministic over a valid policy.
fn explain(p: &SelectionPolicy, meta: &Meta) -> Explanation

// First-class deterministic override — and the override state is recorded IN the Explanation.
fn select_with_override(p: &SelectionPolicy, candidates: &[Candidate],
                        meta: &Meta, forced: Choice)
        -> Result<(Choice, Explanation), SelectError>

// Explanation (abstract — exact field names owned by mycelium-select / M-221; see Q1):
//   { inputs considered (the queried Meta), per-candidate cost (in bits),
//     chosen option, override state } — RFC-0005 §2.2.
```

## 4. Guarantee matrix (the load-bearing deliverable — RFC-0016 §4.5)

Rows = exported ops. Encoded as a checked table (the RFC-0003 §4 template), asserted in tests once code lands — never prose only. The honesty crux of this module lives in the final column: **every selection op is EXPLAIN-able = yes** (it emits a reified, inspectable `Explanation`).

| Op | Guarantee tag | Fallibility (explicit error set) | Declared effects | EXPLAIN-able? |
|---|---|---|---|---|
| `build` (validate a policy) | `Exact` | `Err(PolicyError)` — a non-total / malformed table (e.g. missing default arm) is refused, never silently completed | none | n/a (constructs the policy; produces no selection) |
| `policy_ref` (content hash) | `Exact` | total — a valid policy always has a `PolicyRef` | none | n/a (identity op, not a selection) |
| `select` | `Exact` | `Err(SelectError)` — an empty candidate set, or a wrong-kind candidate (`WrongSiteKind`, M-222), is an explicit refusal, never a coercion | none (pure of `policy`+`meta`+`candidates`) | **yes** — emits a valid `Explanation` (RFC-0005 §2.2) |
| `explain` | `Exact` | total over a valid policy (`explain(policy, meta)` is total + deterministic, M-221) | none | **yes** — the `Explanation` / trace *is* the artifact |
| `select_with_override` | `Exact` | `Err(SelectError)` as `select`; a forced choice outside the candidate set is an explicit refusal | none | **yes** — the override state is recorded *in* the `Explanation` (an overridden selection stays inspectable, M-221) |

**Tag justification (VR-5).** Every row is `Exact` and that is the honest tag — *not* a downgrade and *not* an overclaim. The policy is a **total predicate over EXACT metadata** (proven/declared bounds, `dtype`, sparsity class — RFC-0005 §2.5): same `Meta` in → same choice out, deterministically (RFC-0005 §2.3). Mycelium structurally **avoids the cardinality-estimation trap** — its "statistics" are exact metadata, not sampled estimates, so the dominant source of cost-optimizer opacity does not arise (RFC-0005 §2.5). There is **nothing probabilistic, learned, or estimated** in this module; claiming `Empirical`/`Declared` accuracy here would be dishonest in the *other* direction. The `Exact` tag is over the *selection decision* (the table is total over exact inputs), **not** a claim about the downstream operation's numerical accuracy — that op carries its own tag in its own module (e.g. `std.swap`'s certificate, `std.dense`'s ε bound).

## 5. §4.1 contract conformance (C1–C6)

- **C1 — never-silent (G2):** `select` over an empty candidate set, and a wrong-kind candidate, are explicit `Err` (`WrongSiteKind`, M-222) — never a coercion, clamp, or silent default-to-first. `build` refuses a non-total table rather than completing it with a hidden fallback. An override outside the candidate set is refused, not snapped to the nearest legal choice.
- **C2 — honest per-op tag (VR-5):** every op is `Exact` and §4 justifies why — a total table over exact metadata. The module makes **no** probabilistic/learned/estimated claim (RFC-0005 §2 — non-learned by construction); tags are not upgraded above the checked basis, and the selection's `Exact` tag is explicitly scoped to the *decision*, not the downstream op's accuracy.
- **C3 — no black boxes / EXPLAIN (SC-3/G11):** **this is the module's reason to exist.** Every selection emits a valid, inspectable `Explanation` — there is no `select` path that returns a choice without one (M-221: "no selection without an EXPLAIN"). The record cites the per-candidate **cost in real declared units (storage bits)**, the inputs considered, the chosen option, and the override state (RFC-0005 §2.2). The policy itself is an inspectable value, and `explain(policy, meta)` answers *"why this choice?"* off the policy alone. No opaque heuristic decides a user-visible outcome.
- **C4 — content-addressed, value-semantic (ADR-003):** a `SelectionPolicy` is an immutable value whose identity is its `PolicyRef` content hash (RFC-0001 §4.6); `Meta.policy_used` records that hash at every selection site, so the deciding policy is recoverable from the value alone (RFC-0005 §3). `select`/`explain` are pure functions of their inputs — content-addressing → determinism + auditability (same policy hash + same `Meta` → same choice). Metadata is not identity.
- **C5 — above the kernel (KC-3):** the policy lattice, the closed `Predicate` language, the `CostModel`, and the `Explanation` schema are owned by the landed `mycelium-select` crate (the M-220 §5 decision that selection stays *out* of the trusted kernel). This module re-exports / wraps them; it adds no trusted code and uses no `wild`/FFI.
- **C6 — declared, bounded effects (RFC-0014):** all exported ops are **pure** — no IO, time, randomness, or unbounded allocation. The candidate set is finite and the predicate language is terminating by construction (RFC-0005 §2), so selection is total and needs no effect budget. Nothing to declare beyond `none`.

## 6. Grounding

- The **total, non-learned, cost-based policy + mandatory EXPLAIN** decision: RFC-0005 §2 (decision), §2.2 (the EXPLAIN record's contents), §2.3 (determinism), §2.4 (override), §2.5 (the exact-metadata advantage that avoids the cardinality trap).
- **Reification / content-addressing / "which policy chose this?"**: RFC-0005 §3; RFC-0001 §4.6 (`Meta.policy_used`, content-addressing); ADR-003 (content-addressed identity); ADR-006 (no black boxes — an unanalyzable policy *is* the black box forbidden).
- **One mechanism, many sites**: RFC-0005 §4 (swap-target + packing-schedule selection share `select`); M-222 (the single `select` adapter behind both).
- **Landed capability crate this consumes**: `mycelium-select` — M-220 (`SelectionPolicy`, closed `Predicate` language, `CostModel` in real storage **bits**, `policy_ref()` content-addressing), M-221 (`Explanation` on every `select`; `explain` total + deterministic; LSP-surfaced, SC-5), M-222 (the wiring + the `WrongSiteKind` refusal). KC-3 §5 (selection stays out of the kernel).
- **The library contract**: RFC-0016 §4.1 (C1–C6), §4.3 (the `select`/`explain` row), §4.5 (the guarantee-matrix obligation). Requirement ids: SC-3 / G2 (no black boxes / never-silent — the operational form this module realizes), VR-5 (honest tags), KC-3 (above the kernel), G11 / SC-5 (the EXPLAIN surfaced via the LSP).

## 7. Open questions (FLAGGED — resolve before ratification)

- **(Q1) The exact `Explanation` record schema.** RFC-0005 §2.2 fixes the record's *contents* (inputs considered, per-candidate cost, chosen option, override state) and M-221 landed a serializable `Explanation`; the precise public **field names / type** the stdlib re-exports are owned by `mycelium-select` and are **not** restated verbatim here (FLAGGED — describe abstractly; reconcile against the landed crate's signature before ratification, never invented). This spec must not fabricate the schema.
- **(Q2) Cost units beyond storage bits.** The landed `CostModel` is in real storage **bits** (M-220) — the right declared unit for the swap/packing sites. Whether future sites need a *different* declared unit (e.g. cycles, energy), and whether a single `CostModel` can carry more than one declared unit without re-introducing "arbitrary internal units" (the RFC-0005 §2 black-box mode), is open. Disposition: out of scope for v0 — surface only the storage-bits cost the landed crate provides.
- **(Q3) Ergonomics vs the always-explicit EXPLAIN (tension A → RFC-0016 §8-Q3).** How much of the EXPLAIN machinery is *implicit-by-default-but-inspectable* vs returned at every call site is the cross-cutting RFC-0016 §8-Q3 ergonomics question. This module is the **sharpest** instance (EXPLAIN is mandatory, never suppressible — C3); whether the default return is `(choice, explanation)` or `choice` with `explanation` fetched on demand is a per-ring design pass, FLAGGED to §8-Q3.
- **(Q4) Composition / conflict precedence as a library surface.** RFC-0005 §4 states multiple applicable policies compose deterministically by a fixed declared precedence. Whether `std.select` exposes policy *composition* as a first-class op (and how the composed `PolicyRef` is content-addressed), or leaves composition to the call site, is open. Disposition: defer to v1 unless a v0 site needs it.

## Meta — changelog

- **2026-06-17 — Draft (needs-design).** Stands up the `std.select` / `explain` module design spec (M-519, #161; Ring 1, Tier A) as the library form of **RFC-0005 / ADR-006** (M-220/221/222): the **total, non-learned, content-addressed** decision-table selection policy (cost in real declared units — storage **bits**, never "arbitrary internal units") and the **mandatory `Explanation` record** — the operational home of C3 / SC-3 (no black boxes, G2), where **no selection exists without an EXPLAIN**. Fixes the scope + boundary (decides + explains; does *not* perform the swap (`std.swap`) or consume the packing schedule (M-250)), the exported-op surface (`build`/`select`/`explain`/`override` + `PolicyRef`), and the load-bearing **guarantee matrix** (5 rows, all `Exact` — a total table over exact metadata, VR-5 honest, no probabilistic/learned claim; every selection op EXPLAIN-able = yes). §4.1 conformance walked clause-by-clause (C3 is the crux: every selection emits a valid, inspectable record citing per-candidate cost in bits; the policy is content-addressed → deterministic + auditable, C4/ADR-003). Four §7 questions FLAGGED — the exact `Explanation` schema (not fabricated; owned by the landed crate), cost units beyond bits, the EXPLAIN-ergonomics tension (→ RFC-0016 §8-Q3), and composition/conflict as a surface. No code; no kernel change (KC-3 — an EXPLAIN consumer above the kernel). Append-only.

- **2026-06-20 — Accepted (maintainer ratification, DN-07).** The maintainer ratified this Rust-first spec: the §4.5 guarantee matrix is asserted in tests, never-silent fallibility and honest per-op tags hold, and the open §7/§8 questions are design/scope calls, not contract violations. No guarantee tag was upgraded without a checked basis (VR-5). Status moves *Implemented (Rust-first) — pending ratification → Accepted*. Append-only; no kernel change (KC-3).
