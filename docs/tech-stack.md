---
type: okf/node
id: "tech-stack-v1"
title: "Tech Stack — Every Dependency, Its Role, and What Is Dead Weight"
status: "verified"
last_updated: 2026-08-04
tags: ["tech-stack", "dependencies", "react", "vite", "tailwind", "supabase", "audit"]
sources:
  - title: "Dependency manifest"
    url: "../package.json"
  - title: "Project conventions as originally written"
    url: "../AI_RULES.md"
  - title: "Build configuration"
    url: "../vite.config.ts"
  - title: "Design tokens"
    url: "../src/index.css"
  - title: "Tailwind theme extension"
    url: "../tailwind.config.ts"
---

# Tech Stack

## 1. At a glance

| Layer | Choice | Version | Verdict |
|---|---|---|---|
| Language | TypeScript | 5.8 | ✅ strict-ish, used throughout |
| UI runtime | React | 18.3 | ✅ |
| Build | Vite + `@vitejs/plugin-react-swc` | 5.4 / 3.11 | ✅ fast, SWC not Babel |
| Styling | Tailwind CSS | 3.4 | ✅ with a real token layer |
| Components | shadcn/ui on Radix | 27 Radix packages | ⚠️ 30 of 49 vendored files unused |
| Routing | react-router-dom | 6.30 | ✅ v7 flags pre-enabled |
| Backend | Supabase (`@supabase/supabase-js`) | 2.105 | ✅ the whole backend |
| Server state | TanStack Query | 5.83 | ❌ **installed, provider mounted, zero usages** |
| Forms | react-hook-form + zod | 7.61 / 3.25 | ❌ **installed, zero usages** |
| Toasts | sonner **and** Radix toast | 1.7 / 1.2 | ⚠️ two systems, one used |
| Charts | recharts | 2.15 | ✅ analytics dashboard only |
| Dates | date-fns + date-fns-tz | 3.6 / 3.2 | ✅ tz used for hardcoded IST |
| Icons | lucide-react | 0.462 | ✅ |
| Carousel | embla-carousel-react | 8.6 | ✅ used directly, not via the shadcn wrapper |
| UA parsing | ua-parser-js | 2.0 | ✅ analytics device buckets |
| Theme | next-themes | 0.3 | ⚠️ only reached by `ui/sonner.tsx`; app is dark-only |
| Tests | vitest + Testing Library + jsdom | 3.2 | ❌ configured, one placeholder assertion |
| Hosting | Vercel | — | ✅ static + SPA rewrite |
| Package manager | ambiguous | — | ⚠️ `bun.lockb`, `pnpm-lock.yaml` **and** `package-lock.json` all committed |

**52 runtime dependencies, 21 dev.** 27 of the 52 are `@radix-ui/*`.

## 2. Frontend

### React 18 + Vite + SWC

Standard modern SPA setup. Notable configuration in `vite.config.ts`:

- `resolve.dedupe` for `react`, `react-dom` and both JSX runtimes — a fix for
  duplicate-React errors, typical of AI-builder scaffolds.
- `optimizeDeps.exclude: ["date-fns-tz"]` with a comment about import issues.
- `hmr.overlay: false` — the error overlay is suppressed in dev, which makes
  runtime errors easier to miss while developing. Worth reconsidering.
- `lovable-tagger` runs only in development mode.
- No manual chunking, no bundle analysis. **Everything ships in one bundle,
  including the entire 1,318-line admin console and the recharts library, to every
  buyer.** Route-level `React.lazy` on `/admin` alone would be a meaningful win —
  see [technical-assessment.md](./technical-assessment.md) O4.

### Tailwind + the design token layer

This is the best-engineered part of the frontend and the reason re-theming is
cheap. `src/index.css` defines everything as HSL CSS variables:

```
--background --foreground --card --popover --primary --primary-glow
--secondary --muted --accent --success --destructive --border --input --ring
--radius
--gradient-hero --gradient-gold --gradient-card
--shadow-glow --shadow-card --shadow-claim
--gradient-slab-gold --gradient-slab-holo --shadow-slab-gold --shadow-slab-holo
```

`tailwind.config.ts` maps each to a Tailwind color name, so components write
`bg-primary` / `text-success` and never a literal color. Custom keyframes add
`pulse-glow`, `claim-pop`, `fade-in` and `gradient-pan` (the panning shimmer used
for the slab rings — chosen over a rotating conic gradient for browser support).

Two caveats:

- **`darkMode: ["class"]` is configured but there is only one palette.** The
  `:root` block *is* the dark theme; there is no `.dark` block and no light
  theme. Adapting this for a business that wants a light storefront means
  authoring a second palette, not flipping a switch.
- Nine `--sidebar-*` variables are **light-theme values** left over from the
  shadcn sidebar scaffold, and `ui/sidebar.tsx` (637 lines) is imported nowhere.
- `body` carries a four-layer background: two radial gradient washes, a dark
  overlay, and `url('/bg-pattern.jpg')` (167 KB) tiled at 700px with
  `background-attachment: fixed`. Fixed attachment is a known scroll-performance
  cost on mobile Safari. *Inferred — not profiled here.*

### shadcn/ui — the audit

49 files in `src/components/ui`, **30 imported nowhere in the app**:

```
accordion  alert  alert-dialog  aspect-ratio  avatar  breadcrumb  carousel
checkbox  collapsible  command  context-menu  drawer  dropdown-menu  form
hover-card  input-otp  menubar  navigation-menu  pagination  progress
radio-group  resizable  scroll-area  separator  sidebar  slider  table
toggle  toggle-group  use-toast
```

Tree-shaking keeps them out of the production bundle, so this is not a
performance problem — it's a **comprehension and maintenance** problem: 3,954
lines of vendored code, most of it noise, and 27 Radix packages in
`package.json` for roughly 14 actually-used primitives. Anyone new to the repo
must work out which half is real.

Note `ui/form.tsx` being unused is the proof that **react-hook-form and zod are
entirely unused** despite `AI_RULES.md` mandating them for all forms.

### The three convention violations in `AI_RULES.md`

The repo ships its own conventions file. Three of its rules are not followed:

| Rule | Reality |
|---|---|
| "For server state … use TanStack Query" | Zero `useQuery`/`useMutation`. All fetching is `useEffect` + `useState`, hand-rolled in each page or in `useCategoryListing`. |
| "Implement all forms using react-hook-form … zod schemas for validation" | No form uses either. Admin's ~20-field form is individual `useState` hooks with inline validation. |
| "Use sonner for all toast notifications" | Followed — but the Radix `<Toaster />` is *also* mounted in `App.tsx` with no callers. |

This matters for a practical reason: an AI coding agent (or a new contributor)
reading `AI_RULES.md` will write code in a style the rest of the codebase does not
use, producing two incompatible idioms. **Either adopt the libraries or rewrite
the rules** — the file is load-bearing for how this codebase gets extended.

## 3. Backend: Supabase

One dependency, `@supabase/supabase-js`, reaches all six services. Client
configuration is minimal (`src/integrations/supabase/client.ts`): `localStorage`
storage, `persistSession`, `autoRefreshToken`. Typed with the generated
`Database` type, so every query is type-checked against the schema.

| Service | Used for | Notes |
|---|---|---|
| PostgREST | Table SELECTs, all RPC calls | Some inserts go direct (`site_visits`, `live_chat_messages`, admin `cards`) |
| Auth | Single admin, email/password | No sign-up UI anywhere |
| Storage | 4 public buckets | No image transformation API in use |
| Realtime | `postgres_changes` on the 7 published tables | Mostly unfiltered subscriptions |
| Edge Functions | `identify-card` (Deno) | The only server-side code we own |
| Postgres | ~30 functions = the real API | Business logic lives here |

Note the **absent** Supabase features that would each remove app code: image
transformations (would fix the full-size-image egress issue), `pg_cron` (would fix
the sweep), and Database Webhooks (would enable "back in stock" notifications).

## 4. Dependencies to remove or adopt

| Dependency | Status | Recommendation |
|---|---|---|
| `@tanstack/react-query` | Provider mounted, 0 usages | **Adopt** — it directly solves the duplicated fetch/refetch/interval logic in `useCategoryListing`, `Index`, `Admin` and `BoxBreaks`, and gives request dedup across tabs. Second-best: remove it and the provider. |
| `react-hook-form`, `@hookform/resolvers`, `zod` | 0 usages | **Adopt for the admin form specifically** (~20 fields with cross-field rules like `sale_price <= price`) or remove all three. |
| Radix toast (`@radix-ui/react-toast`, `ui/toast.tsx`, `ui/toaster.tsx`, `ui/use-toast.ts`, `hooks/use-toast.ts`) | Mounted, 0 callers | **Remove.** Keep sonner. |
| `next-themes` | Reached only by `ui/sonner.tsx` | Keep only if a light theme is planned; otherwise inline sonner's theme and drop it. |
| 30 unused `ui/*` files + their Radix packages | Dead | **Delete**, listed above. Biggest single comprehension win. |
| `ua-parser-js` | Used | Keep. |
| `recharts` | Used in the admin dashboard only | Keep, but lazy-load with `/admin`. |
| `embla-carousel-react` | Used directly | Keep; delete the unused `ui/carousel.tsx` wrapper. |
| Three lockfiles | `bun.lockb` + `pnpm-lock.yaml` + `package-lock.json` | **Pick one and delete two.** Three lockfiles means three possible dependency graphs and non-reproducible builds depending on who installs. |

Effort for the whole cleanup: **half a day**, no behaviour change, and it removes
roughly 2,500 lines and ~15 packages.

## 5. External service dependencies

| Service | Called from | Auth | Failure behaviour | Coupling |
|---|---|---|---|---|
| **Google Gemini** | Edge Function | `GEMINI_API_KEY` / `GEMINI_API_KEYS` | Multi-key rotation on rate limits; 20 s abort per attempt; `502` to the client | AI scanner only |
| **pokemontcg.io** | **Browser, no key** | none | `console.error`, returns `[]` | 🔴 Pokémon-specific — replace per vertical |
| **PokemonPriceTracker** | Edge Function | `POKEMONPRICETRACKER_API_KEY` | Optional; skipped if unset | 🔴 Pokémon-specific |
| **open.er-api.com** | Browser | none | 1 h cache; falls back to the last good rate, then a hardcoded 90 | 🟡 USD→INR only; needed only if source prices are in USD |
| **YouTube** | iframe | none | Placeholder card when no video id | 🟡 Streaming is YouTube-only |
| **WhatsApp** (`wa.me`) | `window.open` | none | none — assumed installed | 🔴 The entire checkout |

The FX module has a good war story in its comments: a hardcoded 83.5 rate went
stale against a real ~96.6, silently underpricing every USD-converted listing by
~16%. It also records that Frankfurter (ECB) was rejected as a source because it
blocks cross-origin browser fetches — verified from a browser, not just
server-side. Both are exactly the notes you want in a codebase.

## 6. Testing and quality tooling

| Tool | State |
|---|---|
| vitest + jsdom + Testing Library + jest-dom | Fully configured, `src/test/setup.ts` even stubs `matchMedia` |
| Actual tests | **One**: `expect(true).toBe(true)` |
| ESLint 9 flat config + typescript-eslint + react-hooks + react-refresh | Configured; `@typescript-eslint/no-unused-vars` is **off** |
| CI | **None** — no `.github/workflows` |
| Type checking in CI | None (only via the Vercel build) |

The infrastructure for testing exists and is unused, which is the cheapest
possible starting point. The highest-value first tests are pure functions with
real business risk and no I/O:

1. Cart maths in `CheckoutSheet` — subtotal, the shipping threshold boundary at
   exactly ₹1500, grand total.
2. The WhatsApp message builder — it *is* the order document.
3. `arrivalWindowFor` / pre-order window derivation from `created_at`.
4. Discount percentage rounding in `CardTile`.
5. `extractYouTubeId` across URL shapes.
6. `getDeviceInfo` bucketing.

Then, with `pgTAP` or a seeded test project, the RPCs that actually carry money:
`claim_units` under concurrency, `finalize_claims` idempotency, and
`apply_site_wide_sale` → `end_site_wide_sale` round-trip fidelity.

## 7. PWA support

`public/manifest.json` — standalone display, portrait orientation, `#0e0e1a`
theme, 192/512 maskable icons. `index.html` adds apple-touch-icon and
`apple-mobile-web-app-*` meta. `PwaInstallBanner.tsx` prompts installation.

**There is no service worker**, so this is "add to home screen", not an offline
app. Reloading without a network shows a blank page. For a mobile-first live-sale
product a minimal service worker caching the shell and product images would be a
genuine upgrade — and would also cut Storage egress.

## 8. Line-count reality check

| Bucket | Lines |
|---|---|
| Hand-written app code (`src/`, excluding `ui/` and generated types) | ~8,650 |
| Generated Supabase types | 898 |
| Vendored shadcn `ui/` | 3,954 (~2,400 of it unused) |
| SQL migrations | ~1,100 |
| Edge Function | ~430 |
| **Effective codebase to understand** | **~6,000 lines**, of which `Admin.tsx` is 1,318 and `CardTile.tsx` is 429 |

That's a genuinely small system. Two files are 29% of it, and both are the files
a re-theme touches most.

## 9. What the stack means for adapting to another business

| Stack element | Portability |
|---|---|
| React + Vite + Tailwind + shadcn | 🟢 Fully portable |
| CSS variable token layer | 🟢 **The asset** — one file re-skins everything |
| Supabase + RPC business logic | 🟢 Portable; the claim/expire/finalize pattern is domain-agnostic |
| `src/config.ts` constants | 🟢 Designed for this |
| `categoryMeta.ts` | 🟡 Portable shape, Pokémon content |
| `item_type` CHECK constraint | 🔴 Must become a table |
| `pokemontcg.ts`, `cardVision.ts`, `CardScanner`, `identify-card` | 🔴 Pokémon-only; delete or replace per vertical |
| Slab/grading columns + visual tiers | 🔴 Trading-card-only (though "visual tier" generalizes well as "featured treatment") |
| `fxRate.ts` (USD→INR) | 🟡 Keep only if sourcing prices in USD |
| WhatsApp checkout | 🟡 Portable across WhatsApp-first markets; needs replacing elsewhere |
| Hardcoded IST in `SaleTimeManager` | 🔴 One-line fix, but it *is* hardcoded |

See [adaptation/theming-and-white-labeling.md](./adaptation/theming-and-white-labeling.md)
for the file-by-file plan.
