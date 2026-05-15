# Cashier Simulator — plain-language PRD

> Plain-language sibling of `PRD.md`. Same rules and decisions, easier wording. If the two disagree, **`PRD.md` wins**.

# Overview
The Cashier Simulator shows what the first screen of the cashier looks like for a synthetic player. The operator picks a country, a currency, and a segment. The simulator runs the full evaluator against every payment method. The visible methods are grouped by family with their Min and Max deposit shown in the chosen currency.

# Design
[Image Placeholder]
[Figma Link]

> **Segments:** Lifecycle tiers and VIP levels come from the central Segmentation Engine. The simulator never defines its own.

> **Disclaimer:** This is a prototype, not a production tool. The evaluator runs against the seed catalog shared with the per-method admin and the Overview.

# Requirements

**Problem context**
* Backoffice users should be able to answer "what would this player see on the cashier right now?" without making a real test deposit.
* Backoffice users should be able to share a profile URL so a colleague opens the exact same simulation with one click.

**Page layout**
* Backoffice users should reach the Cashier Simulator from the same top-nav strip as Overview and Method Config.
* Backoffice users should see one single pane. The cashier fills the page.
* Backoffice users should see a breadcrumb `Country ▸ Currency ▸ Segment` in the cashier header.
* Backoffice users should see a "View rules for {country}" button at the bottom of the cashier.

**Breadcrumb inputs — the "what amounts?" axis**
* Backoffice users should pick the player **country** from a searchable dropdown. Country names are full names, not codes.
* Backoffice users should pick the player **currency** from a dropdown. The dropdown only unlocks after a country is chosen. The list contains only the currencies licensed for that country.
* The system should default the currency to the country's primary currency. The system should redraw Min and Max on every method whenever currency changes.
* Backoffice users should pick the player **segment** from a single dropdown. The dropdown has two groups: Lifecycle and VIP. The dropdown only lists tiers that an eligible method has actually configured for the chosen currency. The dropdown is disabled if no method configures any tier for that currency.

**Trust & visibility fold — collapsed by default**
* Backoffice users should be able to open a "Trust & visibility" fold. The fold contains three controls.
* The first control is a **Trusted Options toggle**. It has two states: Not met and Met. An inline line explains what each state means. Picking Met sets settled deposit count to 10 and cumulative deposit amount to 1000 EUR. These values are above each method's Trusted Options threshold.
* The second control is **PSP daily count** (PSP = payment provider). It is a number input. It feeds the orchestration layer's daily cap check.
* The third control is **Show hidden methods**. It is a debug toggle.
* The system should write the trust state into the URL hash.

**Evaluator defaults — not exposed in the UI**
* The system should run all five evaluator layers on every change: Hard gate, Trusted Source, Trusted Options, Schedule, Orchestration.
* The system should fix the affiliate ID to empty. The system should fix direction to Deposit. The system should fix weekday and time to the current UTC moment.
* The trace should still explain a hide caused by Schedule or by an Affiliate Bypass rule. Operators just cannot fake those conditions in v1.

**Currency hard gate**
* The system should hide a method if the player's currency is not in the method's `currencies` list.
* Backoffice users should see the failing currency match in the method's trace.
* Backoffice users should see a "No methods match {currency}" message if every method is hidden by the gate.

**Cashier list**
* Backoffice users should see methods that pass every visibility layer as tiles.
* The tiles should be grouped by family in this order: Card, Wallet, Bank Transfer, Open Banking, Other.
* Each tile should show: family icon, method title, Min and Max deposit in the player's currency, and a verdict pill.
* The verdict pill should be colour-coded: green for `show`, blue for `routed → X`, amber for `message`, grey for `hidden`. Grey tiles only appear when Show hidden methods is on.
* Backoffice users should click any tile to open its trace.

**Min / Max and tier resolution**
* The system should show Min and Max deposit under every visible tile in the player's currency.
* The system should resolve the player's effective tier as the highest tier they qualify for. VIP outranks Lifecycle. If neither is set, the NDC floor applies (NDC = New Depositing Customer — the mandatory baseline tier).
* The system should read Min and Max from the resolved tier on the method's Limits tab. Empty tier fields inherit from NDC.
* The system should convert Min and Max from EUR to the player's currency using the seed FX table (FX = currency conversion).
* The system should round display values to whole units.

**Trace expander**
* Backoffice users should click any tile to open a numbered trace. This works on visible tiles, and on hidden tiles when debug is on.
* The trace should list each evaluator layer with its input and its decision.
* The trace should end with the final verdict: `show`, `hide`, `routed`, or `message`.

**Show-hidden debug toggle**
* When the toggle is on, the system should render hidden methods at the bottom of the cashier in a faded style. Each hidden tile should show a chip naming the layer that blocked it.

**Rules-for-country modal**
* Backoffice users should click "View rules for {country}" to open a modal.
* The modal should summarise the configured rules for every method for the selected country and the player's resolved tier.
* The modal should be a centered overlay with a backdrop. Escape, click-outside, and an X control should close it.
* The modal should have one row per method that passes the hard gates.
* The modal columns should be: Method, Schedule, Player's tier limits, Orchestration.

**Shareable profile URL**
* The system should encode the profile into the URL hash on every change. The encoded state is: country, currency, segment, trust toggle, PSP daily count, show-hidden.
* Backoffice users should click "Copy share link" to copy the current URL.
* Opening the link elsewhere should reproduce the same simulation.

**One evaluator everywhere**
* The system should reuse the same `evaluate(method, ctx)` function as the per-method tester widget.
* The simulator and the per-method admin's tester should always agree on the verdict for the same `(method, ctx)` input.

---

**Field validations**
| **Field** | **Type** | **Rule** | **Error** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Country | Searchable dropdown | Required. Must be Active in the country catalog. | "Pick a country" | Unlocks the currency dropdown |
| Currency | Dropdown | Required after country. Must be in the country's licensed currencies. | "Pick a currency" | Drives FX conversion |
| Segment | Dropdown with Lifecycle and VIP groups | Optional. Must exist in the Segmentation Engine catalog. Only tiers an eligible method configures for the currency appear. | — | Disabled if no eligible tiers |
| Trusted Options | Toggle | Required. One of Not met or Met. | — | Met prefills settled count = 10, amount = 1000 EUR |
| PSP daily count | Number | Optional. Whole number, zero or more. | "Must be ≥ 0" | Feeds orchestration |
| Show hidden methods | Toggle | Optional. | — | Debug-only |

**Cashier tile display**
| **Field** | **Source** | **Rule** | **Error** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Method title | Catalog | Required. | — | From the method's title |
| Family icon | Catalog | Required. | — | From the method's family |
| Min deposit | Resolved tier + FX | Required. Zero or more. | "Min unavailable" | EUR → player currency |
| Max deposit | Resolved tier + FX | Required. Greater than Min after FX. | "Max unavailable" | EUR → player currency |
| Verdict pill | Evaluator | Required. One of show, routed, message, hide-with-debug. | — | Hidden tiles only show with debug on |

**Rules-for-country modal**
| **Field** | **Source** | **Rule** | **Error** | **Notes** |
| :--- | :--- | :--- | :--- | :--- |
| Method (row) | Catalog | Required. One row per method that passes the hard gates. | — | Hidden methods excluded |
| Schedule | Method config | Required. "24/7" or compact "Mon-Fri HH-HH UTC". | — | Always UTC |
| Player's tier limits | Resolved tier + FX | Required. "{Min} – {Max} {currency}". | — | Resolved tier only, not the full ladder |
| Orchestration | Method config | Optional. One-line summary or "—". | — | e.g. "Daily 3 → route → GPay" |

# User Flows

## Main flows

### See the cashier for a player (happy path)
1. The backoffice user opens the Cashier Simulator.
2. The user picks country "Germany".
3. The currency dropdown unlocks and defaults to "EUR". The segment dropdown unlocks.
4. The user picks Lifecycle "Silver".
5. The user leaves Trusted Options on "Not met".
6. The cashier renders: visible methods grouped by family with Min and Max in EUR.
7. The user clicks any tile to open its trace.

### Switch currency to check Min / Max
1. The user changes currency to "USD".
2. The system redraws Min and Max on every method using the seed FX table.
3. Methods whose `currencies` list does not include "USD" disappear. The trace says "currency gate".
4. The "View rules for Germany" button reflects the new currency.

### Flip Trusted Options to compare verdicts
1. The user keeps country, currency, and segment unchanged.
2. The user flips Trusted Options from "Not met" to "Met".
3. Methods that were hidden by the Trusted Options layer now show. The trace explains which counter passed.
4. The user flips back to "Not met" to see the difference.

### Open the rules-for-country modal
1. The user clicks "View rules for Germany".
2. The modal opens with one row per visible method.
3. The user closes the modal with Escape.

### Share a profile URL
1. The user fine-tunes a profile that reproduces a player escalation.
2. The user clicks "Copy share link".
3. The user pastes the URL into Slack or Linear.
4. A colleague opens the link and lands on the same simulation.

## Side flows

* **No country picked:** the cashier shows a "Pick a country to start" placeholder.
* **Country picked but the currency list is empty:** the cashier shows "{country} has no licensed currencies — escalate to the catalog team".
* **No methods match the chosen currency:** the cashier shows "No methods match {currency} for {country}". The rules modal button is disabled.
* **Segmentation Engine is unavailable:** the segment dropdown is disabled with a retry banner. The simulator can still run with segment = blank. The NDC floor applies.
* **A method has Orchestration on and a cap is breached:** the tile renders blue (`routed → X`) or amber (`message`). The trace explains which cap fired.
* **A method's fallback chain is exhausted:** the tile renders grey (or is hidden when debug is off). The trace shows the cascade.
* **The URL hash is malformed when opening a shared link:** the simulator opens with defaults. A toast says "Could not parse the shared profile — starting fresh".
* **More than 50 visible methods:** the cashier scrolls inside its own container. Family section headers stay sticky.

---

### [REQUIRES PM INPUT]

* **Currency gate severity:** confirm that hiding a method on currency mismatch matches the real cashier. If the real cashier applies FX silently in some cases, the gate becomes a warning instead of a hide.
* **Country → currency catalog:** confirm which team owns the country catalog. Confirm whether "primary currency" is a first-class field or computed (e.g. first entry in the licensed list).
* **FX rate source:** v1 uses a seed table. Confirm the production source and how often it refreshes.
* **Min / Max rounding:** v1 uses whole units. Confirm production rules per currency (JPY has no decimals, EUR has two, BHD has three).
* **Trusted Options preset defaults:** v1 uses count = 10 and amount = 1000 EUR. Confirm or switch to per-method-aware values (each method's threshold + 1).
* **Affiliate and Schedule inputs (dropped in this version):** confirm operators do not need to fake these in v1. The trace still explains hides caused by them.
* **Direction:** v1 hard-codes Deposit. Confirm whether the simulator also needs the Withdrawal landing screen.
* **Shareable URL secrecy:** the URL hash carries the full profile in plain text. Confirm there is no PII risk, or whether the link needs short-token redirection.
* **Family ordering:** v1 uses Card · Wallet · Bank Transfer · Open Banking · Other. Confirm this matches the real cashier.
