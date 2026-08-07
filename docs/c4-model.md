---
type: okf/node
id: "c4-model-v1"
title: "C4 Model — Context, Container, Component and Deployment Diagrams"
status: "verified"
last_updated: 2026-08-04
tags: ["c4", "architecture", "diagrams", "mermaid"]
sources:
  - title: "Route table and providers"
    url: "../src/App.tsx"
  - title: "Shared listing data layer"
    url: "../src/hooks/useCategoryListing.ts"
  - title: "Admin console"
    url: "../src/pages/Admin.tsx"
  - title: "Core schema and RPCs"
    url: "../supabase/migrations/20260710000000_fresh_project_schema.sql"
---

# C4 Model

Four levels, following Simon Brown's C4: **Context** (who uses it and what it
talks to), **Container** (the deployable/runnable pieces), **Component** (the
internals of the two significant containers), and **Deployment** (where it all
physically runs). Mermaid is used rather than the C4-DSL so these render in
GitHub, Notion and Obsidian without tooling.

---

## Level 1 — System Context

```mermaid
flowchart TB
  buyer["👤 <b>Buyer / Collector</b><br/>Anonymous. Identified only by a<br/>name + phone + localStorage UUID.<br/>Mobile-first."]
  seller["👤 <b>Seller / Admin</b><br/>Single Supabase Auth user.<br/>Lists stock, runs sales and breaks,<br/>confirms payment by hand."]

  sys["<b>Card Claimer</b><br/>Live-sale storefront where buyers claim<br/>scarce physical stock in real time and<br/>finish the purchase over WhatsApp."]

  wa["<b>WhatsApp</b><br/>wa.me deep links + a community group.<br/>Where payment is actually agreed."]
  yt["<b>YouTube</b><br/>iframe embed of the live box-break stream."]
  gem["<b>Google Gemini</b><br/>Vision: identify a photographed card.<br/>Search grounding: estimate a price."]
  ptcg["<b>pokemontcg.io</b><br/>Card catalog, images, USD market prices.<br/>No API key."]
  ppt["<b>PokemonPriceTracker</b><br/>Japanese-print price, used only as a<br/>labeled fallback proxy."]
  fx["<b>open.er-api.com</b><br/>Live USD→INR rate, cached 1 hour."]
  xp["<b>Yanks Diecast</b><br/>Sister storefront, cross-promo link."]

  buyer -->|"browses, claims, chats"| sys
  seller -->|"manages catalog and sales"| sys
  sys -->|"pre-filled cart message"| wa
  buyer -.->|"sends it, negotiates payment"| wa
  seller -.->|"replies with payment details"| wa
  sys -->|"embeds stream"| yt
  sys -->|"identify + price a scan"| gem
  sys -->|"catalog + price lookup"| ptcg
  sys -->|"proxy price fallback"| ppt
  sys -->|"FX rate"| fx
  sys -->|"outbound link"| xp

  style sys fill:#1e3a8a,stroke:#60a5fa,stroke-width:3px,color:#fff
  style buyer fill:#334155,color:#fff
  style seller fill:#334155,color:#fff
```

**The defining boundary**: money is *outside* the system. Card Claimer's job ends
at generating a WhatsApp message. Everything about the data model, the 10-minute
claim TTL and the meaning of the `transactions` table follows from that.

---

## Level 2 — Containers

```mermaid
flowchart TB
  buyer["👤 Buyer"]
  seller["👤 Admin"]

  subgraph sys["Card Claimer"]
    direction TB

    spa["<b>Storefront SPA</b><br/><i>React 18 · TypeScript · Vite · Tailwind · shadcn/ui</i><br/>All 11 routes, buyer and admin UI in one bundle.<br/>Static assets, no SSR."]

    pgrst["<b>Data API</b><br/><i>Supabase PostgREST</i><br/>Table SELECTs and RPC POSTs.<br/>Enforces RLS per request."]

    rt["<b>Realtime</b><br/><i>Supabase Realtime · WebSocket</i><br/>postgres_changes on 6 published tables.<br/>RLS-aware."]

    auth["<b>Auth</b><br/><i>Supabase GoTrue</i><br/>Email + password. One user.<br/>No public sign-up."]

    store["<b>Object Storage</b><br/><i>Supabase Storage</i><br/>4 public buckets: card-images,<br/>card-videos, prize-images, break-images.<br/>No image transforms."]

    ef["<b>identify-card</b><br/><i>Supabase Edge Function · Deno</i><br/>Only server-side code we own.<br/>Exists to hold GEMINI_API_KEY."]

    db[("<b>Application Database</b><br/><i>PostgreSQL</i><br/>9 tables · ~30 functions · RLS everywhere.<br/><b>Holds all business logic.</b>")]
  end

  cdn["<b>Vercel Edge CDN</b><br/>Static hosting + SPA rewrite"]

  gem["Google Gemini"]
  ptcg["pokemontcg.io"]
  ppt["PokemonPriceTracker"]
  fx["open.er-api.com"]
  wa["WhatsApp"]
  yt["YouTube"]

  buyer --> cdn
  seller --> cdn
  cdn -->|"serves bundle"| spa

  spa -->|"HTTPS · anon key<br/>or admin JWT"| pgrst
  spa <-->|"WSS"| rt
  spa -->|"signInWithPassword"| auth
  spa -->|"upload · public GET"| store
  spa -->|"functions.invoke"| ef
  spa -->|"fetch, no key"| ptcg
  spa -->|"fetch"| fx
  spa -->|"window.open"| wa
  spa -.->|"iframe"| yt

  pgrst --> db
  rt --> db
  auth --> db
  store --> db
  ef --> gem
  ef --> ppt

  style spa fill:#1e40af,color:#fff
  style db fill:#166534,stroke:#22c55e,stroke-width:3px,color:#fff
  style ef fill:#78350f,color:#fff
  style sys fill:#0f172a,stroke:#475569,color:#fff
```

### Container responsibilities

| Container | Technology | Owns | Explicitly does **not** own |
|---|---|---|---|
| Storefront SPA | React 18.3, TS 5.8, Vite 5, Tailwind 3.4, shadcn/ui | All rendering, filter/sort/search, cart display, countdown timers, WhatsApp message composition, realtime subscription wiring, client-side video compression | Stock arithmetic, price authority, order creation, expiry enforcement |
| Data API (PostgREST) | Supabase-managed | Row-level authorization on every request, RPC dispatch | Any business rule of its own |
| Application Database | PostgreSQL | **Stock decrement, claim TTL, order finalization, site-wide discounts, leaderboard aggregation, slot uniqueness, analytics aggregation** | Presentation, notifications, scheduling |
| Realtime | Supabase-managed | Change fan-out to open tabs | Delivery guarantees the app relies on (all reads have an explicit fetch path) |
| Auth | GoTrue | Admin session, JWT refresh | Buyer identity — buyers have no account |
| Object Storage | Supabase Storage | Serving media at original resolution | Resizing, watermarking, CDN transforms |
| identify-card | Deno on Supabase Edge | Holding the Gemini key, key rotation on rate limits, 20 s timeouts, grounded price search, JP proxy fallback | Writing anything to the database |

---

## Level 3a — Components of the Storefront SPA

```mermaid
flowchart TB
  subgraph shell["App shell — src/App.tsx"]
    qc["QueryClientProvider<br/>⚠️ mounted, zero useQuery calls"]
    tp["TooltipProvider"]
    t1["Radix Toaster<br/>⚠️ mounted, zero useToast callers"]
    t2["Sonner Toaster<br/>✅ the one actually used"]
    at["AnalyticsTracker<br/>headless, 1 row per tab session"]
    br["BrowserRouter · 11 routes"]
  end

  subgraph pages["Route components — src/pages"]
    idx["Index<br/>hub: 4 tiles + aggregate counts"]
    cat["Singles · Slabs · Sealed · Accessories<br/>4 near-identical pages,<br/>own filter state each"]
    lb["Leaderboard<br/>monthly + per-sale"]
    bk["BoxBreaks → LiveBreak → LiveBreakChat"]
    adm["Admin — 1,318 lines, 5 tabs<br/>⚠️ largest file in the repo"]
  end

  subgraph hooks["Hooks — src/hooks"]
    ucl["useCategoryListing(itemType)<br/>fetch + realtime + sweep + claim/unclaim<br/>✅ the shared data layer"]
    ub["useBuyer()<br/>localStorage identity + session UUID"]
    uas["useAdminSession()<br/>Supabase auth session"]
  end

  subgraph dom["Domain components — src/components"]
    ng["NameGate — blocking identity modal"]
    ct["CardTile — 429 lines: media, badges,<br/>tier ring, stepper, claim, countdown"]
    cg["CategoryGrid — skeleton / empty / grid"]
    cs["CheckoutSheet — cart + WhatsApp message"]
    cc["ClaimCountdown · CountdownTimer"]
    mc["MediaCarouselDialog — embla"]
    sg["SlotGrid · BreakCheckoutSheet · LiveChat"]
    mgr["Admin managers: SaleManager · SaleTimeManager ·<br/>SiteWideSaleManager · BoxBreakManager ·<br/>SalesHistory · AnalyticsDashboard · EditCardDialog ·<br/>CardScanner · AdditionalPhotosField"]
  end

  subgraph libs["Libraries — src/lib and src/config.ts"]
    cfg["config.ts — brand + commerce constants"]
    cm["categoryMeta.ts — the 4 categories"]
    cv["cardVision.ts — Edge Function client"]
    pt["pokemontcg.ts"]
    fxr["fxRate.ts — 1h cached USD→INR"]
    vc["videoCompression.ts + videoUpload.ts"]
    di["deviceInfo.ts — ua-parser-js"]
    yts["youtube.ts"]
  end

  subgraph ui["src/components/ui — 49 vendored shadcn files"]
    uinote["⚠️ 30 of 49 are imported nowhere"]
  end

  sbc["src/integrations/supabase/client.ts<br/>the single Supabase client"]
  types["src/integrations/supabase/types.ts<br/>898 generated lines — the type contract"]

  shell --> pages
  idx --> ng & cfg & cm
  cat --> ucl & ng & cg & cs & cfg & cm
  cg --> ct
  ct --> cc & mc & cfg
  cs --> cfg & ub
  bk --> sg & mgr & yts
  adm --> uas & mgr & cv & pt & fxr & vc
  ucl --> sbc & ub
  ub -.-> LS[("localStorage")]
  cv --> sbc
  pages --> ui
  dom --> ui
  sbc --> types

  style qc fill:#7f1d1d,color:#fff
  style t1 fill:#7f1d1d,color:#fff
  style adm fill:#7f1d1d,color:#fff
  style uinote fill:#7f1d1d,color:#fff
  style ucl fill:#14532d,color:#fff
  style t2 fill:#14532d,color:#fff
  style sbc fill:#1e40af,color:#fff
```

### What this diagram tells you about changing the app

- **`useCategoryListing` is the good abstraction.** Four category pages share one
  data layer (fetch + realtime + sweep + claim/unclaim); each page keeps only its
  own filter UI, because the facets genuinely differ. Adding a fifth category is
  a new 200-line page reusing this hook, not new plumbing.
- **`CardTile` (429 lines) and `Admin` (1,318 lines) are the two hotspots.** Any
  visual redesign hits `CardTile`; any workflow change hits `Admin`. Both are
  where a re-skin will actually cost time.
- **`config.ts` + `categoryMeta.ts` + `index.css` form the de-facto brand layer**,
  which is why re-theming is cheap. See
  [adaptation/theming-and-white-labeling.md](./adaptation/theming-and-white-labeling.md).
- **Three components marked red are pure removable weight**: an unused query
  cache, a second unused toast system, and 30 unused vendored UI files.

---

## Level 3b — Components of the Application Database

The database is the real backend, so it gets a component diagram of its own.

```mermaid
flowchart TB
  subgraph public_iface["Public surface — granted to anon"]
    p1["claim_units<br/><i>row lock, stock decrement, claim insert</i>"]
    p2["release_claim<br/><i>session-scoped, returns stock</i>"]
    p3["finalize_claims<br/><i>claims → checked_out, writes transactions</i>"]
    p4["get_my_claims<br/><i>SECURITY DEFINER read-around for RLS</i>"]
    p5["release_expired_claims<br/><i>the TTL sweep</i>"]
    p6["claim_break_slots · release_break_slot_claim<br/>finalize_break_slot_claims<br/>release_expired_break_slot_claims"]
    p7["list_sales · get_monthly_leaderboard<br/>get_sale_leaderboard"]
  end

  subgraph admin_iface["Admin surface — authenticated only, revoked from PUBLIC + anon"]
    a1["start_sale · end_active_sale · update_sale_prize"]
    a2["mark_claim_as_sold · admin_release_claim"]
    a3["apply_site_wide_sale · end_site_wide_sale"]
    a4["admin_mark_break_slot_sold<br/>admin_release_break_slot_claim"]
    a5["7 analytics RPCs:<br/>get_visitor_overview · get_device_breakdown<br/>get_browser_breakdown · get_os_breakdown<br/>get_daily_visits · get_top_entry_pages<br/>get_referrer_breakdown"]
  end

  subgraph tables["Tables — RLS enabled on all"]
    t_cards[("cards<br/>catalog + denormalized stock")]
    t_claims[("claims<br/>ephemeral cart, 10-min TTL<br/>🔒 no public SELECT")]
    t_sales[("sales<br/>one active enforced by partial unique index")]
    t_tx[("transactions<br/>append-only ledger, order_id groups a checkout")]
    t_set[("app_settings<br/>singleton id = 1")]
    t_bb[("box_breaks")]
    t_bsc[("break_slot_claims<br/>UNIQUE(break_id, slot_number)")]
    t_chat[("live_chat_messages<br/>public INSERT")]
    t_sv[("site_visits<br/>🔒 admin SELECT, public INSERT +<br/>column-grant UPDATE(last_seen_at)")]
  end

  subgraph triggers["Triggers"]
    tr1["trg_new_card_site_wide_sale<br/>BEFORE INSERT ON cards<br/><i>backs up sale_price and applies the live<br/>discount to a card added mid-sale</i>"]
  end

  subgraph realtime["Realtime publication"]
    rtp["<b>Published (7):</b> cards · claims · app_settings · transactions<br/>box_breaks · break_slot_claims · live_chat_messages<br/><br/>site_visits has REPLICA IDENTITY FULL but was<br/>never added to the publication — no fan-out<br/><br/>all 8 are REPLICA IDENTITY FULL"]
  end

  p1 --> t_cards & t_claims
  p2 --> t_cards & t_claims
  p3 --> t_claims & t_tx
  p4 --> t_claims
  p5 --> t_cards & t_claims
  p6 --> t_bsc & t_bb
  p7 --> t_sales & t_tx
  a1 --> t_sales
  a2 --> t_claims & t_tx & t_cards
  a3 --> t_cards & t_set
  a4 --> t_bsc
  a5 --> t_sv
  tr1 --> t_cards
  tables --> rtp

  style t_claims fill:#7f1d1d,color:#fff
  style t_sv fill:#7f1d1d,color:#fff
  style admin_iface fill:#14532d,color:#fff
  style public_iface fill:#1e3a8a,color:#fff
```

**The pattern to preserve**: buyers never touch a table directly for anything
that matters. `claims` and `site_visits` are readable only through
`SECURITY DEFINER` functions or an admin session, which is what keeps buyer phone
numbers and traffic data off the public API.

---

## Level 4 — Deployment

```mermaid
flowchart TB
  subgraph device["Buyer / Admin device"]
    br2["Browser<br/>+ optional PWA install<br/>(manifest.json, standalone, portrait)"]
    ls2[("localStorage · sessionStorage")]
  end

  subgraph vercel["Vercel — global edge"]
    v1["Static site<br/>/dist from `vite build`<br/>vercel.json: /(.*) → /index.html"]
    v2["Build: Rollup + SWC on push"]
  end

  subgraph sbproject["Supabase project — single region"]
    direction TB
    sb1["PostgreSQL instance<br/>+ RLS + 30 functions + 1 trigger"]
    sb2["PostgREST"]
    sb3["Realtime server"]
    sb4["GoTrue"]
    sb5["Storage + CDN<br/>card-images · card-videos<br/>prize-images · break-images"]
    sb6["Edge Runtime (Deno)<br/>identify-card<br/>secrets: GEMINI_API_KEY(S),<br/>GEMINI_VISION_MODEL,<br/>POKEMONPRICETRACKER_API_KEY"]
  end

  subgraph manual["Out-of-band — no CI"]
    m1["supabase/migrations/*.sql<br/>applied by hand"]
    m2["supabase functions deploy identify-card"]
    m3["Admin user created in the<br/>Supabase dashboard"]
  end

  ext["Gemini · pokemontcg.io · PokemonPriceTracker<br/>open.er-api.com · YouTube · wa.me"]

  br2 -->|"HTTPS"| v1
  br2 -->|"HTTPS · WSS"| sbproject
  br2 -->|"HTTPS"| ext
  v2 --> v1
  m1 -.-> sb1
  m2 -.-> sb6
  m3 -.-> sb4
  sb6 --> ext

  style manual fill:#78350f,stroke:#f59e0b,color:#fff
  style sbproject fill:#052e16,stroke:#22c55e,color:#fff
```

### Deployment facts

| Aspect | State |
|---|---|
| Environments | **One.** No staging project, no preview database. Vercel preview builds would point at production Supabase. |
| Frontend CI/CD | Vercel on push. No lint/typecheck/test gate (`.github/workflows` does not exist). |
| Database CD | **Manual.** Migrations applied by hand; 12 of 20 files are legacy or untimestamped, so the folder is not a replayable history. |
| Edge Function CD | **Manual** — `supabase functions deploy identify-card --project-ref govervcxumkbpmnnotpr`. |
| Secrets | 3 public `VITE_*` vars in Vercel (and committed to `.env` in the repo — publishable keys, so not a leak, but a bad default); 3 server secrets in Supabase Edge config. |
| Scheduled jobs | **None.** See [system-architecture.md](./system-architecture.md) §9. |
| Regions | Whatever region the Supabase project was created in; the CDN is global but every API call is single-region. For an India-focused audience (INR, IST, Indian phone defaults) confirm the project sits in `ap-south-1`. *Assumption — verify in the dashboard.* |
| Backups | Supabase plan default only. No application-level export of `transactions`. |

---

## Cross-level traceability

| C4 element | Source of truth |
|---|---|
| Storefront SPA | `src/` (59 app files + 49 vendored UI files) |
| Route list | `src/App.tsx:26-40` |
| Shared listing data layer | `src/hooks/useCategoryListing.ts` |
| Buyer identity | `src/hooks/useBuyer.ts` |
| Brand + commerce config | `src/config.ts`, `src/lib/categoryMeta.ts`, `src/index.css` |
| Public + admin RPC surface | `supabase/migrations/20260710000000_fresh_project_schema.sql`, `20260727000000_box_breaks.sql`, `20260728030000_site_visits_analytics.sql` |
| Realtime publication | same migrations, `ALTER PUBLICATION supabase_realtime ADD TABLE` |
| Storage buckets + policies | same migrations, `INSERT INTO storage.buckets` |
| Edge Function | `supabase/functions/identify-card/index.ts` |
| Static hosting rewrite | `vercel.json` |
