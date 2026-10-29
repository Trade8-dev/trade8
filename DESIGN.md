# Trade8 - Vertical Detailed Design Document

> Secure Post-Quantum-Safe Trading Platform That Heals and Scales Itself.

> Audience: maintainers, reviewers, and architects working anywhere in the
> trade8 workspace. This document is the **vertical map**: what exists, how
> the pieces fit, and the invariants that hold across services. Per-unit
> depth: `services/*/DESIGN.md`, `crates/*/README.md`,
> `strategies/*/README.md`, `apps/etrader/{README.md,BUSINESS.md}`;
> business view in the top-level `BUSINESS.md`.

## 1. Scope & Context

Trade8 is the Sentin8 **broker-dealer / professional-trading vertical**. It
owns, as one deployed unit:

- **Order management** - the full order lifecycle (submit → authorize →
  accept → route → fill → terminal), working order state, overnight parking,
  the short-sale locate pool, session control, and the reference catalogs.
- **Pre-trade risk** - the synchronous authorization gate and the
  trader-state/limit/restriction administration behind it.
- **Broker connectivity** - FIX 4.2/4.4 + OUCH 3.1 client sessions,
  destination routing, message templates, fee charging, fills.
- **Market data** - vendor-feed ingestion, normalization, NBBO-style
  consolidation, entitlement-gated SSE streaming, price alerts.
- **Bar aggregation** - OHLCV bars (1s-1D), EOD roll-up, history API.
- **Day P&L** - event-sourced positions, realized/unrealized, LFT,
  summaries, overnight carry.
- **The trader app** - `etrader` (Flutter), a pure BFF client.

**Deliberately out of scope** (owned elsewhere): identity/sign-in and the
authorization *decision* (Sentin8:Auth + Shield); token issuance to clients
(Sentin8:BFF); scheduling (Sentin8:Cron); user-authored strategy execution
(Sentin8:Func); audit ingest (Sentin8:Sys); object storage (Sentin8:Storage);
venue-side matching/clearing (exchange8, a sibling vertical).

### System context (C4 level 1)

```mermaid
graph TD
    TRADER[Trader / manager] --> APP[etrader trader app]
    ADMIN[Catalog / risk / ops admin] --> DASH[sentin8:dash RFW pages]
    subgraph trade8[trade8 vertical - this workspace]
        APP -- "BFF-only transport" --> BFF[sentin8:bff]
        BFF -- "/proxy/trade8/v1/*, SSE, WS" --> SERVICES[the six services]
    end
    SERVICES --> PLATFORM[sentin8 platform<br/>auth · bus · cron · func · sys · db · storage]
    ROUTER2[Router] -- "FIX 4.2/4.4, OUCH 3.1" --> BROKERS[External brokers]
    ROUTER2 -- "adapter callback" --> X8GW[exchange8 Gateway<br/>optional sibling vertical]
    VENDOR[Market-data vendors] -- "ITCH/IDC/WS" --> FEED2[Feed]
```

### Container view (C4 level 2)

```mermaid
graph TD
    subgraph clients[Client tier]
        APP[etrader] --> BFF
        DASH --> PROXY[/proxy/trade8 admin views/]
    end
    subgraph svcs[trade8 services - Rust binaries on sentin8 shell]
        DESK[trade8-desk<br/>orders · catalogs · borrow · session]
        RISK[trade8-risk<br/>authorize gate · trader state]
        ROUTER[trade8-router<br/>FIX/OUCH · routes · fees · fills]
        FEED[trade8-feed<br/>adapters · consolidation · SSE · alerts]
        BARS[trade8-bars<br/>OHLCV · EOD · history]
        PNL[trade8-pnl<br/>positions · P&L · carry]
    end
    subgraph sdk[shared code]
        FACADE[sdk/rust/trade8<br/>namespace facade trade8::*]
        CONTRACTS[contracts: types + events]
        FIXED[fixed: ×10⁴ money]
        CODECS[fix / ouch codecs]
        ADAPT[feed-adapters]
        CLIENT[trade8-client: bus/caller/cron]
    end
    subgraph wasms[Sentin8:Func strategy modules]
        FLAT[flexible-flat on_flatten]
        BASKET[basket-runner on_basket]
        PALERTS[price-alerts on_quote]
    end
    subgraph data[db trade8 - one database, schema per service]
        SCHEMA_DESK[(desk schema, 30 tables)]
        SCHEMA_ROUTER[(router)]
        SCHEMA_RISK[(risk)]
        SCHEMA_FEED[(feed)]
        SCHEMA_BARS[(bars)]
        SCHEMA_PNL[(pnl)]
    end
    BFF -- "HTTPS /proxy/trade8/v1/*" --> DESK
    BFF -- proxy reads --> RISK
    BFF -- "quotes/depth" --> FEED
    BFF -- "bars" --> BARS
    BFF -- "pnl" --> PNL
    DESK -- "POST /v1/authorize (M2M mTLS)" --> RISK
    DESK -- "BBO snapshot call" --> FEED
    svcs -- "outbox rows → shell relay → POST /v1/produce" --> BUS[sentin8:bus]
    svcs -- "SSE consumer groups" --> BUS
    FACADE --> CONTRACTS & FIXED & CODECS & ADAPT & CLIENT
    svcs --> FACADE
    DESK -- SQL (role trade8_desk_service) --> SCHEMA_DESK
    ROUTER -- SQL (role trade8_router_service) --> SCHEMA_ROUTER
    RISK -- SQL --> SCHEMA_RISK
    FEED -- SQL --> SCHEMA_FEED
    BARS -- SQL --> SCHEMA_BARS
    PNL -- SQL --> SCHEMA_PNL
```

## 2. Runtime Architecture

### The vertical's request families

```mermaid
graph LR
    subgraph hot[Order hot path - sub-second]
        A[Trader submit<br/>WS or REST] --> B[Desk: structural checks<br/>Pending order]
        B -- "M2M POST /v1/authorize" --> C[Risk: snapshot verdict<br/>µs-class math]
        C -- allow --> D[Desk: one tx = order +<br/>order_event + outbox]
        D -- "bus: order:accepted" --> E[Router: route → lease →<br/>journal → order:routed]
        E --> F[Broker]
    end
    subgraph fast[Market-data path - streaming]
        G[Vendors] --> H[Feed adapters] --> I[Normalizer + consolidation]
        I -- "SSE frames" --> APP
        I -- "outbox events" --> BUS2[bus]
    end
    subgraph derived[Derived-state path - event-driven]
        E -- "fill:executed" --> J1[Pnl: positions + summaries]
        I -- "trade:printed" --> J1
        I -- "trade:printed" --> J2[Bars: bucket FSMs]
        D -- "lifecycle events" --> J3[Risk: reservation accounting]
    end
```

### Boot and runtime wiring (shared shell)

Every service boots on `sentin8::shell::ServiceCore`:

```mermaid
sequenceDiagram
    autonumber
    participant CAP as Sentin8:Captain
    participant SH as Shell (sentin8::shell::serve)
    participant SVC as trade8 service
    participant DB as db trade8 (own schema)
    participant BUS as sentin8:bus
    CAP->>SH: start binary (catalog priority, DSN env = TRADE8_{LEAF}_DATABASE_URL)
    SH->>SH: fail-fast: DSN must name db trade8
    SH->>SVC: with_state(core): M2M DCR, Shield boot load, pool cap
    SVC->>DB: bootstrap (catalog/snapshot/links) on boot-env context
    SH->>DB: boot outbox+audit relay (8b): NOTIFY + 5s poll → POST /v1/produce
    SVC->>BUS: spawn bus consumer groups (SSE, ack/nack)
    SVC->>SH: with_router(extend_router): public + admin + policy routes
    Note over SVC,DB: health at /v1/health, registration via --print-registration
```

### Request lifecycle - the authoritative order-submit family

```mermaid
sequenceDiagram
    autonumber
    participant T as Trader (BFF proxy)
    participant D as Trade8:Desk
    participant S as Shield
    participant R as Trade8:Risk
    participant DB as desk schema
    T->>D: POST /v1/orders (JWT bearer → BFF)
    D->>S: verify + policy require trade8:desk:order:write (live roles)
    D->>D: structural validation → Pending
    D->>R: authorize (full order context, Intent::Open)
    R-->>D: verdict + itemized trace
    alt deny / Risk unreachable
        D-->>T: 422/503 with reject code + trace (fail-closed)
        D->>DB: journal the rejection (order_event + outbox)
    else allow
        D->>D: accept (reserve locate if short)
        D->>DB: begin(tx): order row + transition + outbox(submitted/accepted)
        D-->>T: 200 {order_id, state, remaining_buying_power}
    end
```

### Request lifecycle - the market-data stream family

```mermaid
sequenceDiagram
    autonumber
    participant V as Vendor link
    participant F as Trade8:Feed
    participant S as Shield
    participant A as App subscriber
    V->>F: RawQuotePacket (adapter, bounded mpsc)
    F->>F: normalize + consolidate (NBBO-style BBO)
    F->>A: GET /v1/stream/{symbol}: snapshot frame first
    F->>S: subscribe-time gate (quote:read / depth:read)
    loop live
        F-->>A: frame {seq, event, data} (per-symbol filtered)
        F->>F: outbox event per quote/print/depth/imbalance
    end
    A->>A: seq gap → resync banner + snapshot refetch
```

## 3. Data Model

One PostgreSQL database `trade8`, **schema-per-service**; each schema is
private to its service (RLS `ENABLE`+`FORCE`, standard env-block policy on
every table, per-service OAuth role, shell-generated `{schema}.outbox` +
`{schema}.audit_journal`).

```mermaid
erDiagram
    desk_symbol ||--o{ desk_order : "instrument"
    desk_market ||--o{ desk_symbol : "lists"
    desk_destination ||--o{ desk_order : "routes to"
    desk_office ||--o{ desk_trader_profile : "home office"
    desk_order ||--o{ desk_order_event : "append-only journal"
    desk_order |o--o| desk_overnight_order : "parked as"
    desk_symbol ||--|| desk_short_pool : "locate"
    desk_symbol ||--o| desk_borrow_list : "ETB/HTB"
    desk_destination |o--o{ desk_msg_template : "typed templates"
    router_session ||--o{ router_message : "wire journal"
    router_order_transaction ||--o{ router_fill_fix_tags : "broker tags"
    risk_limit }o--|| risk_trader_state : "scoped by"
    pnl_position_snapshot ||--|| pnl_trader_summary : "folds into"
    bars_bar_1s ||--o{ bars_bar_1d : "rolled up"
    feed_feed_link ||--o{ feed_alert : "traffic fires rules"
    desk_order_event ||--o{ pnl_position_snapshot : "fills derive"
```

| Schema | Owned tables (selection) | Read grants declared by |
|---|---|---|
| `desk` (30) | `"order"`, `order_event`, `overnight_order`, `keystroke_log`, `short_pool`, `short_pool_office_block`, `borrow_list`, `short_exempt`, catalogs (`market`, `symbol`, `future`, `destination`, `gateway`, `msg_template`, `fee_catalog`, `office`, `trader_profile`, …), `sequence` | - (reads nothing) |
| `router` (4) | `message` (wire journal), `session` (seq/state), `order_transaction` (fills - sole writer), `fill_fix_tags` | `desk.{destination,gateway,market_gateway_destination,office_broker,msg_template}` |
| `risk` (8) | `limit`, `trader_state`, `execution_restriction`, `max_orders_restriction`, `allowed_symbol`, `level`, `action`, `level_action` | `desk.short_exempt` |
| `feed` (4) | `feed_link` (secret-ref credentials), `quote_source`, `alert_rule`, `alert` | - |
| `bars` (7) | `bar_1s…bar_1h` (LIKE base), `bar_1d` (+vwap/official), `eod_report` | - |
| `pnl` (5) | `opening_balance`, `position_snapshot`, `symbol_resume`, `trader_summary`, `office_summary` | `desk.currency` |

Cross-service read access is **declared, not blanket** (register
`cross_schema_grants`, applied by `vertical enroll`). The `sentin8` superuser
is for DDL only; no role carries BYPASSRLS.

### Cross-vertical state machines

```mermaid
stateDiagram-v2
    [*] --> Pending : submit (structural OK)
    Pending --> Accepted : Risk allow
    Pending --> Rejected : Risk deny / dup
    Accepted --> Routed : Router lease + journal
    Routed --> PartiallyFilled : broker fill
    PartiallyFilled --> Filled
    PartiallyFilled --> Replaced : cancel-replace (new chain id)
    Accepted --> Cancelled
    Routed --> Cancelled
    PartiallyFilled --> Cancelled
    Filled --> [*]
    Cancelled --> [*]
    Rejected --> [*]
    note right of Replaced : new Pending order (chain_root)
```

```mermaid
stateDiagram-v2
    direction LR
    [*] --> preopen
    preopen --> open
    open --> close_pending
    close_pending --> closed
    closed --> preopen : next session
    note right of closed : day-TIF remnants cancelled; overnight parked
    note left of preopen : overnight orders restored to Pending
```

```mermaid
stateDiagram-v2
    direction LR
    [*] --> down
    down --> connecting
    connecting --> logged_in
    logged_in --> down : watchdog (stale heartbeat) / logout
    note right of logged_in : outgoing_seq leased atomically per order
```

## 4. API Surface

Public API is BFF-proxied (`/proxy/trade8/v1/...`); every route is
Shield-gated (`sentin8-policy`, live-resolved roles; the JWT scope is a
pre-filter, never the decision). Per-service inventories live in
`services/*/DESIGN.md` §4; the vertical surface:

| Service | Public routes (method×path) | Admin (RFW `GET /v1/admin/ui/{page}`) |
|---|---|---|
| Desk | `POST/GET /v1/orders`, `GET/DELETE /v1/orders/{id}`, `POST /v1/orders/{id}/replace`, `POST /v1/orders/batch`, `POST /v1/positions/flatten`, `GET/POST /v1/session`, `GET /v1/symbols`, `GET/POST /v1/borrow`, `GET /ws` | overview, orders, symbols, markets, destinations, offices, borrow, session |
| Risk | `POST /v1/authorize` (M2M-only), `GET/POST /v1/limits`, `GET/POST /v1/trader-state` | limits, traders, restrictions, levels, exposure |
| Router | `GET /v1/sessions`, `POST /v1/sessions/{id}/reset`, `GET /v1/messages`, `POST /v1/journal/purge` (Cron), `POST /v1/internal/executions/{dest}` (M2M) | sessions, journal, fees |
| Feed | `GET /v1/quotes/{symbol}`, `GET /v1/depth/{symbol}`, `GET/POST /v1/alerts`, `GET /v1/feeds`, `GET /v1/stream/{symbol}` (SSE) | feeds, sources, alerts (+ack) |
| Bars | `GET /v1/bars`, `GET /v1/bars/latest`, `GET/POST /v1/eod` | coverage, eod |
| Pnl | `GET /v1/pnl/summary`, `/v1/pnl/positions`, `/v1/pnl/office`, `/v1/pnl/traders`, `POST /v1/pnl/session-close` (Cron) | traders, overnight |

Every service also mounts the shared Shield policy-admin block
(`/v1/admin/policy-sets`, `/v1/admin/relations`, `/v1/admin/attributes`),
`GET /v1/health`, and RFW POST-alias data endpoints.

### Event vocabulary (`trade8::contracts::events`)

Naming law: `vertical:service:resource:operation` - colon-separated, no
dots (test-enforced in the contracts crate). Delivery is outbox-first
(`{leaf}.outbox` row in the domain tx → shell relay → Bus `POST /v1/produce`
→ SSE consumer groups → ack/nack → DLQ).

```mermaid
sequenceDiagram
    autonumber
    participant P as Producer service
    participant O as {leaf}.outbox (own schema)
    participant REL as Shell relay
    participant B as sentin8:bus
    participant C as Consumer group
    participant DLQ as Bus DLQ
    P->>O: insert_outbox_row (same tx as the domain write)
    REL->>O: NOTIFY {leaf}_outbox_changed + 5s poll
    REL->>B: POST /v1/produce (M2M produce scope, idempotency_key=row id)
    B->>C: SSE consumer group (e.g. trade8:pnl:router)
    alt handler ok
        C-->>B: ACK
    else handler fails
        C-->>B: Nack → retry (backoff) → DLQ (manual replay)
    end
```

| Family | Emitted (as implemented) |
|---|---|
| desk | `order:submitted/accepted/rejected/cancelled/replaced/routed/overnight/overnight:restored/flatten`, `session:state`, `borrow:updated`, `keystroke:logged` |
| router | `fill:executed/cancelled/rejected`, `session:status` |
| risk | `limit:updated`, `trader:state` |
| feed | `quote:updated`, `depth:updated`, `imbalance:updated`, `trade:printed`, `alert:fired`, `feed:status` |
| bars | `bar:closed`, `eod:completed` |
| pnl | `position:updated`, `trader:summary`, `office:summary`, `overnight:mismatch` |

Declared-but-unemitted constants (prune-or-wire open items, tracked per
service): `desk:catalog:updated`, `desk:borrow:reserved/released/adjusted`,
`risk:restriction:updated`; the `vertical-registry.json` basket trigger
references `trade8:desk:order:batch-requested`, which no producer emits yet.

Consumer wiring (SSE groups over `trade8::client::run_consumer`):

| Consumer | Families | Interested in |
|---|---|---|
| Risk | desk, router, feed, pnl | reservation accounting, position counters, halt/uptick mirror, loss ladder |
| Router | desk | `order:accepted` → route/journal/ack |
| Bars | feed | `trade:printed` → bucket FSMs |
| Pnl | router, desk, feed | fills → positions; session close → carry; prints → marks |

### Declarative triggers (`vertical-registry.json`)

| Trigger | Target |
|---|---|
| `trade8:desk:order:flatten` → | Func `trade8-flexible-flat` `on_flatten` (3 retries exp.) |
| `trade8:desk:order:batch-requested` → | Func `trade8-basket-runner` `on_basket` (3 retries exp.) |
| `trade8:feed:quote:updated` → | Func `trade8-price-alerts` `on_quote` (5 retries linear) |

## 5. Cross-Cutting Concerns

```mermaid
graph LR
    REQ[request] --> V[1 verify token<br/>issuer/sig/expiry/aud/type]
    V --> T[2 resolve tenant<br/>env_id - reject if unresolvable]
    T --> FG[3 feature gate]
    FG --> AUTH[4+5 Shield authorize<br/>RBAC+ABAC+ReBAC, live roles]
    AUTH --> RLS[6 set RLS context<br/>transaction-local, same conn/tx]
    RLS --> EX[7 execute]
    EX --> AUD[8 audit journal<br/>local row → Sys ingest, retried]
```

- **Authorization** - every route evaluates the embedded Shield engine at
  step 5; policy sets are seeded per service (`scope-mirror` over the live
  effective scope set), hot-reloaded; step 5 is the only gate.
- **Tenant isolation** - `set_tenant_context` + `begin()` on the same
  connection; explicit `WHERE env_id = $N` defense-in-depth; RLS is the
  backstop (probe-role matrix tests prove 0-rows without context).
- **Idempotency** - client order ids (Desk), transaction ids (fills),
  atomic sequence leases (Router), merge upserts (Bars), fill dedupe
  (Pnl), bounded-queue backpressure everywhere fan-out exists.
- **Secrets** - provider credentials live as *references*
  (`credentials_ref`) in `desk.destination` / `feed.feed_link`; no
  plaintext anywhere in the vertical's tables.
- **TLS** - post-quantum platform TLS (`sentin8-tls`: X25519MLKEM768,
  ML-DSA-65); service-to-service calls present the workload certificate
  (auth DESIGN §4.3f - no M2M token); the BFF holds
  user tokens server-side (the app never sees one).
- **Audit** - desk `keystroke_log` + per-schema `audit_journal` rows in the
  domain tx; Sys ingest is retried until pushed.
- **Drain** - service restarts go through Sentin8:Captain
  (`make upgrade-instance`); sessions/engines are DB-backed so the swap
  is seamless.

## 6. Failure Modes & Recovery

```mermaid
flowchart TD
    subgraph orderfail[Order path]
        A[Risk unreachable] -- Desk fail-closed --> F1[422/503, nothing persisted]
        B[persist fails post-accept] --> F2[engine reverts accepted state<br/>cancel_with_actor]
    end
    subgraph busfail[Event path]
        C[relay outage] --> F3[outbox rows durable,<br/>attempts++, 5s poll resumes]
        D[consumer fails] --> F4[Nack → retry → DLQ replay]
    end
    subgraph sessionfail[Crash]
        E[Desk crash] --> F5[order_event replay rebuilds]
        G[Router crash] --> F6[session counters from DB;<br/>resend window from journal]
        H[Pnl crash] --> F7[opening_balance carry-in +<br/>bus redelivery]
        I[Bars crash] --> F8[interval tables + print re-accumulation]
    end
```

- The full day-reconstruction procedure is scripted: `docs/dr-replay-runbook.md`
  (`make drill-replay`) - quarterly drill, byte-equal convergence required.
- Every durable write is either (a) idempotent-under-redelivery or
  (b) transactional with the journal; no dual-written financial state
  exists anywhere in the vertical.

## 7. Concurrency & Performance Model

  (wasm32 strategies exempt); master-design cross-cutting decision.
- **Hot-path budgets** (master §11; harness-pinned per service): order
  acceptance P50 < 5 ms / P99 < 25 ms in-envelope; quote-to-SSE < 1 ms;
  bar roll-up and fill-to-summary in their CI-harness budgets.
- **State shapes** - Desk: mutex-guarded working map + snapshot catalog;
  Risk: single `arc-swap` snapshot generation per authorize + mutex journal
  maps; Feed: `RwLock` maps + broadcast SSE channel; Bars: `RwLock` bucket
  FSMs; Pnl: `RwLock` books/summaries + dedupe set; Router: consumer loop +
  atomic SQL leases. All locks guard pure in-memory operations only (no
  I/O under lock).
- **Pools** - `sentin8_config::pg_pool::connect` caps every pool
  (`SENTIN8_PG_POOL_MAX`); store calls are tx-scoped (≤1 connection per
  request).
- **Backpressure** - bounded mpsc (adapters), bounded broadcast + per-topic
  seq (SSE), bounded queues (fan-out), SQL-side lease/downgrade instead of
  in-process scheduling.

## 8. Invariants, Insecure-Defaults Rejected, Unsafe Rationale

1. **One writer per financial record** - Desk writes orders/ journal;
   Router is the sole writer of fills (relocated from Desk deliberately);
   Pnl writes only derived positions/summaries.
2. **No un-authorized acceptance** - Desk accepts only on a Risk allow
   verdict, fail-closed when Risk is unreachable.
3. **No cross-service table access without declaration** - the three read
   grants in §3 are the complete list; everything else is API/events.
4. **Money never floats** - `trade8::fixed` raw integers + explicit scale
   end-to-end (×10⁴ money, instrument tick scales, ×10⁸ crypto).
5. **Event names are the contract** - colon-separated four-segment names,
   verified by contract tests; consumers dedupe on idempotency keys.
6. **Secrets by reference**; no plaintext credentials, tokens, or keys in
   any trade8 table.
7. **No `unsafe` blocks** in the workspace; codecs verify checksums/frame
   lengths and return typed errors - ingest paths never panic on bad input.
8. **Journal-first** - every derived surface (positions, fills, bars) is
   reconstructable from the event log + opening balances.

## 9. Testing Strategy

```mermaid
graph TD
    U[Unit: engine pipelines, codec golden bytes, math scale cases] --> POL[tests/policy.rs ×6<br/>Shield engine-vs-seed suites]
    U --> SQL[tests/sql_matrix.rs ×6<br/>service role + probe role, RLS matrix,<br/>transactional writes, skip-offline]
    POL --> E2[e2e-ts Playwright<br/>UI only through the BFF]
    SQL --> E2
    U --> LAT[Latency harnesses<br/>per-service budgets]
```

| Layer | Owns | Count (approx) |
|---|---|---|
| Unit (in-crate) | state machines, codecs, math, event law | 180+ |
| `tests/policy.rs` | Shield wiring per service (deny/shadow/M2M parity/relations) | 66 |
| `tests/sql_matrix.rs` | RLS matrix, transactional persistence, per-schema behaviors | ~45 (skip-offline) |
| Latency harnesses | budget guards per service | 6 |
| `e2e-ts` | app shell + declarative triggers (Function exec) via browser+BFF only | smoke + 2 trigger specs |

Deliberately **not** tested in this workspace: platform transport (sentin8),
broker live-wire behavior (sim/parity adapters cover), venue matching
(exchange8).

## 9.1 Fleet-run scope (group operations,)

The group Makefile's `make start` boots ONLY the platform base
(`sentin8:acme, sentin8:auth, sentin8:proxy, sentin8:bus, sentin8:sys,
sentin8:bff, sentin8:web`) plus this group's services - the
`SENTIN8_SERVICES` boot pin when captain is down, the idempotent
`captain start <types>` API form when it is up. `make stop` stops only
this group's instance rows (the captain and platform stay up). The
full rule (service / group / tree-root scopes and the platform-scoped
dev reset): `design/sentin8-repository-split-design.md` §4.5.2.

## 10. Open Questions (vertical-level)

- **Dead event constants** - prune or emit `catalog:updated`,
  `borrow:reserved/*`, `restriction:updated`; and confirm the async basket
  trigger (`order:batch-requested`) vs the synchronous batch path.
- **Router socket transport** - dial-out for native FIX/OUCH endpoints
  (state machines, codecs, resend windows are ready; the transport wiring
  remains).
- **Session-open anchor for Bars** - per-day Cron reset vs calendar read.
- **Cold-tier replay loop** (Bars archive → rehydrate; Pnl journal-first
  replay beyond bus retention).
- **Deny-by-default trader states** - env opt-in instead of
  active-by-default.
- **Registration/route parity** - a few `registration.rs` data-endpoint
  declarations have no route (or vice versa): implement or prune.
- **Catalog replication across envs** - owner-env authoring vs per-env
  copies reconciled by events.

---

**Architecture & Design:** trade8 and its underlying concepts were designed by [Hamid Jahanzad](https://www.linkedin.com/in/hamidjahanzad/).
