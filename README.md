# Trade8

Trade8 is the Sentin8 **professional-trading / broker-dealer** vertical:
multi-tenant order management, pre-trade risk, day P&L, market-data
distribution, and broker connectivity (FIX 4.2/4.4, OUCH 3.1) for
proprietary desks and introducing brokers.

Built as Rust services on the Sentin8 platform, with a Flutter trader
app (`etrader`) and Dash RFW admin pages.

## Services

| Service | Responsibility | Docs |
|---|---|---|
| `trade8-desk` | Order lifecycle, working-order state, catalogs, short-locate pool, session control | `services/desk/README.md` |
| `trade8-router` | FIX 4.2/4.4 + OUCH 3.1 broker sessions, destination routing, fees, fills | `services/router/README.md` |
| `trade8-risk` | Synchronous pre-trade authorization gate (limits, restrictions, trader state) | `services/risk/README.md` |
| `trade8-feed` | Vendor feed adapters → L1/L2/imbalance prints → entitlement-gated SSE | `services/feed/README.md` |
| `trade8-bars` | OHLCV bars (1s–1D), EOD roll-up, history API | `services/bars/README.md` |
| `trade8-pnl` | Event-sourced positions, day P&L, overnight carry, office roll-ups | `services/pnl/README.md` |

Each service directory carries its `README.md` (overview),
`DESIGN.md` (detailed design), and `BUSINESS.md` (business requirements).

## Workspace layout

| Path | Contents |
|---|---|
| `services/` | The six deployable services (binaries + per-service SQL + tests) |
| `crates/` | Shared libraries: `contracts` (domain types), `fixed` (money math), `fix`/`ouch` (wire codecs), `feed-adapters` (vendor parsers), `trade8-client` (bus/service/cron plumbing) |
| `strategies/` | Sentin8:Func WASM strategy modules (flatten, basket, composite alerts) |
| `sdk/rust/trade8` | The `{workspace}::{leaf}` namespace facade — consumers import as `trade8::contracts::…` etc. |
| `apps/etrader` | The Flutter trader app (BFF-only transport) |
| `sql/` | Seed, probe, RLS-check, parity-audit, disposition-audit SQL |
| `docs/` | Operational runbooks (DR replay) |
| `packaging/` | Debian packaging |
| `e2e-ts/` | Playwright UI tests (browser-only, via the BFF proxy) |

## Documentation map

Start here, then drill in:

| Document | Role |
|---|---|
| `README.md` (this file) | What the vertical is; workspace layout; how to run it |
| `BUSINESS.md` | The vertical's business requirements — capabilities, rules, personas, KPIs, acceptance criteria |
| `DESIGN.md` | Vertical detailed design — system/container views, data model, event vocabulary, cross-cutting invariants, testing |
| `services/*/README.md` + `DESIGN.md` + `BUSINESS.md` | Per-service overview, wire-level detail, business functionality |
| `crates/*/README.md` | Shared-library contracts (types, money math, codecs, adapters, client plumbing) |
| `strategies/README.md` + `strategies/*/README.md` | The Sentin8:Func strategy modules (flatten, basket, alerts) |
| `apps/etrader/README.md` + `BUSINESS.md` | The trader app — structure and business view |
| `packaging/README.md` | Debian packaging + post-install platform flow |
| `docs/dr-replay-runbook.md` | Disaster-recovery / day-reconstruction drill |

`BACKLOG.md` tracks open stories at the epics level.

## App

- **`etrader`** — Flutter trader app (web/mobile/desktop) via the Sentin8
  App Store. All traffic through Sentin8:BFF (`/proxy/trade8/...`,
  SSE `/subscribe`, WS `/ws`).

## Design

For how the vertical is built: the top-level `DESIGN.md` maps the whole
system (context, containers, data model, events, invariants); each
`services/*/DESIGN.md` carries that service's wire-level detail. For the
business view: the top-level `BUSINESS.md`, then the per-unit
`BUSINESS.md` documents.

## Build & run

```bash
make help          # target index
make build-rust    # cargo build --workspace --release
make check         # quick compile check
make clippy        # lint (warn-free required)
make test-rust     # Rust tests (DB suites skip when offline)
make seed-demo     # seed the demo catalog into db trade8
```

## Status

Implemented vertical: 16-crate Cargo workspace (6 service binaries, 5
library crates, 3 WASM strategy modules + the namespace facade), live
PostgreSQL schemas in db `trade8` (schema-per-service, RLS-enforced),
Shield-policy authorization, outbox-first event production, and the etrader
Flutter app. See `BACKLOG.md` for open stories.

---

**Design attribution:** trade8 and its underlying concepts were designed by Hamid Jahanzad <Hamid@Jahanzad.com>.
