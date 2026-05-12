# Payment Methods Admin v2 — Cashier Simulator wireframe

Operator-facing preview tool that renders the **first screen of the cashier** as a synthetic player would see it. Set a country, currency, segment, and trust state — the simulator runs the full evaluator (visibility + orchestration) and renders the visible methods grouped by family with Min / Max deposits in the chosen currency.

## Sibling apps

- **Overview**: <https://payment-methods-overview-v2.vercel.app/> — provider-grouped list of all methods
- **Method Config**: <https://payment-methods-admin-v2.vercel.app/> — per-method config screen (5 tabs)

## Stack

Single static `index.html`, vanilla JS, no build step. Deployed via Vercel auto-deploy on push to `main`.

## PRD

See [PRD.md](./PRD.md) for the locked spec (requirements, validation tables, user flows, side flows).
