---
type: okf/bundle
version: "0.2"
title: "Card Claimer — System, Product & Adaptation Knowledge Base"
description: "Architecture, data model, tech stack, infrastructure, UX flows and white-labeling guide for the Yanks TCG live-sale storefront (card-claimer-yn), written so the same codebase can be re-themed and re-pointed at other claim-and-checkout businesses without a rebuild."
maintainers: ["yashdeep@avpschool.in"]
last_updated: 2026-08-04
tags: ["architecture", "supabase", "react", "vite", "ecommerce", "live-sale", "white-label", "ux"]
---

# Card Claimer — System, Product & Adaptation Knowledge Base

This bundle documents **one specific running system**: a React + Supabase
storefront where a seller lists collectible cards, buyers *claim* units in real
time during a scheduled "live sale", and checkout happens by handing the cart to
WhatsApp. The repository is `simplethingslabs/card-claimer-yn`, deployed as a
static SPA on Vercel against a single Supabase project.

It was written to answer one question: **what can be changed cheaply, and what
cannot**, for someone who intends to re-skin and re-point this system at other
businesses that operate the same way (a seller with inventory, a time-boxed
drop, a claim-then-negotiate checkout) rather than rebuild it.

## Read this in order

| # | Node | Audience | What it answers |
|---|------|----------|-----------------|
| 1 | [architecture-for-product-managers.md](./architecture-for-product-managers.md) | PM, founder, non-engineer builder | Plain-language tour: what the pieces are, what each costs, what "changing the theme" actually touches |
| 2 | [system-architecture.md](./system-architecture.md) | Engineer | Runtime topology, trust boundaries, request paths, state ownership |
| 3 | [c4-model.md](./c4-model.md) | Engineer, architect | C4 Context / Container / Component / Deployment diagrams |
| 4 | [data-model.md](./data-model.md) | Engineer | Every table, column, constraint, RLS policy and RPC, plus ERD |
| 5 | [system-design-flows.md](./system-design-flows.md) | Engineer, PM | Sequence diagrams for the 8 flows that define the product |
| 6 | [tech-stack.md](./tech-stack.md) | Engineer | Every dependency, why it's there, and which ones are dead weight |
| 7 | [infrastructure-requirements.md](./infrastructure-requirements.md) | Engineer, ops, finance | Hosting, env vars, quotas, the binding constraints, cost at 3 scales |
| 8 | [technical-assessment.md](./technical-assessment.md) | Engineer, tech lead | Honest pros/cons, ranked risks, and a concrete simplification plan |
| 9 | [product/ux-current-state.md](./product/ux-current-state.md) | PM, designer | Every screen and flow as it exists today, documented screen by screen |
| 10 | [product/ux-friction-and-simplification.md](./product/ux-friction-and-simplification.md) | PM, designer | 18 ranked friction points with fixes and effort estimates |
| 11 | [adaptation/theming-and-white-labeling.md](./adaptation/theming-and-white-labeling.md) | Builder | The actual playbook: 4 layers of change, file-by-file, cheapest first |

## The system in six sentences

1. A **static React SPA** (Vite build, no server-side rendering) is served from
   Vercel's CDN; there is no application backend of our own.
2. The browser talks **directly to Supabase** — PostgREST for table reads,
   Postgres functions (RPC) for every write that matters, Storage for media,
   Realtime for live updates, and one Deno Edge Function for the AI card
   scanner.
3. **Business logic lives in Postgres**, not in JavaScript: claiming stock,
   expiring claims, finalizing orders, applying a site-wide discount and
   computing leaderboards are all `SECURITY DEFINER` SQL functions.
4. **Buyers have no accounts.** Identity is a name + phone + random session UUID
   in `localStorage`; that UUID is the only thing tying a cart to a person.
5. **No money moves through the app.** "Checkout" builds a pre-filled WhatsApp
   message and marks the claims as checked out — payment is a human
   conversation afterwards.
6. **One Supabase Auth user is the admin.** Any authenticated session has full
   write access to everything; there is no role table and no public sign-up.

## What is genuinely cheap to change

Ranked by effort, detailed in
[adaptation/theming-and-white-labeling.md](./adaptation/theming-and-white-labeling.md):

- **Hours** — colors, gradients, shadows, radius, fonts, logo, favicon, PWA
  manifest, seller name, currency symbol, WhatsApp number, claim window,
  shipping thresholds (all live in `src/index.css`, `src/config.ts`, `public/`).
- **Days** — category names/icons/routes, page copy, feature on/off (box breaks,
  leaderboard, slabs, AI scanner), listing form fields.
- **Weeks** — replacing the Pokémon-specific catalog lookup, the `item_type`
  CHECK constraint, and the vocabulary hardcoded inside JSX ("Trainer", "XP",
  "Pokémon Cards Live Sale") with configuration.
- **Rebuild territory** — in-app payments, buyer accounts/order history, SEO-
  indexable product pages, multi-tenancy in one deployment.

## Conventions used in this bundle

- Code references are `path/to/file.ts:line` where a specific line matters.
- Diagrams are Mermaid, renderable by GitHub, Notion, Obsidian and most AI
  agents without a plugin.
- **Verified** means read from source in this repository at
  commit `3026be2`. **Inferred** means a reasonable reading not confirmed by
  running the system. **Assumption** means it needs checking before you rely on
  it. Every non-obvious claim is tagged.
- Quotas and prices for third-party services are marked as needing verification
  against the current vendor pricing page; they move.
