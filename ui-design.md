# Dakar — User Interface Design Specification

**Status:** v1.0 — the design authority for the Dakar operations console and client surface
**Audience:** privileged contributors (Law 4 applies: this document may use spec vocabulary; the repo never does — see §16 for the repo-safe vocabulary)
**Companions:** system_architecture.md (topology), agentic.md (agent behavior the UI surfaces), model.md (formal model), operator-runbook.md (operator procedures the UI accelerates), dakar-javadoc.md (code contracts)

This document is the complete product design specification for the Dakar user interface. Implementation proceeds from it with minimal ambiguity: every screen, flow, component, token, and state below exists because it serves a defined workflow against the real API surface. Where the UI needs read surfaces the API does not yet expose, §14 defines the exact contracts to build.

---

## 1. Purpose, principles, audiences

### 1.1 What the UI is for

Dakar's backend is an agentic freight desk: every mutation is a rule-checked, journaled action; automation (dispatcher, lifecycle coordinator) acts on the same desk humans use; negotiation advice is computed from episodic memory and offered to humans, never forced. The UI's job is to make that system **legible and operable**:

- Operators see what is happening, what the automation decided and why, and where their attention is needed.
- Operators act through the same governed actions (assign, quote, signal) with refusals rendered as first-class outcomes, not failures.
- Clients request deliveries, follow progress in client vocabulary, and accept quotes.

### 1.2 Core principles (binding)

1. **Simplicity first** — one primary action per context; everything else is secondary.
2. **Information density where appropriate** — the board and feeds are dense; forms are spacious.
3. **Progressive disclosure** — summary → selection → detail panel → full record.
4. **Consistent interaction patterns** — one selection model, one form pattern, one error pattern everywhere.
5. **Minimal cognitive load** — state is encoded visually and redundantly (shape + icon + label, never hue alone).
6. **Accessibility from the beginning** — WCAG 2.2 AA targets; the canvas has a DOM mirror (§13).
7. **Responsive** — desktop-first operator console that degrades gracefully to tablet; client surface is fully mobile-capable.
8. **Design around user tasks, not backend objects** — navigation is by job (Operations, Negotiation, Fleet…), not by table.
9. **Honest state** — no optimistic writes; refusals, staleness, and outages are rendered truthfully (§12.3, §13).

### 1.3 Audiences and actors

| Actor | Surface | Auth | Jobs |
|---|---|---|---|
| **Operator (staff)** | Operations console | Staff access key (header `X-Staff-Access-Key`) | Supervise the board, register trucks, assign, quote with advice, progress signals, watch disputes, respond to audit findings |
| **Client (requester)** | Client surface | None (public endpoints) | Request delivery, track status, accept quote |
| **Dispatcher automation** | Visible in console | n/a (backend actor) | Auto-assigns trucks; its decision traces are rendered in the activity feed and as graph annotations |
| **Lifecycle coordinator** | Visible in console | n/a (backend actor) | Expires stale quotes, flags overdue payment; its actions appear in the feed and as pressure cues on the board |
| **Negotiation advisor** | Inline in quote flow | n/a (computed on demand) | Suggests offer with floor/ceiling and episode evidence; human decides |

The automation actors are **first-class visible actors**: every feed entry carries its actor, and the console never hides that a machine acted.

---

## 2. Domain grounding

The UI renders exactly these objects and verbs — nothing invented:

- **Delivery**: client, tonnage, origin/destination coordinates, cargo class, deadline; lifecycle `REQUESTED → MATCHED → QUOTED → AGREED → IN_TRANSIT → AWAITING_PAYMENT → COMPLETED`, with `DISPUTED` (payment overdue) and `CANCELLED` branches. Terminal: `COMPLETED`, `CANCELLED`.
- **Truck**: capacity, position, driver hours, maintenance due; states `GARAGE, EN_ROUTE_EMPTY, LOADING, EN_ROUTE_LOADED, OFFLOADING, MAINTENANCE`; at most one assigned delivery.
- **Quote**: offer amount (currency from backend config, e.g. XOF — the UI is currency-agnostic and renders what the API returns), validity window, floor/ceiling policy bounds.
- **Signals** (staff-reported ground truth): `DEPARTED, ARRIVED_AT_PICKUP, LOADED, ARRIVED_AT_DESTINATION, PAYMENT_CONFIRMED, OFFLOADED`.
- **Operations journal**: the append-only record of every action (action, actor, delivery, payload, time).
- **Decision traces**: the automation's recorded reasoning.
- **Client status projection** (client vocabulary only, never shown to operators as the primary state): `RECEIVED, PREPARING_QUOTE, QUOTE_READY, SCHEDULED, DRIVER_EN_ROUTE, IN_TRANSIT, AWAITING_PAYMENT, PAYMENT_OVERDUE, DELIVERED, CANCELLED`.

Operator jobs the design optimizes, in priority order:

1. **Situational awareness** — "what is happening right now, and where do I look first?" (Operations board)
2. **Quote and move work** — get REQUESTED→AGREED fast, with advice (Negotiation)
3. **Keep the fleet healthy** — register, maintain, watch driver hours (Fleet)
4. **Supervise automation** — see what agents did and why; intervene by acting on the same desk (Activity + board annotations)
5. **Resolve exceptions** — disputes, expiring quotes, audit findings (board cues + runbook links)

---

## 3. Geographic model

Coordinates are the canonical spatial representation end-to-end (the domain stores latitude/longitude pairs; the API accepts `originLat/originLon/destinationLat/destinationLon` and truck `lat/lon`). Addresses are **display metadata**.

When a user specifies any spatial entity (origin, destination, truck position):

1. Free-text entry is geocoded through **Nominatim** (`/search`, `format=jsonv2`, `limit=5`) as the user types, debounced to respect the public instance's absolute limit of 1 request/second.
2. The user picks a result; its latitude/longitude become the stored value; the chosen `display_name` is kept as display metadata alongside the coordinates.
3. **Confidence**: Nominatim's result ordering and `importance` field drive an ambiguity cue — a single high-importance hit fills silently; multiple comparable hits render as a disambiguation list; zero hits fall through to manual entry.
4. **Manual correction is always available**: a coordinate pair field (decimal degrees, validated to ±90/±180) and, on the preview map inset, a draggable pin. Manual entry wins over geocoding.
5. Reverse geocoding (`/reverse`) labels raw coordinates that arrive without an address (e.g. truck positions) — best effort, cached, never blocking.
6. **Attribution**: "© OpenStreetMap contributors" is displayed adjacent to any geocoded result list and on the location inset, per ODbL.
7. **Operational posture**: the public `nominatim.openstreetmap.org` instance is for development and demo volume only (≤1 rps, results cached in `sessionStorage` keyed by normalized query). Production self-hosts Nominatim (containerized, same API) — the UI only needs a base-URL config. No API key exists in either mode; nothing to secrete.

The project is region-agnostic. "Dakar" is the product name, nothing more: no default region, no canned facility directory. Facilities appear on the board because operations reference them (§10.2).

---

## 4. Design system

Built by studying Cloudscape's *philosophy* — generous spacing rhythm, strong hierarchy, restrained chrome, dense-but-calm data surfaces — without copying its look, components, or tokens.

### 4.1 Typography

Font: **Source Sans 3** (variable, vendored woff2 in the repo — no font CDN), fallback `system-ui, -apple-system, "Segoe UI", sans-serif`. Code/mono contexts: `ui-monospace, "Cascadia Mono", monospace`.

| Role | Size/line | Weight | Usage |
|---|---|---|---|
| display | 28/34 | 640 | Page title on empty/landing states only |
| h1 | 22/28 | 640 | Screen title |
| h2 | 18/24 | 620 | Panel/section title |
| h3 | 15/20 | 620 | Card title, drawer title |
| h4–h6 | 13/18 | 620 | Sub-groupings; h5/h6 add letter-spacing 0.02em + uppercase |
| subtitle | 14/20 | 400 | Screen/section descriptors, secondary ink |
| body | 14/20 | 400 | Default |
| body-strong | 14/20 | 620 | Inline emphasis, table key cells |
| caption | 12/16 | 400 | Timestamps, meta, helper text |
| label | 12/16 | 620 | Form labels, chip text, uppercase h6-style optional |
| code | 13/18 | 440 mono | Identifiers, request ids, trace excerpts |
| figure | 24/28 | 640 tabular-lining | Stat-tile values (`font-variant-numeric: tabular-nums` only in aligned columns) |

### 4.2 Spacing, radius, elevation, borders, motion

- **Spacing scale** (4px base): 2, 4, 8, 12, 16, 24, 32, 48, 64. Panel padding 16; card padding 12; form row gap 12; section gap 24.
- **Corner radius**: 4 (controls, chips), 8 (cards, panels), 12 (dialogs, drawers). Never fully-round rectangles; fully-round only for dot markers and avatars.
- **Elevation**: level 0 flat (board, tables), level 1 `0 1px 2px rgba(0,0,0,.08)` (cards), level 2 `0 4px 16px rgba(0,0,0,.16)` (drawers, dialogs, toasts). Dark mode halves shadow alpha and adds a hairline border instead.
- **Borders**: hairline `1px` in the border token; separators prefer whitespace first, hairline second, never heavy rules.
- **Motion**: 120ms ease-out for hover/focus washes; 200ms ease-in-out for drawers/dialogs; vehicle interpolation on the board is continuous (§10.4). `prefers-reduced-motion: reduce` disables interpolation animation (positions jump per poll), pulses, and drawer slides (instant swap). Nothing meaningful is conveyed by motion alone.

### 4.3 Brand and semantic palette

Token prefix `--dk-`. Naming uses **ink / surface / accent** vocabulary (firewall-safe by construction). Values below are the validated reference set (validated with the palette validator against both surfaces); they are swappable in one file without touching component styles.

| Token | Light | Dark | Role |
|---|---|---|---|
| `--dk-surface` | `#fcfcfb` | `#1a1a19` | Board/chart surface |
| `--dk-plane` | `#f9f9f7` | `#0d0d0d` | Page background |
| `--dk-ink` | `#0b0b0b` | `#ffffff` | Primary text |
| `--dk-ink-2` | `#52514e` | `#c3c2b7` | Secondary text |
| `--dk-ink-3` | `#898781` | `#898781` | Muted/meta |
| `--dk-hairline` | `#e1e0d9` | `#2c2c2a` | Gridlines, separators |
| `--dk-edge-rest` | `#c3c2b7` | `#383835` | Baselines, inactive board connections |
| `--dk-border` | `rgba(11,11,11,.10)` | `rgba(255,255,255,.10)` | Rings, card borders |
| `--dk-accent` | `#2a78d6` | `#3987e5` | Primary actions, focus, links, selection |
| `--dk-accent-2` | `#1baf7a` | `#199e70` | Secondary accent (client surface highlights) |
| `--dk-good` | `#0ca30c` | `#0ca30c` | Health: good (completions, confirmations) |
| `--dk-warn` | `#fab219` | `#fab219` | Health: attention (expiring quote, payment near due, maintenance due) |
| `--dk-serious` | `#ec835a` | `#ec835a` | Health: degraded (advice unavailable, stale data) |
| `--dk-crit` | `#d03b3b` | `#d03b3b` | Health: critical (DISPUTED, audit divergence) |
| `--dk-good-text` | `#006300` | `#0ca30c` | Success text on surfaces |

Dark mode is **selected, not flipped**: dark values are their own steps validated against the dark surface. Both themes ship from day one; the OS preference (`prefers-color-scheme`) is the default and a toggle stamps `data-theme` on the root, which wins both ways.

**Health colors are reserved.** They mark state of health only, always paired with an icon and a label, and are never used as lifecycle-stage identity.

### 4.4 Lifecycle stage ramp (ordinal)

Delivery lifecycle stage is **ordinal**, encoded as one blue ramp light→deep (validated for ordinal use: lightest step ≥2:1 on its surface):

| Stage | Light | Dark |
|---|---|---|
| REQUESTED | `#86b6ef` | `#184f95` |
| MATCHED | `#5598e7` | `#2a78d6` |
| QUOTED | `#2a78d6` | `#5598e7` |
| AGREED | `#1c5cab` | `#86b6ef` |
| IN_TRANSIT | `#104281` | `#b7d3f6` |

(Validated ordinal ramps — monotone lightness, visible step gaps, surface-near end ≥2:1 — checked in both modes with the palette validator.)
| AWAITING_PAYMENT | neutral chip (`--dk-ink-2`) + clock icon; `--dk-warn` overlay when near/past due |
| DISPUTED | `--dk-crit` + icon + label (health, not stage) |
| COMPLETED | receded: `--dk-ink-3` + `--dk-good-text` check |
| CANCELLED | receded: `--dk-ink-3`, struck label |

Stage is never conveyed by hue alone: every stage chip carries its label; board marks pair ramp steps with shape/progress cues (§10).

### 4.5 Iconography

Single stroke-style icon set, 16px grid, 1.5px stroke, drawn in-repo as inline SVG symbols (no icon font, no CDN). Initial set: truck, crate (delivery), route (two points + path), coins (quote/payment), clock (deadline/expiry), wrench (maintenance), shield-check (audit ok), alert-triangle, alert-octagon (critical), robot (automation actor), person (staff actor), storefront (client actor), check, x, chevron, search, filter, sun/moon (theme), key (access), copy, external.

### 4.6 Component chrome defaults

Buttons: primary (accent fill, white label), secondary (hairline border, ink label), danger (crit fill, confirmation-gated), ghost (wash on hover only). Heights 32 (default) / 28 (dense rows). Focus: 2px accent ring outside the border, `:focus-visible` only. Disabled state: 40% alpha + `aria-disabled`; never remove from tab order when it can explain itself (tooltip says why).

---

## 5. Information architecture

### 5.1 Sitemap

```
Operator console (/, staff-key gated after landing)
├── Operations          — activity graph board + detail panel + mini-feed   [default]
├── Deliveries          — table of deliveries, filters, row → detail panel
├── Negotiation         — quote queue (QUOTED + MATCHED needing quotes), advice flow
├── Fleet               — truck table + register form + truck detail
├── Activity            — full journal + decision-trace feed, filterable
└── Access              — staff key entry/clear, theme, poll cadence, about

Client surface (/client)
├── Request             — create delivery form (geocoded origin/destination)
└── Track               — status timeline by delivery id; quote accept when QUOTE_READY
```

### 5.2 Navigation model

- Operator console: fixed left rail (56px icons-only collapsed / 208px expanded), top bar (product mark, global search by id, staleness indicator, theme toggle, access status). Content area center; **persistent right detail panel** (320–400px) on Operations; other screens open the same panel as an overlay drawer.
- Client surface: single column, top bar with product mark + Request/Track tabs. No rail.
- **Routing**: hash-based (`#/operations`, `#/deliveries/:id`, `#/negotiation`, `#/fleet`, `#/activity`, `#/access`; client: `#/request`, `#/track/:id`). Hash routing works under plain static serving with zero server rewrites; ids in the fragment never reach server logs.
- Entry points: console lands on Operations; deep links to any screen/entity via hash survive reload. Exit points: none destructive; closing mid-form warns only when a form is dirty.
- Shortcuts (§12.4) provide lateral movement (g o / g d / g n / g f / g a, `/` for search).

---

## 6. User flows

Every flow specifies: start state, intent, sequence, branches, validation, errors, completion. Error copy always renders the API's `message` verbatim beneath a human headline, plus the `X-Request-Id` in a copyable caption (per runbook: quote it in incident notes).

### 6.1 Operator: unlock console (access key)

- **Start**: any console route without a stored key. Screens render with data areas replaced by a lock empty-state.
- **Intent**: authenticate staff calls.
- **Sequence**: Access screen (auto-routed) → key field (password-type input, paste-friendly) → "Unlock" fires a probe call (`GET /v1/deliveries?limit=1`-shaped board read) → on 2xx, key goes to `sessionStorage` (never `localStorage`, never a cookie, never the URL) → return to prior route.
- **Branches/errors**: 401 → "Key not accepted" + retry (key cleared from field, not stored). Network failure → offline banner (§13), key kept in field.
- **Completion**: top-bar access chip shows "Staff ✓"; all staff calls attach the header. "Clear key" is one click; keys die with the tab (session scope).

### 6.2 Client: request a delivery

- **Start**: `/client#/request`, empty form.
- **Intent**: create a delivery and get a tracking id.
- **Sequence**: client id (UUID field with "generate" affordance for demo) → tonnage (decimal string) → origin (geocode entry, §3) → destination (geocode entry) → cargo class (select) → deadline (date-time picker, must be future) → review card (shows resolved coordinates + display names) → Submit. The UI generates an `Idempotency-Key` (UUID) when the form becomes dirty and holds it across retries.
- **Validation**: client-side mirrors of the API's rules (tonnage positive decimal, coordinates in range, deadline future, cargo class from enum); server remains authoritative.
- **Branches/errors**: 400 `INVALID_*` → field-anchored error using API message; 503 `TEMPORARILY_BUSY` → auto-retry banner honoring `Retry-After` (same idempotency key — safe); repeat submit after success returns 200 with the same delivery (idempotent — UI treats as success); 500 `REQUEST_NOT_RECORDED` → hard error state, "nothing was recorded, safe to retry".
- **Completion**: success panel with delivery id (copy button), deep link to `#/track/:id`.

### 6.3 Client: track and accept quote

- **Start**: `#/track/:id` (deep link or id pasted into Track tab).
- **Intent**: follow progress; accept a quote when offered.
- **Sequence**: poll `GET /v1/deliveries/{id}` (public) every 5s while visible → render **client-status timeline** (RECEIVED → … → DELIVERED) with current stage emphasized → when status is `QUOTE_READY`, an accept card appears → "Accept quote" → `POST /v1/quotes/{id}/accept`.
- **Branches/errors**: 404 → "No delivery with this id" empty state; accept 409 `ACCEPT_REFUSED` (e.g. quote expired between poll and click) → card flips to explanation with API reasons, timeline refreshes (honest state — the coordinator may have expired it); 503 → retry banner.
- **Completion**: timeline advances (SCHEDULED…); terminal states render a closing card (DELIVERED with confirmation; CANCELLED with plain statement).

### 6.4 Operator: register a truck

- **Start**: Fleet → "Register truck".
- **Sequence**: capacity (decimal) → position (geocode/manual, §3) → driver hours (duration picker rendering ISO-8601, e.g. PT10H) → maintenance due (optional date) → Submit → `POST /v1/trucks`.
- **Errors**: 400 `INVALID_*` field-anchored; 409 `REGISTRATION_REFUSED` renders desk reasons verbatim in a refusal callout (amber, not red — refusals are the system working); 401 → key flow (§6.1).
- **Completion**: success toast with truck id; fleet table refreshes; the new truck appears on the board at its position's facility.

### 6.5 Operator: assign a truck (manual, alongside the dispatcher)

- **Start**: Operations board (select a REQUESTED delivery) or Deliveries row → detail panel "Assign".
- **Intent**: match a truck to a delivery — knowing the dispatcher automation does the same every 15s.
- **Sequence**: panel lists candidate trucks (GARAGE, capacity ≥ tonnage — client-side filter over fleet read) with capacity/driver-hours/distance context → pick → Confirm → `POST /v1/deliveries/{id}/assign`.
- **Branches**: **the dispatcher may win the race** — 409 `ASSIGNMENT_REFUSED` (e.g. already matched) renders as an informational refusal: "The desk refused: <reasons>. The dispatcher may have assigned it first — the board reflects the current state." Feed shows the dispatcher's action with its trace. This is a designed moment: automation supervision, not error.
- **Completion**: delivery becomes MATCHED; board animates the truck's route emphasis; feed entry appears with actor.

### 6.6 Operator: quote with advice (HITL)

- **Start**: Negotiation queue (deliveries in MATCHED/QUOTED) → select → quote panel; or delivery detail → "Quote".
- **Sequence**: panel requests `GET /v1/deliveries/{id}/quote-advice` → advice card renders: suggested offer (figure), floor/ceiling (range bar with the suggestion marked), evidence line ("based on N similar past negotiations") → operator keeps or edits the offer (input pre-filled with suggestion) → Submit → `POST /v1/deliveries/{id}/quote`.
- **Advice honesty**: the advisor **clamps into policy, never rejects the human**; the card explains bounds ("Policy floor F — offers below are refused by the desk"). If advice fails: 503 `ADVICE_UNAVAILABLE` / 502 `ADVICE_REJECTED` → card degrades to "Advice unavailable — quote manually" (serious tone) with floor/ceiling omitted; the quote form still works.
- **Errors**: 409 `QUOTE_REFUSED` (e.g. below floor) → refusal callout with reasons; 409 `NOT_QUOTABLE` on advice → panel explains the delivery moved on; 503 `EMBEDDING_UNAVAILABLE` → advice degraded state.
- **Completion**: QUOTED with expiry countdown chip on board/queue; client sees QUOTE_READY.

### 6.7 Operator: progress signals

- **Start**: delivery detail panel (AGREED/IN_TRANSIT), signal group visible.
- **Sequence**: the panel offers the **next sensible signals** (from current state) as buttons — DEPARTED, ARRIVED_AT_PICKUP, LOADED (opens loadedTons field), ARRIVED_AT_DESTINATION, PAYMENT_CONFIRMED, OFFLOADED — full list behind "All signals" disclosure (the desk, not the UI, is the authority on legality).
- **Errors**: 409 `SIGNAL_REFUSED` / `NO_TRUCK_ASSIGNED` → refusal callout with reasons; the panel refreshes to true state.
- **Completion**: board vehicle advances along its route milestone (§10.4); feed entry.

### 6.8 Operator: watch disputes and pressure

- **Start**: Operations board — pressure cues surface automatically: quote-expiry countdowns (warn under threshold), payment-due countdowns, DISPUTED marks (critical).
- **Sequence**: select cued entity → panel shows the pressure detail (due time, time in state) → operator resolves out-of-band (payment confirmation arrives as a signal) or cancels per policy.
- **Completion**: cue clears when state moves; dispute stat tile decrements.

### 6.9 Operator: supervise automation

- **Start**: Activity screen or board annotation ("🤖 assigned T-… — trace").
- **Sequence**: feed merges journal actions and decision traces, filterable by actor (staff/dispatcher/coordinator/client/product), action, delivery; selecting an entry highlights the affected entities on the board (focus & context, §10.5) and opens the trace text (code style) in the panel.
- **Completion**: operator either agrees (does nothing) or intervenes with a manual action — same desk, same rules.

### 6.10 Stale data, offline, and audit posture

- Board reads are follower reads (may lag seconds): the top bar shows "as of HH:MM:SS" from the freshest poll; staleness beyond 3× poll cadence turns the indicator serious (§13).
- Full offline (fetch fails): banner "Reconnecting…" with backoff; last snapshot stays rendered with a stale wash; no writes offered while offline.
- Audit divergence is a backend severity-1 (runbook §7); if the activity feed ever carries a divergence entry, the console pins a critical banner linking the runbook procedure. The UI never mutates outside the desk.

---

## 7. Screen inventory

| Screen | Purpose | Primary action | Secondary actions | Data (endpoint) | Hierarchy |
|---|---|---|---|---|---|
| **Operations** | Situational awareness; entry to every flow | Select entity → act in panel | Filters (status/actor), zoom, theme | Board read (§14.2), fleet read (§14.1), activity tail (§14.3) | Graph 70% · panel 30% · stat row top · mini-feed bottom-left |
| **Deliveries** | Scan/filter all deliveries | Open detail panel | Filter by status; copy id | Board read (rich list) | Table full-width, drawer detail |
| **Negotiation** | Quote queue with advice | Quote (with advice) | Refresh advice; jump to board | Board read filtered + `GET …/quote-advice` | Queue list left · advice+form right |
| **Fleet** | Fleet health; registration | Register truck | Open truck detail; filters | Fleet read + `POST /v1/trucks` | Table + form drawer |
| **Activity** | Full audit trail of actions & reasoning | Filter/inspect | Jump-to-entity on board | Activity read (§14.3) | Single feed column + filter row |
| **Access** | Key, theme, cadence | Unlock/clear key | Theme, poll cadence | probe read | Narrow single column |
| **Client: Request** | Create delivery | Submit request | Generate client id | `POST /v1/deliveries` + Nominatim | Single column form, review card |
| **Client: Track** | Follow one delivery; accept quote | Accept quote (when offered) | Copy id, refresh | `GET /v1/deliveries/{id}`, `POST /v1/quotes/{id}/accept` | Timeline column |

Every screen defines all states in §13's matrix.

---

## 8. Layout system

- **Grid**: 12-column fluid, 8px gutters inside panels, 16px page margins (24 on ≥1440px). Max content width uncapped on Operations (board wants space); 1200px cap elsewhere.
- **App frame**: top bar 48px fixed; left rail 56/208px; right panel 320px (≥1280px: 400px). Content scrolls; frame does not.
- **Breakpoints**: ≥1280 desktop (rail expanded, panel persistent); 960–1279 compact (rail collapsed to icons, panel persistent on Operations, drawer elsewhere); 600–959 tablet (rail collapsed, panel becomes right drawer overlay everywhere, board stat row wraps); <600 (operator console shows a "best on a larger screen" notice but remains functional with the board full-bleed and all panels as drawers; client surface is designed mobile-first and fully capable).
- **Drawers**: right-side, 200ms slide (instant under reduced motion), scrim at 32% ink, focus-trapped, Esc closes (§13 focus rules).
- **Dialogs**: only for destructive confirmation (cancel delivery) and the key prompt when a 401 interrupts a flow. Everything else is panel/drawer — dialogs are a last resort.
- **Toasts**: bottom-right stack, max 3 visible, 6s auto-dismiss, pause on hover, never for errors that need action (those render in place).

---

## 9. Component library

Each component defines states (rest/hover/focus/active/disabled/loading/error), variants, keyboard behavior, and ARIA. Highlights (full specs live with the styleguide preview):

- **Buttons** (primary/secondary/danger/ghost; sizes 32/28) — Enter/Space activate; busy state swaps label for spinner + `aria-busy`.
- **Text field / textarea** — label above, helper/error caption below (`aria-describedby`); error sets `aria-invalid`; suffix slot for units (t, h).
- **Geocode field** (§3) — combobox pattern (`role="combobox"` + listbox), debounced 1 rps, results show name + coordinates + OSM attribution footer; arrow keys traverse; manual coordinate disclosure beneath.
- **Duration picker** — hours stepper rendering ISO-8601 ("PT10H") with plain-language echo ("10 hours").
- **Select** — native `<select>` styled; never a custom dropdown where native works.
- **Chips** (stage chip, actor chip, filter chip) — stage chips: ramp dot + label; actor chips: icon + name (🤖 dispatcher, ⏲ coordinator as *icons*, robot/clock glyphs from §4.5 — no emoji in product UI); filter chips removable with Backspace.
- **Badges** (counts) and **status marks** (health icon + label, §4.3 reserved palette).
- **Stat tile** — caption label + figure + optional delta in success/critical text; not a chart; no hover layer.
- **Table** — dense 40px rows, sticky header, sortable columns (aria-sort), row selection → panel; keyboard: arrows + Enter; column of ids in code style with copy affordance.
- **Timeline** (client track + delivery history) — vertical, stage-ramp dots, current stage pulsing (static ring under reduced motion), timestamps in caption.
- **Feed entry** — actor chip + action verb + entity link + relative time; expandable trace excerpt (code style, clamped to 6 lines with "expand").
- **Refusal callout** — amber panel, "The desk refused" headline, API reasons verbatim as a list, request id caption. Distinct from the red error callout (system failure).
- **Range bar** (quote advice) — floor/ceiling track with suggested-offer marker and the operator's current input ghosted; values always as text beside the bar (never bar-alone).
- **Toast, dialog, drawer, empty state, skeleton loader** — per §8; skeletons only for structure (never fake data), 800ms minimum before appearing to avoid flicker.
- **Board canvas** (§10) — the one bespoke component.

---

## 10. The activity board (primary visualization)

The centerpiece is an **abstract activity graph** — a topological rendering of current operations — not a geographic map. Geography answers "where on Earth"; operators ask "what is happening and where do I look". Normalized topology answers that at any scale, keeps clusters legible, and makes state — not distance — the dominant signal.

### 10.1 Answers it must give at a glance

- What is moving right now, and how far along is it?
- Which facilities are busy, idle, or in trouble?
- Which routes are active, at risk (expiring quote, payment due), or disputed?
- What did the automation just do?
- Where should attention go first?

### 10.2 Nodes — operational facilities

Nodes are **derived from operations**, not configured: distinct coordinate clusters referenced by live entities (delivery origins, destinations, truck positions/garages). Coordinates within a merge radius (default 250m) render as one facility; the reverse-geocoded name (§3.5) labels it, falling back to a short coordinate form.

| Node state | Derivation | Encoding |
|---|---|---|
| idle | no active entity references | small, `--dk-edge-rest` ring, label at 60% ink — recedes |
| active | referenced by en-route work | medium, accent ring |
| loading / offloading | a truck at node in LOADING/OFFLOADING | medium, accent fill + crate icon, subtle pulse (static under reduced motion) |
| queued | >1 truck waiting at node | badge with count |
| maintenance | truck in MAINTENANCE at node | wrench mark + label (warn palette) |
| attention | any critical cue references it | critical ring + alert icon |

Size steps: 8/12/16px radius (idle/active/busy). Never more than three sizes — size is a coarse signal.

### 10.3 Edges — operational connections

An edge exists where operations connect two facilities: each non-terminal delivery contributes origin→destination; each en-route truck contributes its current leg. Edges are **not roads**; rendered length is normalized by the layout (§10.6), not distance.

| Edge state | Derivation | Encoding |
|---|---|---|
| at rest | connection exists, nothing moving | 1px `--dk-edge-rest`, 40% opacity |
| active traversal | vehicle currently on it | 2px stage-ramp stroke + slow directional dash drift (static dashes under reduced motion) |
| at risk | expiring quote / payment near due on its delivery | 2px warn overlay glow + clock mark at midpoint |
| disputed | DISPUTED delivery | 2px crit, alert mark, label |
| completed (fading) | delivery reached terminal state | fades over 30s then drops to rest/disappears |

Stage identity on an edge uses the ordinal ramp (§4.4) **plus** the delivery's stage chip in its midpoint label on hover/selection — never hue alone.

### 10.4 Vehicles

Trucks render as markers riding their active edge, interpolating between milestone fractions; multiple vehicles share an edge with 6px lane offsets.

- **Progress model**: signals are ground truth. Milestones map the signal sequence onto the leg: garage→pickup leg (DEPARTED 0 → ARRIVED_AT_PICKUP 1), pickup dwell (LOADED), pickup→destination leg (0 → ARRIVED_AT_DESTINATION 1), destination dwell (OFFLOADED). Between milestones the marker eases toward the next known fraction — position is **presentational interpolation, clearly not telemetry**: the marker tooltip states the last confirmed signal and its time.
- **Encoding**: EN_ROUTE_LOADED = filled truck glyph (deep ramp step); EN_ROUTE_EMPTY = outlined truck glyph; LOADING/OFFLOADING = docked at node with crate mark; GARAGE = parked dot at facility; MAINTENANCE = wrench-marked at facility (warn).
- Marker size ≥12px, hit target ≥24px.

### 10.5 Focus and context

Selecting any entity (click/tap/keyboard from the mirror list):

- The selected entity and its operational neighborhood (its delivery's route, its truck, both facilities) render at full strength; everything else drops to 25% opacity.
- The right panel fills with the entity detail (§10.8).
- Related feed entries highlight in the mini-feed.
- Esc or surface-click clears focus. Focus state is in the hash (`#/operations?focus=delivery:ID`) — shareable.

### 10.6 Layout algorithm

Deterministic, stable, cheap: facilities lay out on a **radial-by-bearing arrangement** — angular position from the compass bearing between facility coordinates (preserving rough directional intuition without geographic scale), radial rings by activity (busy facilities pull inward). Positions update only when facilities appear/disappear; existing nodes never jump on data refresh (stability beats optimality). Zoom/pan with bounds; double-click/`0` resets. This is O(n) per layout pass and independent of delivery count.

### 10.7 Scalability

- Inactive elements fade (idle nodes 60%, rest edges 40%).
- \>40 facilities: idle facilities collapse into directional cluster nodes with counts; expand on zoom or click.
- Level-of-detail: labels hide below zoom thresholds (tooltips still work); dash animation suspends when >60 active edges (and under reduced motion).
- Filters (top of board): by delivery status, truck status, actor-of-last-action; filtering dims, never removes (context preserved).
- Poll cadence 3s (configurable in Access); rendering is snapshot-driven — one immutable state per poll, renderer diffs.

### 10.8 Detail panel (companion, persistent)

| Selection | Panel contents |
|---|---|
| **Vehicle** | id (code+copy), status chip, capacity, driver hours, assigned delivery link, last confirmed signal + time, maintenance due (warn when near), actions: none directly (trucks act via signals on their delivery) |
| **Facility** | name + coordinates (copy), current trucks (with states), expected arrivals (en-route toward it), recent feed entries touching it |
| **Route/Delivery** | id, stage chip + since-when, client id, tonnage, cargo class, deadline countdown, quote card (offer, expiry countdown, floor/ceiling if staff), assigned truck link, **action group for current stage** (assign §6.5 / quote §6.6 / signals §6.7 / cancel), history timeline, refusal/decision-trace excerpts |

### 10.9 AI annotation layer

The graph explains automation, grounded in real recorded data (never speculative):

- Dispatcher assignment → a brief accent flash on the chosen route + feed entry with robot chip; selecting it shows the decision trace ("why this truck") in the panel.
- Coordinator expiry/dispute actions → the corresponding warn/crit cue appears with a clock/alert mark; the annotation links the acting entry.
- Advice context (Negotiation) — the range bar and evidence count (§6.6).
- Future (⛔ cloud leg — designed placement only): a copilot drawer anchored bottom-right for conversational queries ("why is this delivery late?") answering from journal + traces via Bedrock; and natural-language filter construction in the board filter row. Neither ships until cloud work is authorized; the drawer slot and filter-row affordance are reserved in the layout so they arrive without redesign.

### 10.10 Renderer guidance

PixiJS 7 (vendored) with four layers: edges (Graphics), nodes (Graphics + BitmapText labels), vehicles (Sprites), annotation overlays. DOM overlays (HTML positioned above the surface) carry all text-heavy UI: tooltips, labels at low counts, badges — canvas text is minimized for a11y and crispness. The **DOM mirror** (§13) is a parallel hidden list of all board entities for assistive tech and keyboard traversal.

### 10.11 Correspondence to the formal model (privileged note)

The board is, deliberately, a live rendering of the operational structure model.md formalizes: facilities and their connections render the state loci; vehicle motion renders in-flight work between them; the feed renders the firing history; refusals surface guard outcomes. This correspondence is for privileged readers only — the repo and the UI copy speak exclusively in board/facility/route/vehicle vocabulary (§16), and no UI identifier or user-visible string references the model.

---

## 11. AI interfaces

Dakar's stance: **AI appears where decisions are made, explained by recorded evidence — never a detached chat box** (chat arrives later as the copilot drawer, §10.9, ⛔).

| Model | Surface | Grounding (exists today) |
|---|---|---|
| Inline assistance | Quote advice card in Negotiation (§6.6) | `GET /v1/deliveries/{id}/quote-advice`: suggestion, floor, ceiling, similar-episode count |
| Explainable decisions | Decision traces in feed + board annotations (§10.9) | decision-trace records (read via §14.3) |
| Automation supervision | Actor-labeled feed; race-refusals rendered as supervision moments (§6.5) | journal actor field |
| Embedded copilot | Reserved drawer slot (⛔ Bedrock cloud leg) | journal + traces + episodes as retrieval substrate |
| Conversational planning / AI-built filters | Reserved filter-row affordance (⛔) | board read model |

Every AI element must show: what it recommends, the bounds it operated under (floor/ceiling), the evidence quantity ("N similar negotiations"), and what the human's alternatives are (edit freely; the desk enforces policy regardless). Confidence is expressed by evidence count and bounds — no invented percentage scores.

---

## 12. Microinteractions

1. **Hover**: rows/cards get a 4% ink wash; board entities get a tooltip after 150ms and a 2px ring; edges thicken by 1px.
2. **Focus**: `:focus-visible` accent ring, 2px, offset 2px — identical everywhere.
3. **Selection**: single-selection model globally; selecting always populates the panel; Esc clears.
4. **Keyboard shortcuts**: `g o/d/n/f/a` navigate; `/` focuses search; `t` theme; `Esc` closes/clears; `?` opens the shortcut sheet; on the board: arrows traverse the mirror list, Enter selects, `0` resets zoom. No single-key destructive actions.
5. **Optimistic updates: none.** Every mutation waits for the desk; buttons enter busy state; refusals render as designed outcomes (§9 refusal callout). Rationale: refusals and races are semantically meaningful here (§6.5) — pretending success would misrepresent the system.
6. **Undo: none.** The journal is append-only; reversal is a new action (cancel), presented as such with confirmation.
7. **Drag/drop: none in v1** (assignment is deliberate, not gestural). Draggable pin in the geocode inset is the single drag surface.
8. **Copy affordances** on every id and request id.
9. **Command palette**: reserved (post-v1), shares the copilot slot's shortcut namespace (`Ctrl/Cmd-K`).

---

## 13. Accessibility and states

### 13.1 Accessibility (WCAG 2.2 AA targets)

- Semantic HTML everywhere outside the canvas; landmarks (`nav`, `main`, `aside`); labels tied to controls; live regions: feed (`aria-live="polite"`), banners (`assertive` for critical only).
- **Canvas strategy**: the board carries `role="img"` with a summary label, and a visually-hidden **DOM mirror** — a grouped list (facilities / vehicles / routes) of focusable entries mirroring board state; keyboard selection drives the same focus-and-context as pointer selection. The mirror is the accessible truth; the canvas is presentation.
- Contrast: text ≥4.5:1 (ink tokens are), UI boundaries ≥3:1; health colors pair icon+label always (warn on light is sub-3:1 by design — mitigated per the reserved-palette rule).
- Keyboard: everything reachable; focus visible; drawers trap focus and restore it on close; no keyboard traps.
- `prefers-reduced-motion`: §4.2 — interpolation, pulses, dash drift, slides all disabled.
- Touch targets ≥24px (board) / ≥32px (controls); client surface ≥44px primaries.
- CVD: stage = ramp + label + shape; health = icon + label; the palette set is validator-checked in both modes.

### 13.2 States matrix (binding for every screen/component)

| State | Treatment |
|---|---|
| initial | skeletons (structure only, 800ms delay) |
| loading (subsequent) | inline spinner in the acting control only — never blank a rendered surface |
| empty | icon + one-line explanation + the one action that fills it ("No trucks yet — Register truck") |
| partial | render what arrived; per-section inline retry for the missing piece |
| success | toast only when the result isn't visible in place |
| warning/attention | warn palette + icon + label (expiring, near-due, degraded advice) |
| error (action) | in-place callout: human headline, API `message` verbatim, request id, retry |
| error (system) | red callout; if fetch-level, escalate to offline banner |
| offline | sticky banner "Reconnecting… (retry in Ns)" with backoff 3→6→12→30s; snapshot stays with stale wash; writes disabled with explanation |
| permission denied (401) | key dialog (§6.1) preserving in-progress form state |
| stale data | "as of HH:MM:SS" chip; >3× cadence → serious tone; follower-read lag is expected and labeled, per-record fresh reads on selection |
| sync in progress | busy state on the initiating control; board updates only on next snapshot |

---

## 14. Data & API appendix

### 14.1 Existing surface (verified against code)

| Endpoint | Auth | UI use |
|---|---|---|
| `POST /v1/deliveries` (+`Idempotency-Key`) | public | client request (§6.2) |
| `GET /v1/deliveries/{id}` | public | client track poll |
| `POST /v1/quotes/{id}/accept` | public | client accept |
| `GET /v1/deliveries` | staff | superseded by the board read below for console use |
| `POST /v1/deliveries/{id}/assign · /quote · /signals` | staff | §6.5–6.7 |
| `GET /v1/deliveries/{id}/quote-advice` | staff | §6.6 |
| `POST /v1/trucks` | staff | §6.4 |

### 14.2 Required new read contracts (staff-only, follower reads, strict limits)

Implementation phase builds these three; the console is designed against them:

- **Fleet read** — `GET /v1/trucks?limit=` → `[{truckId, status, capacityTons, lat, lon, driverHours, assignedDeliveryId?, maintenanceDueAt?, updatedAt}]`
- **Board read** — `GET /v1/deliveries?view=board&limit=` (extends the existing list) → `[{deliveryId, status, clientId, tonnage, cargoClass, originLat, originLon, destinationLat, destinationLon, deadlineAt, agreedPrice?, quoteExpiresAt?, paymentDueAt?, assignedTruckId?, updatedAt}]`
- **Activity read** — `GET /v1/activity?limit=&actor=&deliveryId=` → merged, newest-first `[{at, kind: "action"|"trace", actor, action?, deliveryId?, summary, detail?}]` from the journal and decision traces.

All three: `AS OF SYSTEM TIME follower_read_timestamp()` (matching the existing list-read pattern), hard limit caps, staff header required. Poll cadences: board+fleet 3s, activity 5s, client track 5s; polls pause when `document.hidden`, abort superseded requests, and back off per §13.2.

### 14.3 Error code → UI treatment

| Code(s) | Treatment |
|---|---|
| `*_REFUSED` (409) | Refusal callout (amber), reasons verbatim, state refresh — designed outcome |
| `NOT_QUOTABLE` (409) | Panel explanation, queue refresh |
| `INVALID_*`, `MISSING_*` (400) | Field-anchored error |
| `DELIVERY_NOT_FOUND`, `NOT_FOUND` (404) | Empty state ("no delivery with this id") |
| `UNAUTHORIZED` (401) | Key flow, form state preserved |
| `TEMPORARILY_BUSY` (503 + Retry-After) | Auto-retry banner honoring the header; idempotent create retries reuse the key |
| `ADVICE_UNAVAILABLE` (503), `ADVICE_REJECTED` (502), `EMBEDDING_UNAVAILABLE` (503) | Advice card degraded state; manual path always open |
| `REQUEST_NOT_RECORDED` (500) | Hard error: "nothing recorded, safe to retry" |
| `INTERNAL_ERROR` (500), `SERVICE_UNAVAILABLE` (503) | System error callout with request id |

---

## 15. Wireframes

Operations (desktop ≥1280):

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ ◆ dakar   [search id…]                     as of 14:03:22 · ☾ · Staff ✓      │
├────┬────────────────────────────────────────────────────┬────────────────────┤
│ ⊞  │  In flight 12   Quoted 4   Disputes 1   Fleet 8/9  │  DELIVERY d-91c4…  │
│ ▤  │  [status ▾][truck ▾][actor ▾]              ⊕ ⊖ ⟲   │  ● IN_TRANSIT 2h   │
│ ⇄  │                                                    │  20t GENERAL       │
│ ▦  │        (activity board canvas)                     │  deadline in 22h   │
│ ≡  │     nodes · routes · vehicles · annotations        │  truck t-77a1 →    │
│ ⚙  │                                                    │  ── actions ──     │
│    │                                                    │  [ARRIVED_AT_DEST] │
│    ├────────────────────────────────────────────────────┤  [All signals ▾]   │
│    │ ▣ feed: 🤖 dispatcher assigned t-77a1 → d-91c4  9s │  ── history ──     │
│    │        ⏲ coordinator expired quote d-3f02      41s │  timeline…         │
└────┴────────────────────────────────────────────────────┴────────────────────┘
```

Negotiation advice card:

```
┌ Quote d-3f02 ───────────────────────────────┐
│ Suggested offer      184 000                │
│ floor ├────────●──────────────┤ ceiling     │
│ 150 000        184 000        230 000       │
│ Based on 17 similar past negotiations.      │
│ Offer [ 184000      ] [Quote]               │
│ ⓘ Offers below the floor are refused.       │
└─────────────────────────────────────────────┘
```

Client track (mobile):

```
┌ dakar ─ Track ┐   ● RECEIVED          ✓
│ d-91c4… [copy]│   ● PREPARING_QUOTE   ✓
│               │   ● QUOTE_READY  ← [Accept quote 184 000]
│               │   ○ SCHEDULED
│               │   ○ IN_TRANSIT …
└───────────────┘
```

High-fidelity mockups live as static pages in `dakar-ui/design/` (styleguide, console, negotiation, client) — they are the visual authority; this document is the behavioral authority.

---

## 16. Implementation guidance & repo vocabulary

- **Stack**: hand-written TypeScript compiled by `tsc` only (no bundler; browser-native ES modules — relative imports written with explicit `.js` extensions), PixiJS vendored, tokens as CSS custom properties, zero runtime npm dependencies (npm for dev typings/tooling only). Served by Quarkus same-origin. One immutable snapshot per poll drives rendering.
- **Secrets**: the staff key exists only in `sessionStorage` and the request header. Never in source, storage beyond the session, URLs, or logs.
- **Logging/telemetry**: UI console logging carries request ids, never the key, never client PII.
- **Repo-safe vocabulary** (Law 4; binding for all `dakar-ui` identifiers, class names, copy, and assets): use *board, facility, node, edge, route, vehicle, journey, stage, feed, refusal, access key, palette, ink, surface, accent*. The proscribed spec terms and their replacements are enumerated in model.md §12 / dakar_internals.md Law 4 — e.g. the canvas element is `board-canvas` (the scaffold's `network-canvas` is renamed at implementation). Platform-owned names (CSS properties, DOM/PIXI/Nominatim API fields) are exempt where unavoidable.
- **Draft-brief sections intentionally not applicable**: user onboarding and account authentication (staff key + public surface only — no user accounts exist), driver-facing UI (drivers are not system users; staff report signals), reporting/analytics dashboards (post-v1; the stat row and Activity screen cover current needs), historical route replay (post-v1; the journal makes it possible later).

## 17. Design rationale log

| Decision | Rationale |
|---|---|
| Abstract activity graph over geographic map | Operators ask "what needs me", not "where on Earth"; normalized topology stays legible at any density; state, not distance, is the signal. Geography remains available as coordinates + labels. |
| No optimistic UI | Refusals and automation races are meaningful outcomes of the desk's rules; rendering them honestly is the product, not a failure mode. |
| Ordinal ramp for stage + reserved health palette | 9 lifecycle states as distinct hues would be unreadable and unvalidatable; progression is ordinal by nature; health is orthogonal and reserved. |
| Nominatim | Free, self-hostable, no key material, honest confidence signal; region-agnostic. |
| Hash routing, no framework | Static serving with zero rewrites; the app is one console — a router library adds nothing but weight. |
| Vendored everything | Demo must work offline; no CDN drift; repo builds are reproducible. |
| DOM mirror for the canvas | Canvas is presentation; the mirror makes board state first-class for AT and keyboard, and doubles as the selection model's backbone. |
| Panel-first, dialog-last | Operators keep context; dialogs reserved for destructive confirmation and auth interrupts. |
