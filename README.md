# Lucky Cat — Multi-Tenant Restaurant SaaS, Built by One Developer with AI Coding Agents (Case Study)

[![Supabase](https://img.shields.io/badge/backend-Supabase-3ecf8e?logo=supabase&logoColor=white)](https://supabase.com)
[![Postgres RLS](https://img.shields.io/badge/security-Postgres%20RLS-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
[![Cloudflare Pages](https://img.shields.io/badge/hosting-Cloudflare%20Pages-f38020?logo=cloudflare&logoColor=white)](https://pages.cloudflare.com)
[![Stripe Connect](https://img.shields.io/badge/payments-Stripe%20Connect-635bff?logo=stripe&logoColor=white)](https://stripe.com/connect)
[![TypeScript](https://img.shields.io/badge/frontend-TypeScript%20%2F%20React-3178c6?logo=typescript&logoColor=white)](https://www.typescriptlang.org)
![type: case study](https://img.shields.io/badge/type-case%20study-informational)
![paying customers](https://img.shields.io/badge/paying%20customers-0-lightgrey)

**Row-Level Security as the tenant boundary · Stripe Connect for split payments · Postgres + Edge Functions, no server to babysit.**

> Source is private, because it is a commercial product. This is the story of how it
> is built: the problem, the architecture, and the decisions. Screenshots and a system
> diagram, no product code.

**Lucky Cat** is a multi-tenant SaaS I designed and built as one developer, directing AI coding agents (Claude Code, Codex), as two connected apps
on one backend:

- **Ownly** — the owner/manager app: shifts and payroll, stock with auto-deduction, a cash-up flow, and a live orders board.
- **Tillr** — the customer ordering PWA: menu, cart, checkout, and real-time order tracking.

Development started in June 2026 and is ongoing.

> **Where it actually stands.** No paying customer has ever used it: zero orders, zero
> revenue, no live client. Stripe **is** in live mode and
> the card rail has been exercised end to end with a real €1 payment of my own, but
> that is infrastructure proven, not a customer. Everything below — the architecture,
> the security review, the migration drift — is real work on a real system; what it is
> not is a deployment somebody pays for.

## 🍽️ The problem

Small restaurants get squeezed between clunky, expensive POS suites and a pile of
disconnected tools (a separate ordering site, a spreadsheet for stock, another for
rota, a card terminal that doesn't talk to any of them). I wanted one system where a
customer's order deducts stock, feeds the owner's live board, and settles a real
split payment — without the owner touching four dashboards.

## 🏗️ Architecture

```mermaid
flowchart TD
    subgraph Clients
        T[Tillr PWA<br/>customer ordering]
        O[Ownly<br/>owner dashboard]
    end

    subgraph Cloudflare
        CF[Pages + Workers<br/>static apps + edge routing]
    end

    subgraph Supabase
        PG[(Postgres + RLS)]
        EF[Edge Functions<br/>orders · payments · webhooks]
        RT[Realtime<br/>live order board]
    end

    S[Stripe Connect<br/>split payments]

    T --> CF --> EF
    O --> CF --> EF
    EF --> PG
    PG --> RT --> O
    EF --> S
    S -->|webhook| EF
```

**One database, many tenants.** The design uses `store_id` and Postgres
**Row-Level Security** to restrict tenant access in the database. Isolation also depends
on the roles, grants, and API paths involved; the audit below found a function that
allowed cross-tenant stock changes. The same schema serves a demo store and any
future tenant, selected by an environment variable per deployment.

**Payments on a live rail.** Stripe **Connect** is wired to take the customer's card and
split it — a platform fee to Lucky Cat, the rest to the restaurant — reconciled by signed
webhooks. Stripe is in live mode and the card rail has been exercised end to end with a
real €1 payment of my own. What has not happened is a split settling into a connected
account, because no restaurant has operated on it: the rail is proven, the payout half
is built and untested.

**Edge-first.** The apps are static builds on **Cloudflare Pages**, with backend logic in
**Supabase Edge Functions** and Postgres.

### Tenant isolation, illustrated

```mermaid
flowchart LR
    subgraph DB["Postgres — one physical table"]
        RowA[("orders row<br/>store_id = A")]
        RowB[("orders row<br/>store_id = B")]
    end

    ReqA["Request, authenticated<br/>as Store A"] -->|"RLS: store_id = A"| RowA
    ReqB["Request, authenticated<br/>as Store B"] -->|"RLS: store_id = B"| RowB
    ReqA -.->|blocked by RLS| RowB
    ReqB -.->|blocked by RLS| RowA
```

*Same table, same schema, every tenant: RLS scopes each row to its store. Functions and
column grants are a separate surface, covered by the audit below.*

## 📸 Screenshots

| Ownly — owner dashboard | Tillr — customer ordering |
|---|---|
| ![Ownly navigation drawer over the owner dashboard](assets/ownly-app.png) | ![Tillr demo ordering screen with delivery, takeaway, and dine-in options](assets/tillr-app.png) |

## 🧠 Some decisions I'd defend in an interview

- **RLS as a database-level tenant boundary.** The intent is to avoid relying only on app-layer tenant filters. The audit found that function execute grants and column write permissions also needed attention; RLS alone did not cover those paths.
- **A demo store seeded from the same schema.** The demo uses throwaway data, and a nightly cron wipes its orders. The Tillr screenshot shows its demo banner, which labels orders as tests and states that no real payments are taken.
- **Security hardening done as its own pass.** A pentest-style review tightened the Content-Security-Policy to enforce mode and restricted anonymous access to owner-only tables. This repository does not include a CSP validation report or a surface-by-surface test matrix.

## 🔒 What a security audit found — and fixed

A security review I ran against the deployed system — adversarial testing against the actual database grants and API behaviour, not just a code read-through — turned up real bugs. Two worth naming, both fixed the same day:

- **Any signed-in session could rewrite another store's inventory.** A stock-adjustment function was meant to run only as an internal database trigger, but Postgres's default grants left it directly callable over the REST API with no ownership check — any logged-in session (including Tillr's anonymous-checkout flow, which still carries an authenticated role) could call it against *any* store's ID — a competitor's, not just their own. The fix: revoke direct execute access; the trigger doesn't need that grant to keep firing.
- **A client-writable `billing_plan` column let a store self-select a cheaper Stripe price.** Row-level security correctly limited writes to a store's own row, but nothing restricted *which* columns — an owner could patch their own `billing_plan` directly, and checkout trusted it at face value. The fix: revoke the blanket write grant, re-grant `UPDATE` on only the columns a store legitimately edits (loyalty settings), pull billing and Stripe fields out of client reach entirely.

The same pass also caught a process bug, not just a code bug: some earlier fixes had been applied straight to the database and never landed in a committed migration, so the schema in git no longer matched what was actually deployed. Part of the fix there was mechanical; part of it was going back and writing the migrations for changes that had only ever existed in the database, so the repo tells the truth about what is running again.

## 🧰 Stack

| Layer | Technology |
|---|---|
| Database | Postgres + Row-Level Security |
| Backend | Supabase Edge Functions, Realtime |
| Payments | Stripe Connect |
| Frontend | TypeScript / React, PWA |
| Hosting | Cloudflare Pages |

More work: [augustobastos.pages.dev](https://augustobastos.pages.dev)
