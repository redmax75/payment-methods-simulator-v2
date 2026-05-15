# Cashier Simulator

# Overview
Operator-facing preview tool that renders the **first screen of the cashier** as a synthetic player would see it. Operator picks country, currency, and segment; the simulator runs the full evaluator against every method and renders the visible methods grouped by family with Min / Max deposits in the chosen currency.

# Design
[Image Placeholder]
[Figma Link]

> **Segments:** Lifecycle tiers and VIP levels are fetched from the central Segmentation Engine. The simulator never defines segments locally.

> **Disclaimer:** Prototype, not production. Evaluator runs against the seed catalog shared with the per-method admin and the Overview.

# Requirements

**Problem context**
* Backoffice users should be able to answer "what would this player see on the cashier landing screen right now?" without a real test account.
* Backoffice users should be able to share a profile URL so a colleague lands on the same simulation with one click.

**Page anatomy**
* Backoffice users should reach the Cashier Simulator from the same top-nav strip as Overview and Method Config.
* Backoffice users should see a **single-pane layout**: the simulated cashier fills the page; a `Country ▸ Currency ▸ Segment` breadcrumb sits in the cashier header.
* Backoffice users should see a "View rules for {country}" button at the bottom of the cashier pane.

**Breadcrumb drill-down (amounts axis)**
* Backoffice users should set the player **country** via a searchable dropdown using full country names.
* Backoffice users should set the player **currency** via a dropdown populated only after a country is chosen; the list is the currencies licensed for that country.
* The system should default currency to the country's primary currency and re-render Min / Max on every method when currency changes.
* Backoffice users should set the player **segment** via a single dropdown with two optgroups: **Lifecycle** and **VIP**. The list contains only tiers an eligible method configures for the chosen currency; disabled if none.

**Trust & visibility fold (collapsed by default)**
* Backoffice users should expand a "Trust & visibility" fold containing three controls:
    1. **Trusted Options toggle** — segmented `[ Not met | Met ]` with an inline consequence line. `Met` sets settled deposit count = 10 and cumulative amount = 1000 EUR (above each method's threshold).
    2. **PSP daily count** — number input feeding orchestration caps.
    3. **Show hidden methods** — debug toggle.
* The system should reflect the trust state in the URL hash.

**Evaluator defaults (not exposed in UI)**
* The system should run all 5 evaluator layers (Hard gate → Trusted Source → Trusted Options → Schedule → Orchestration) on every change.
* The system should fix affiliate = empty, direction = deposit, weekday + time = current UTC. The trace still explains schedule and affiliate hides; operators cannot fake those conditions in v1.

**Currency hard gate**
* The system should treat currency match as a hard gate: a method is hidden if `player.currency ∉ method.currencies`.
* Backoffice users should see the failing currency match called out in the method's trace.
* Backoffice users should see "No methods match {currency}" when the gate hides every method.

**Cashier list rendering**
* Backoffice users should see methods that pass every visibility layer rendered as tiles grouped by family in this order: **Card · Wallet · Bank Transfer · Open Banking · Other**.
* Backoffice users should see, per tile: family icon, method title, **Min / Max deposit in the player's currency**, verdict pill.
* Backoffice users should see verdict colours: green (`show`), blue (`routed → X`), amber (`message`), grey (`hidden`, only when Show hidden is on).
* Backoffice users should click any tile to expand a trace pane showing every evaluator layer and its decision.

**Min / Max and tier resolution**
* The system should display Min and Max deposit under every visible tile in the player's currency.
* The system should resolve the player's effective tier as the highest tier they qualify for: VIP outranks Lifecycle; if neither, the NDC floor applies.
* The system should read Min / Max from the resolved tier on the method's Limits tab; blank tier fields inherit from NDC.
* The system should FX-convert Min / Max from EUR via the seed FX table; rounded to whole units.

**Trace expander**
* Backoffice users should click any tile (visible or hidden-with-debug-on) to expand a numbered trace listing each evaluator layer's input and decision and the final verdict.

**Show-hidden debug toggle**
* When on, the system should render hidden methods at the bottom of the cashier in a faded style with a chip identifying which layer blocked them.

**Rules-for-country modal**
* Backoffice users should click "View rules for {country}" to open a modal summarising configured rules per method for the selected country and the player's resolved tier.
* The system should render the modal as a centered overlay; **Escape**, click-outside, and X close it.
* The system should render one row per method passing the hard gates with columns: **Method**, **Schedule**, **Player's tier limits**, **Orchestration**.

**Shareable profile URL**
* The system should encode the profile state (country, currency, segment, trust toggle, PSP daily count, show-hidden) into the URL hash on every change.
* Backoffice users should copy the current URL via a "Copy share link" button; pasting it elsewhere reproduces the simulation.

**Reuse of the evaluator**
* The system should reuse the same `evaluate(method, ctx)` function as the per-method tester widget.
* The system should produce identical verdicts in the Simulator and in the per-method admin's tester widget for the same `(method, ctx)` input.

---

**Field validations**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Country | Searchable Predefined List | Required, must be Active | "Pick a country" | Unlocks currency dropdown |
| Currency | Predefined List | Required after country, must be in the country's licensed currencies | "Pick a currency" | Drives FX conversion |
| Segment | Predefined List (Lifecycle + VIP optgroups) | Optional; must exist in Segmentation Engine catalog; only tiers an eligible method configures appear | — | Disabled if no eligible tiers |
| Trusted Options | Toggle | Required, one of Not met / Met | — | Prefills settled count + amount |
| PSP daily count | Number | Optional, integer ≥ 0 | "Must be ≥ 0" | Feeds orchestration |
| Show hidden methods | Toggle | Optional | — | Debug-only |

**Cashier tile display contract**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Method title | Derived | Required | — | From catalog |
| Family icon | Derived | Required | — | From method's `family` |
| Min deposit | Derived | Required; ≥ 0; resolved tier's Min after FX | "Min unavailable" | EUR → player currency |
| Max deposit | Derived | Required; > Min after FX | "Max unavailable" | EUR → player currency |
| Verdict pill | Derived | Required, one of show / routed / message / hide-with-debug | — | Hidden only with debug on |

**Rules-for-country modal contract**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Method (row) | Derived | Required; one row per method passing hard gates | — | Hidden methods excluded |
| Schedule summary | Derived | Required; "24/7" or "Mon-Fri HH-HH UTC" | — | UTC |
| Player's tier limits | Derived | Required; "{Min} – {Max} {currency}" | — | Resolved tier only |
| Orchestration | Derived | Optional; one-line summary or "—" | — | e.g. "Daily 3 → route → GPay" |

# User Flows

## Main Flows

### See the cashier for a player (happy path)
1. Backoffice user opens the Cashier Simulator
2. Picks country "Germany"
3. Currency dropdown unlocks and defaults to "EUR"
4. Segment dropdown unlocks; picks Lifecycle "Silver"
5. Leaves Trusted Options on **Not met**
6. The cashier renders: visible methods grouped by family with Min / Max in EUR
7. Backoffice user clicks any tile to expand its trace

### Switch currency to audit Min / Max
1. Backoffice user changes Currency to "USD"
2. The system re-renders Min / Max via seed FX
3. Methods whose `currencies` array does not include "USD" disappear with a trace entry "currency gate"
4. The "View rules for Germany" button reflects the new currency

### Flip Trusted Options to compare verdicts
1. Backoffice user keeps country / currency / segment fixed
2. Flips Trusted Options to **Met**
3. Methods gated by Trusted Options now show; trace explains which counter passed
4. Flips back to **Not met** to confirm the difference

### Open the rules-for-country modal
1. Backoffice user clicks "View rules for Germany"
2. Modal opens with one row per visible method
3. Closes via Escape

### Share a profile URL
1. Backoffice user fine-tunes a profile reproducing a player escalation
2. Clicks "Copy share link"
3. Pastes the URL into Slack / Linear
4. A colleague opens the link and lands on the same simulation

## Side Flows

* **No country selected:** cashier pane shows a "Pick a country to start" placeholder.
* **Country picked but currency dropdown empty:** "{country} has no licensed currencies — escalate to the catalog team".
* **No methods match the chosen currency:** "No methods match {currency} for {country}"; rules modal button disabled.
* **Segmentation Engine unavailable:** segment dropdown disabled with retry banner; simulator runs with segment = blank (NDC floor applies).
* **Method has Orchestration enabled and a cap is breached:** tile renders blue (`routed → X`) or amber (`message`); trace explains which cap fired.
* **Method's fallback chain is exhausted:** tile rendered grey (or hidden if debug off); trace shows the cascade.
* **URL hash is malformed when opening a shared link:** the simulator opens with defaults and shows a toast "Could not parse the shared profile — starting fresh".
* **More than 50 visible methods after filtering:** the cashier scrolls within its own container; family section headers remain sticky.

---

### [REQUIRES PM INPUT]

* **Currency gate severity** — confirm hiding on currency mismatch matches the real cashier; if FX is applied silently in production, the gate becomes a warning.
* **Country → currency catalog** — confirm catalog ownership and whether "primary currency" is first-class or computed.
* **FX rate source** — v1 uses a seed table; confirm production source and refresh cadence.
* **Min / Max rounding** — whole units in v1; confirm per-currency decimal rules (JPY 0, EUR 2, BHD 3).
* **Trusted Options preset defaults** — count = 10 / amount = 1000 EUR; confirm or make per-method-aware.
* **Affiliate + schedule inputs (dropped in this version)** — confirm operators don't need to fake these in v1; trace still explains hides driven by them.
* **Direction handling** — v1 hard-codes deposit; confirm if the withdrawal landing screen is needed.
* **Shareable URL secrecy** — URL hash carries the full profile in plain text; confirm no PII risk.
* **Family ordering** — Card · Wallet · Bank Transfer · Open Banking · Other; confirm it matches the real cashier.
