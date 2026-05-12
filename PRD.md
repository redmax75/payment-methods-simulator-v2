# Cashier Simulator

# Overview
Operator-facing preview tool that renders the **first screen of the cashier** as a synthetic player would see it. Backoffice users set a country, currency, segment, and trust state, and the simulator runs the full evaluator (visibility + orchestration) against every method, then renders the visible methods grouped by family with their Min / Max deposits in the chosen currency. Replaces the need to make real test deposits or escalate via screenshots when auditing "why does this player see / not see this method?".

# Design
[Image Placeholder]
[Figma Link]

> **Note on segments:** Inherits the global rule — every segment surfaced (Lifecycle tiers, VIP levels) is fetched live from the central Segmentation Engine. The simulator never defines segments locally; it consumes the catalog.

> **Disclaimer:** This is a **prototype**, not a production tool. All inputs are operator-typed and the evaluator runs against the seed catalog shared with the per-method admin and the Overview wireframes. Live player data, real PSP counter feeds, and live FX rates are out of scope for v1.

# Requirements

**Problem context**
* Backoffice users should be able to answer "what would *this* player see on the cashier landing screen right now?" without needing a real test account or a deposit.
* Backoffice users should be able to compare *trusted* vs *untrusted* traffic for the same country/segment with a single toggle.
* Backoffice users should be able to share a profile URL so a colleague (Risk, Ops, Compliance) lands on the same simulation with one click.

**Page anatomy**
* Backoffice users should be able to reach the Cashier Simulator from the same top-nav strip as Overview and Method Config (the three sibling Vercel apps).
* Backoffice users should see a **two-pane layout**: profile form on the left, simulated cashier on the right.
* Backoffice users should see a sticky "View rules for {country}" button anchored at the bottom of the cashier pane.

**Always-visible player profile inputs (happy path)**
* Backoffice users should be able to set the player **country** via a searchable dropdown using full country names (not ISO codes).
* Backoffice users should be able to set the player **currency** via a dropdown that populates only **after** a country is chosen; the dropdown lists the currencies licensed for that country in the country catalog.
* The system should default the currency to the country's primary currency, and re-render Min / Max on every method when the currency changes.
* Backoffice users should be able to set the player **Lifecycle segment** and **VIP level** via two dropdowns fetched live from the Segmentation Engine.
* Backoffice users should see a **Trust state preset toggle** with four mutually exclusive options: **Untrusted**, **Trusted Options**, **Affiliate-trusted**, **Both** (see Trust preset section).

**Trust preset toggle**
* The system should expose a Trust state toggle that prefills the underlying Advanced fields so backoffice users do not need to know which inputs drive trust:
    1. **Untrusted** — affiliate = none, settled deposit count = 0, cumulative deposit amount = 0 EUR.
    2. **Trusted Options** — affiliate = none, settled deposit count = above each method's threshold (default = 10), cumulative amount = 1000 EUR.
    3. **Affiliate-trusted** — affiliate = seed trusted affiliate (`AFF-VIP-001`), settled count = 0, amount = 0 EUR.
    4. **Both** — affiliate = seed trusted affiliate, settled count = 10, amount = 1000 EUR.
* Backoffice users should be able to override any prefilled value in the Advanced fold; doing so does not flip the preset label.
* The system should reflect the preset in the URL hash so that sharing the link preserves it.

**Advanced inputs (collapsed by default)**
* Backoffice users should be able to expand an **Advanced** section to override individual fields manually.
* Backoffice users should be able to set: affiliate ID, settled deposit count, cumulative deposit amount (EUR), weekday + time (UTC), direction (deposit / withdrawal), and per-method PSP counters (daily / monthly / custom × count / amount).
* The system should default direction to **Deposit** in v1 (the cashier landing screen this simulator mirrors is the deposit flow).
* The system should default weekday + time to the current UTC moment.
* Backoffice users should see "Advanced overrides active" amber chip on the form header whenever any Advanced field deviates from its preset-supplied value.

**Currency handling and method visibility**
* The system should treat **currency match as a hard gate**: a method is hidden if `player.currency ∉ method.currencies`.
* Backoffice users should see the failing currency match called out in the method's trace when expanded.
* Backoffice users should see a "No methods match {currency}" empty state in the cashier pane when the currency gate hides every method.

**Cashier list rendering (right pane)**
* Backoffice users should see only the methods that pass every visibility layer rendered as tiles/rows, grouped by family in this order: **Card · Wallet · Bank Transfer · Open Banking · Other**.
* Backoffice users should see, per method tile: family icon, method title, **Min / Max deposit in the player's currency**, status indicator, and any active trust tag (see Affiliate-trust indicators).
* Backoffice users should see verdict colour coding on each tile: green (`show`), blue (`routed → X`), amber (`message`); grey (`hidden`, visible only when "Show hidden methods" debug is on).
* Backoffice users should be able to click any tile to expand a trace pane showing every evaluator layer that fired (Hard gate → Trusted Source → Trusted Options → Schedule → Orchestration) and its decision.

**Min / Max display and tier resolution**
* The system should display **Min and Max deposit** under every visible method tile in the player's selected currency.
* The system should resolve the player's **effective tier** at simulation time as the highest tier the player qualifies for: VIP level outranks Lifecycle; if neither is set or matched, the NDC floor applies.
* The system should read Min / Max from the resolved tier on the method's Limits tab; blank tier fields inherit from NDC (the mandatory floor).
* The system should FX-convert Min / Max from EUR to the player's currency via the seed FX table; values stored on the method itself remain EUR-normalised.
* The system should round display values to whole units (no minor cents) for the wireframe; production currency formatting is parked.

**Affiliate-trust indicators on the cashier**
* Backoffice users should see an inline tag on each method tile when the **Affiliate-trusted** or **Both** preset is active:
    * `{affiliate} trust: {segment}+ required` — when the affiliate has a Min Lifecycle segment override for this method.
    * `{affiliate} trust: any segment` — when no segment gate is configured.
* The system should render an amber chip "Trust bypass not applied — segment too low" on the tile when the player's resolved segment is below the affiliate's Min Lifecycle requirement; the trace should also show the fallthrough to Trusted Options (Layer 2).

**Trace expander per method**
* Backoffice users should be able to click any method tile (visible or hidden-with-debug-on) to expand a numbered trace listing each evaluator layer's input and decision.
* The system should always include the final verdict (`show` / `hide` / `routed` / `message`) at the bottom of the trace.
* Backoffice users should be able to collapse the trace via a second click or an X control.

**Show-hidden debug toggle**
* Backoffice users should be able to flip a "Show hidden methods (debug)" toggle in the profile form.
* When on, the system should render hidden methods at the bottom of the cashier pane in a faded style with a chip identifying which layer blocked them.

**Rules-for-country modal**
* Backoffice users should be able to click a sticky bottom button "View rules for {country}" to open a modal summarising the configured rules per method for the selected country and the player's resolved tier.
* The system should render the modal as a centered overlay with a backdrop; **Escape**, **click-outside**, and an X control should close it.
* The system should render one row per method that survives the hard gates (country + currency + direction); rows mirror what is visible in the cashier above.
* The system should show the following columns: **Method**, **Schedule** (compact: "24/7" or "Mon-Fri 06-23 UTC"), **Trust bypass affiliates** (first 1-2 names + "+N more"), **Player's tier limits** (one Min/Max pair for the resolved tier, in player's currency), **Orchestration** (one-liner like "Daily 3 → route → GPay" or "—").

**Shareable profile URL**
* The system should encode the entire profile state (country, currency, segment, VIP, trust preset, all Advanced overrides, show-hidden toggle) into the URL hash on every change.
* Backoffice users should be able to copy the current URL via a "Copy share link" button in the form header; pasting it elsewhere should reproduce the exact simulation on load.
* The system should not store any profile data on a server — the URL is the only artefact.

**Reuse of the evaluator**
* The system should reuse the same `evaluate(method, ctx)` function as the per-method tester widget and the per-method admin.
* The system should produce identical verdicts in the Simulator and in the per-method admin's tester widget for the same `(method, ctx)` input.

---

**Always-visible profile validations**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Country | Searchable Predefined List (full name) | Required, must be Active in country catalog | "Pick a country" | Drives hard gate Layer 0 and unlocks the currency dropdown |
| Currency | Predefined List | Required after country, must be in the country's licensed currencies | "Pick a currency" | Drives the currency hard gate and Min/Max FX conversion |
| Lifecycle segment | Predefined List | Optional; must exist in Segmentation Engine catalog | "Segment no longer available upstream" | Fetched live |
| VIP level | Predefined List | Optional; must exist in Segmentation Engine catalog | "VIP level no longer available upstream" | Fetched live; outranks Lifecycle for tier resolution |
| Trust state preset | Toggle (4 options) | Required, one of Untrusted / Trusted Options / Affiliate-trusted / Both | — | Prefills Advanced fields |
| Show hidden methods | Toggle | Optional | — | Debug-only |

**Advanced overrides validations**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Affiliate ID | Text | Optional; if non-empty must exist in affiliate registry | "Unknown affiliate ID" | Set automatically by Affiliate-trusted preset |
| Settled deposit count | Number | Optional, integer ≥ 0 | "Must be a non-negative integer" | Set automatically by Trusted Options / Both presets |
| Cumulative deposit amount | Number (EUR) | Optional, ≥ 0 | "Must be ≥ 0 EUR" | EUR-normalised |
| Weekday | Predefined List | Optional, Mon–Sun | — | Defaults to current UTC weekday |
| Time | Time (HH:MM, UTC) | Optional | "Invalid time" | Defaults to current UTC time |
| Direction | Predefined List | Required, one of Deposit / Withdrawal | — | Defaults to Deposit in v1 |
| PSP counter (per method) | Number | Optional; each ≥ 0 | "Counter must be ≥ 0" | Daily / monthly / custom × count / amount |

**Cashier tile display contract**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Method title | Derived | Required (from catalog) | — | Read from `title` in the method catalog |
| Family icon | Derived | Required | — | Read from method's `family` field |
| Min deposit (display) | Derived | Required; ≥ 0; resolved tier's Min after FX | "Min unavailable" | EUR → player currency via seed FX |
| Max deposit (display) | Derived | Required; > Min after FX | "Max unavailable" | EUR → player currency via seed FX |
| Verdict pill | Derived | Required, one of show / routed / message / hide-with-debug | — | Hidden methods only shown when debug toggle is on |
| Trust tag | Derived | Optional; shown only when Affiliate-trusted / Both preset is active | — | Pulls from method's Routing Rules Min Lifecycle segment override |
| Trust segment shortfall chip | Derived | Optional; shown when affiliate has Min segment > player's resolved segment | — | Amber chip; trace explains the fallthrough |

**Rules-for-country modal contract**
| **Field** | **Type** | **Validation Rule** | **Error message** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Method (row) | Derived | Required; one row per method passing hard gates | — | Hidden methods excluded |
| Schedule summary | Derived | Required; "24/7" or compact "Mon-Fri HH-HH UTC" | — | UTC always |
| Trust bypass affiliates | Derived | Optional; "{N} affiliates" or "{name1}, {name2} +{N}" | — | From Routing Rules tab |
| Player's tier limits | Derived | Required; "{Min} – {Max} {currency}" | — | Resolved tier only, not the full ladder |
| Orchestration | Derived | Optional; one-line summary or "—" | — | e.g. "Daily 3 → route → GPay" or "Daily €5000 → hide" |

# User Flows

## Main Flows

### See the cashier for a player (happy path)
1. Backoffice user opens the Cashier Simulator
2. Picks country "Germany" from the searchable dropdown
3. Currency dropdown unlocks and defaults to "EUR"; segment dropdown unlocks
4. Picks Lifecycle "Silver", leaves VIP blank
5. Leaves Trust state on **Untrusted**
6. Right pane renders the cashier: visible methods grouped by family with Min / Max in EUR
7. Backoffice user clicks any method tile to expand its trace and confirm why it showed

### Switch currency to audit Min / Max
1. From the previous flow, backoffice user changes Currency to "USD"
2. The system re-renders Min / Max on every method using the seed FX table
3. Any method whose `currencies` array does not include "USD" disappears with a trace entry "currency gate"
4. The "View rules for Germany" button at the bottom still works and reflects the new currency

### Compare trusted vs untrusted traffic
1. Backoffice user keeps the country / currency / segment fixed
2. Flips Trust state from **Untrusted** to **Affiliate-trusted**
3. The Advanced fold's affiliate ID auto-fills with `AFF-VIP-001`
4. Each method tile shows the affiliate's Min Lifecycle requirement as a tag (`AFF-VIP-001 trust: Silver+ required` etc.)
5. Backoffice user lowers Lifecycle to "Bronze" → amber chip "Trust bypass not applied — segment too low" appears on the affected tiles and methods that depended on the bypass disappear
6. Backoffice user flips back to **Trusted Options** to see how earned-trust changes the verdicts

### Open the rules-for-country modal
1. Backoffice user clicks "View rules for Germany" at the bottom of the cashier
2. Modal opens with one row per visible method
3. Backoffice user reads the Trust bypass affiliates column to confirm `AFF-VIP-001` applies to the expected methods
4. Closes the modal via Escape

### Share a profile URL
1. Backoffice user fine-tunes a profile that reproduces a player escalation
2. Clicks "Copy share link"
3. Pastes the URL into a Slack thread / Linear ticket
4. A colleague opens the link and lands on the same simulation with all inputs preserved

## Side Flows

* **No country selected:** the cashier pane shows a "Pick a country to start" placeholder; Trust toggle is disabled
* **Country picked but currency dropdown empty:** "{country} has no licensed currencies in the catalog — escalate to the catalog team"; cashier pane stays placeholder
* **No methods match the chosen currency:** cashier pane shows "No methods match {currency} for {country}"; the rules modal button is disabled
* **Segmentation Engine unavailable:** segment and VIP dropdowns disabled with retry banner; simulator can still run with segment = blank (NDC floor applies, no affiliate min segment can be checked)
* **Trust preset chosen but Advanced fields manually overridden:** "Advanced overrides active" amber chip appears on the form header
* **Affiliate ID typed in Advanced is unknown:** the run still works; trace notes "Unknown affiliate — no Trust bypass applied"
* **Player's resolved segment is below the affiliate's Min segment override:** amber chip on the tile + trace shows fallthrough to Trusted Options layer
* **Method has Orchestration enabled and a cap is breached:** tile renders blue (`routed → X`) or amber (`message`); the trace explains which cap fired and which alternative was picked
* **Method's fallback chain is exhausted:** tile rendered grey (or hidden if debug off); trace shows the cascade
* **Custom range PSP counters set but Orchestration's custom range disabled on that method:** counters are ignored; trace notes "Custom range cap disabled for this method"
* **URL hash is malformed when opening a shared link:** the simulator opens with defaults and shows a toast "Could not parse the shared profile — starting fresh"
* **More than 50 visible methods after filtering:** the cashier pane scrolls within its own container; family section headers remain sticky
* **Show-hidden debug toggle on but no methods are hidden:** the toggle is a no-op; UI still confirms the toggle state in the form header

---

### [REQUIRES PM INPUT]

* **Currency gate severity** — confirm that hiding a method on currency mismatch matches the real cashier; if the real cashier sometimes applies FX silently, the gate becomes a warning instead of a hard hide
* **Country → currency catalog** — confirm which team owns the country catalog and whether "primary currency" is a first-class field or computed (e.g. first entry in the licensed list)
* **FX rate source** — for v1 we use a seed FX table; confirm what the production source should be and how often it refreshes
* **Min / Max rounding rules** — whole units in v1; confirm if production requires decimal places per currency (e.g. JPY no decimals, EUR two decimals, BHD three decimals)
* **Trusted Options preset defaults** — we use count = 10 / amount = 1000 EUR as "above any reasonable threshold"; confirm the exact prefill or make it per-method-aware (each method's threshold + 1)
* **Trust preset for "Both" — semantics** — confirm that both presets are applied additively (affiliate trust *and* earned trust evaluated independently) rather than requiring both to fire
* **Rules-for-country modal vs slide-in panel** — we shipped a modal in v1; revisit after operator feedback to decide if a slide-in panel from the right is preferred (so cashier remains visible while reading rules)
* **Direction handling** — v1 hard-codes deposit on the landing-screen mirror; confirm whether the simulator needs to also render the withdrawal landing screen and when
* **Shareable URL secrecy** — the URL hash carries the full profile in plain text; confirm there is no PII risk (synthetic data only) or whether the link needs short-token redirection
* **Family ordering in the cashier pane** — we used Card · Wallet · Bank Transfer · Open Banking · Other; confirm this matches the real cashier
