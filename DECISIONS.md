# Cashier Simulator — Decisions log

Recap of every locked decision made while writing the PRD and building the v1 wireframe. Use this as the "why" companion to `PRD.md` (the "what"). When something looks puzzling in the spec, the rationale is here.

**Date:** 2026-05-12
**Wireframe URL:** https://payment-methods-simulator-v2.vercel.app
**Repo:** https://github.com/redmax75/payment-methods-simulator-v2

---

## 1. Problem framing

| Decision | Rationale |
| :--- | :--- |
| The Simulator solves three concrete pains: no whole-cashier preview, cross-method side effects are invisible, no shareable evidence for Risk / Compliance escalations. | Operators currently choose between the per-method visibility tester (one method at a time) or a real test deposit (expensive, slow). Neither answers "what does *this* player see across the whole cashier?" |
| v1 is a **prototype, not a production tool**. All inputs operator-typed; evaluator runs against the same seed catalog as the other two wireframes. | Builds value fast without depending on live PSP counters, live FX, or live segment fetches. Production data integration is parked. |

---

## 2. Scope of what the Simulator renders

| Decision | Rationale |
| :--- | :--- |
| The right pane mirrors the **cashier landing screen** — a list of methods as the player sees on first arrival. | Operators were asking for "what would the player see?" — answer is the landing list, not the per-method forms. |
| **No payment forms, no detail screens, no redirect flows.** Just the list. | Keeps the wireframe focused; payment-form fidelity is a different (and large) project. |
| Methods grouped by family in this order: **Card · Wallet · Bank Transfer · Open Banking · Other**. | Mirrors typical cashier visual structure. Order is a `REQUIRES PM INPUT` item — confirm with cashier team. |
| **Min and Max deposit shown under every method tile** in the player's selected currency, with the resolved tier label. | Operators reminded us the real cashier shows Min/Max; the simulator needs that fidelity to be useful. |

---

## 3. Player profile inputs — what's visible vs. in Advanced

The "input visibility" question stayed unanswered explicitly, so the PRD locked the inferred default:

| Bucket | Fields | Rationale |
| :--- | :--- | :--- |
| **Always visible** | Country · Currency (after country) · Lifecycle segment · VIP level · Trust state preset · Show-hidden debug toggle | Country + Segment is the operator's stated minimum. Currency follows from country. Trust preset is the audit shortcut. |
| **Advanced fold (collapsed by default)** | Affiliate ID · Settled deposit count · Cumulative deposit amount · Weekday · Time · Direction · PSP daily count | Too much to show by default. Operators expand only when they need to override the Trust preset or tweak edges. |
| **"Overrides active" amber chip** appears on the form header when any Advanced field drifts from its preset-supplied value. | Operator never loses sight that a hidden field is making the result different from the headline preset. |

---

## 4. Currency handling — three locked answers

| Question | Decision | Alternatives rejected |
| :--- | :--- | :--- |
| Should the simulator hide a method when `player.currency ∉ method.currencies`? | **Yes — treat as a hard gate.** | Show with FX silently applied; show with an "EUR-only — FX applies" badge. |
| Where do currency options come from after a country is picked? | **Country catalog defines the licensed currencies** for each country. | Union of currencies across methods licensed in that country; single fixed currency per country. |
| Display Min/Max in the player's currency — how? | **Always show under method name** with `Min X · Max Y · (tier) tier`. | On hover / row expand; defer to v2. |

FX-conversion runs on the display only — stored values stay EUR-normalised.

---

## 5. Trust model — what makes a player "trusted"

| Decision | Rationale |
| :--- | :--- |
| Segment **alone** does not make a player trusted. Segment only matters as (a) a *modifier* on affiliate-bypass rules, or (b) a *tier selector* for Limits. | Came out of clarifying the architecture with the user — they correctly noted that "segments only come to the table when you choose affiliate bypass rules". |
| Trust comes from **two independent layers**: Affiliate-trusted (Layer 1) and Trusted Options (Layer 2, earned trust). | Matches the architecture locked in the per-method PRD. |
| **Renamed "Earned-trust" → "Trusted Options"** in the toggle. | The Layer 2 name had already been renamed in the rest of the system; the simulator picks up the same terminology. |

---

## 6. Trust preset toggle — UI shortcut for the audit

A top-of-form toggle prefills the Advanced fields so operators don't have to know which inputs drive trust:

| Preset | Affiliate | Settled count | Settled amount |
| :--- | :--- | :--- | :--- |
| Untrusted | none | 0 | 0 EUR |
| Trusted Options | none | 10 (above any reasonable threshold) | 1000 EUR |
| Affiliate-trusted | `AFF-VIP-001` (seed trusted affiliate) | 0 | 0 |
| Both | `AFF-VIP-001` | 10 | 1000 EUR |

| Decision | Rationale |
| :--- | :--- |
| 4 mutually exclusive states. | Covers the natural audit matrix (no trust / earned trust / affiliate trust / both). |
| Operator can override any prefilled field; doing so does **not** flip the preset label. | Keeps the preset label honest as a "what I chose" indicator; the chip in the form header flags drift. |
| Trust preset is encoded in the URL hash. | A shared link preserves "I was looking at the Affiliate-trusted view". |
| "Both" preset semantics — additive (each layer evaluated independently) vs requires-both-fire — flagged as `REQUIRES PM INPUT`. | Behaviour matches the evaluator (additive) but the wording could mean either; needs PM confirmation. |

---

## 7. Affiliate trust — what surfaces on the method tiles

When **Affiliate-trusted** or **Both** preset is active, each method tile shows an inline tag:

| Tag | When |
| :--- | :--- |
| `{affiliate} trust: {segment}+ required` | The affiliate has a Min Lifecycle segment override for this method |
| `{affiliate} trust: any segment` | The affiliate has bypass but no segment gate |
| `⚠ Trust bypass not applied — {segment}+ required` (amber) | Player's resolved segment is below the affiliate's Min — trust does not apply, and the trace shows fallthrough to Layer 2 |

| Decision | Rationale |
| :--- | :--- |
| Show the min segment requirement **per method per affiliate**, inline on the tile. | User explicitly requested this — without it, the operator can't tell which methods the bypass *actually* unlocks for the current player. |

---

## 8. Country rules summary — one chosen surface

| Decision | Rationale |
| :--- | :--- |
| Render as a **centered modal** with backdrop. | User accepted either modal or slide-in; picked modal for v1 as the simpler build and more "intentional audit" UX. |
| Closes on Escape / click-outside / X. | Standard modal affordances. |
| One row per method that passes hard gates (country + currency + direction) — methods hidden by hard gates are excluded so the modal mirrors the cashier above. | Avoids cluttering with methods the operator can already see are gone. |
| Columns: **Method · Schedule · Trust bypass affiliates · Player's tier limits · Orchestration**. | Compresses the 5 admin tabs into a one-line-per-method view. |
| Player tier limits show only the *resolved* tier (not the full ladder). | Resolved tier is the only one that materially affects this player. |
| Schedule rendered compact (`24/7` or `Mon-Fri 06-23 UTC`). | Verbosity is wasted at audit time. |
| Trust bypass affiliates listed as `{name} · {minSegment}+` chips, first 1-2 + "+N more". | Most methods have ≤ 2 trusted affiliates; truncation is acceptable. |
| Marked for revisit (`REQUIRES PM INPUT`): may swap to **slide-in panel from the right** if operators find modal too blocking. | User explicitly said "we see the working prototype if we need more we will adjust". |

---

## 9. Min/Max tier resolution

| Decision | Rationale |
| :--- | :--- |
| Effective tier = highest tier the player qualifies for: **VIP outranks Lifecycle**. | Matches typical industry practice (VIP is the "stronger" axis). |
| If neither VIP nor Lifecycle matched, fall back to the **NDC mandatory floor**. | NDC is already the floor in the per-method admin's Limits tab. |
| Blank tier fields inherit from NDC. | Same inheritance pattern the per-method admin uses. |
| FX from EUR → player currency for **display only**; values on the method itself remain EUR-normalised. | Source of truth stays single-currency; rendering layer handles per-player views. |
| Rounding: **whole units in v1** (no minor cents). Per-currency precision (JPY 0, EUR 2, BHD 3) flagged as `REQUIRES PM INPUT`. | Wireframe-level shortcut. |

---

## 10. Layered evaluator — what runs when

Re-confirmed from the per-method admin architecture (no changes for the simulator):

```
Layer 0 — Hard gates  (status / country / direction / currency)
Layer 1 — Trusted Source  (affiliate bypass with optional min segment)
Layer 2 — Trusted Options (earned trust: count OR amount threshold)
Layer 3 — Schedule        (per-weekday UTC windows)
Layer 4 — Orchestration   (PSP caps → alternative / hide / message)
```

| Decision | Rationale |
| :--- | :--- |
| Currency is treated as a **Layer 0 hard gate**. | Newly added when the user asked for the currency switch. |
| Bypass logic: Layer 1 can skip Layer 3 (Bypass Schedule) and Layer 2 (Bypass Trusted Options) independently. | Matches the per-method admin's split-toggle model. |
| Layer 4 runs only when Layers 0-3 produce `show`. | If a method is already hidden, Orchestration doesn't apply. |
| Orchestration alternatives are checked **single-hop** (no recursive chain following) via a visibility-only sub-evaluator. | Matches the locked decision from the Orchestration v1 spec. |
| Verdicts: **show · hide · routed · message** (4-valued). | Matches the per-method admin tester widget. |
| Same `evaluate(method, ctx)` reused across simulator and per-method tester. | Single source of truth for "what would happen". |

---

## 11. Shareable profile URL

| Decision | Rationale |
| :--- | :--- |
| All profile state encoded in the URL hash on every change. | No backend required — the URL *is* the artefact. |
| "Copy share link" button in the form header. | Lets ops paste into Slack / Linear tickets for escalation. |
| `REQUIRES PM INPUT`: confirm there's no PII risk in the URL. | Synthetic data only in v1 — but the design should consider short-token redirection for production. |

---

## 12. Empty / debug states

| Decision | Rationale |
| :--- | :--- |
| No country → cashier shows "Pick a country to start"; rules button disabled. | Country is required for hard-gate evaluation. |
| Country with no licensed currencies → "no licensed currencies — escalate to catalog team". | Surfaces a backend catalog gap rather than failing silently. |
| All methods hidden by currency mismatch → "No methods match {currency}". | Avoids an empty list with no explanation. |
| "Show hidden methods (debug)" toggle adds a faded section at the bottom of the cashier showing methods that were filtered out + which layer hid them. | Audit aid. |

---

## 13. Scope deferrals — what we deliberately did NOT include in v1

| Deferred | Reason |
| :--- | :--- |
| Real cashier UX (payment forms, redirects, prices) | Different product; this is a simulator, not the cashier. |
| Regression test runner / scripted suites | Existing `visibility-test-cases.md` covers this for the per-method admin. |
| Multi-player or cohort view | Single profile at a time; cohort view parked as v2. |
| Live PSP counter feed | All counters operator-typed in v1. |
| Live FX rates | Seed FX table in v1; production source is `REQUIRES PM INPUT`. |
| Live segment fetches | Lifecycle / VIP lists hardcoded in v1; will fetch from Segmentation Engine in production. |
| Withdrawal landing screen | v1 mirrors deposit only; withdrawal `REQUIRES PM INPUT`. |
| RBAC and audit log | Deferred at the Overview level; same applies here. |

---

## 14. Architecture & deployment

| Decision | Rationale |
| :--- | :--- |
| **Separate Vercel app** (`payment-methods-simulator-v2.vercel.app`) parallel to the Overview and per-method admin. | User wanted wireframes and URLs separated so each piece is its own demoable URL. |
| **Separate GitHub repo** (`redmax75/payment-methods-simulator-v2`). | Same separation rationale. |
| Single-file HTML + vanilla JS, no build step. | Matches the pattern of the other two wireframes; deploys instantly via Vercel auto-deploy on push to `main`. |
| Top-nav strip in each wireframe cross-links to the other two via absolute URLs. | The three are sibling apps, not a monorepo. |
| **PRD lives in the same repo as its wireframe** (each PRD with its own wireframe project). | Locked when the overview wireframe was split out; same topology here. |

---

## 15. Parked `REQUIRES PM INPUT` items (full list as of v1 ship)

These are documented in `PRD.md` and re-listed here for ease of triage:

1. **Currency gate severity** — confirm hide-on-mismatch matches the real cashier; or is FX applied silently?
2. **Country → currency catalog** — confirm catalog ownership + whether "primary currency" is a first-class field.
3. **FX rate source** — production source + refresh cadence.
4. **Min/Max rounding** — per-currency precision (JPY/EUR/BHD).
5. **Trusted Options preset defaults** — fixed (10 / 1000) vs per-method-aware (threshold + 1).
6. **"Both" preset semantics** — additive vs both-must-fire.
7. **Rules modal vs slide-in panel** — revisit after operator feedback.
8. **Direction handling** — withdrawal landing screen in v2?
9. **Shareable URL secrecy** — PII risk; need for short-token redirection?
10. **Family ordering** — confirm `Card · Wallet · Bank Transfer · Open Banking · Other` matches the real cashier.

---

## How to use this document

- When a new contributor asks "why did we hide methods on currency mismatch?" — point them at §4.
- When a follow-up PRD revisits the modal vs slide-in panel — §8 has the original framing.
- When someone proposes adding a field to the always-visible form — check §3 and the "input visibility" rationale before reopening.
- When the PRD picks up `REQUIRES PM INPUT` answers — convert them from §15 into locked rows in the relevant section.
