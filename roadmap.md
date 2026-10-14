# Crypto Roguelike — Build Roadmap

Oct 13, 2026 · @Kevin · updated Oct 14, 2026 to match `docs/system-design.md`

## Overview

A single-player, browser-based roguelike where you trade fake crypto in a market of simulated agents. The goal is a live, playable portfolio piece for fintech roles, with a Rust core that shows matching, ledgers, risk and determinism.

**The run.** One run is a trading year of 4 quarters. Each quarter has a rising profit quota (roughly +20%, +50%, 2x, 4x). Between quarters, the player drafts 1 of 3 upgrades. Quota numbers and balance are deliberately undecided; they get tuned once the market exists.

**Lose conditions.**

- Miss the quarter's quota: fired.
- Breach the max drawdown from peak equity: funding pulled instantly.
- Get liquidated on a leveraged position (once leverage is unlocked).

**Win condition.** Survive all 4 quarters. Score = final equity adjusted for volatility (a simplified Sharpe ratio). Endless mode continues with rising quotas.

**Daily run.** Same seed for everyone each day, one attempt, and a shareable result line. No backend required.

**Why it showcases fintech skills.** It needs a real order book, a double-entry ledger, integer money, double-spend prevention, margin and liquidation, and a deterministic simulation.

## Architecture

Everything runs client-side, so hosting is a free static site. The Rust core runs in a Web Worker; React only renders and sends commands. Only the optional headline call leaves the browser.

The core is split into bounded contexts, one crate each. Cargo enforces the dependency direction, so contexts can't reach into each other. Full reasoning and the risk analysis behind it: `docs/system-design.md`.

```
kernel                      shared value types: Price, Qty, Amount, ids, InstrumentSpec, rounding
  ↑
exchange · ledger · market_data · risk · sim · run      bounded contexts, never depend on each other
  ↑
app                         GameSession, use cases, internal event queue, views
  ↑
protocol                    wire Command / Event / Query types, serde, versioning
  ↑
wasm · cli · server         adapters
```

| Crate | Owns |
| --- | --- |
| `kernel` | Value types, the one notional/rounding function |
| `exchange` | Order books, matching, self-trade prevention, time in force |
| `ledger` | Double-entry postings, reservations, fees, conservation |
| `market_data` | Candles, depth, mark price |
| `risk` | Equity, drawdown, later margin and liquidation decisions |
| `sim` | Agents, world events, per-agent RNG streams |
| `run` | Quarters, quotas, draft, score |
| `app` | Orchestrates use cases; the only crate that knows all contexts |
| `protocol` | Wire format; the only place `serde` is used |
| `wasm`, `cli`, `server` | Adapters |

Crates are added when a phase needs them, not up front (see each phase).

## Core contract

Everything outside the Rust core talks to it through `protocol`: commands in, events out, and queries for views. The browser now and a server later are just two adapters around the same core (ports and adapters).

```rust
// crates/protocol — wire types. Domain types are mapped into these.

pub struct Envelope<T> {
    pub request_id: u64, // client-chosen; lets a server deduplicate retries
    pub body: T,
}

#[non_exhaustive]
pub enum Command {
    PlaceOrder { instrument: InstrumentId, side: Side, kind: OrderKind },
    Cancel { id: OrderId },
    Advance { ticks: u32 },
    PickUpgrade { choice: u8 },
}

pub struct Sequenced<T> {
    pub seq: u64, // every outgoing event is numbered; clients resume from the last seen
    pub body: T,
}

/// Integration events: only what the player and UI need. Agent-to-agent
/// fills and ledger postings stay inside the core.
#[non_exhaustive]
pub enum Event {
    OrderAccepted { id: OrderId },
    OrderRejected { request_id: u64, reason: RejectReason },
    PlayerFill(FillView),
    Candle(CandleView),
    QuarterEnded { quarter: u8, equity: String, quota_met: bool },
    DraftOffered { options: Vec<UpgradeId> },
    RunWon { score: String },
    RunLost { reason: LoseReason },
}

#[non_exhaustive]
pub enum Query {
    BookDepth { instrument: InstrumentId, levels: u8 },
    Portfolio,
    Candles { instrument: InstrumentId, from_day: u32 },
    OpenOrders,
}

/// crates/app. Owns one instance of each context, the clock, id generators,
/// RNG streams and run state. It is a unit of work, not one giant aggregate.
pub struct GameSession { /* ... */ }

impl GameSession {
    pub fn new(seed: u64) -> Self { todo!() }
    pub fn handle(&mut self, cmd: Command) -> Vec<Sequenced<Event>> { todo!() }
    pub fn query(&self, q: Query) -> View { todo!() }
}
```

Money crosses to JavaScript as decimal strings: JS numbers are only exact up to 2⁵³, so `u64` and `i128` values would lose precision. TypeScript types are generated from the Rust protocol types.

| Adapter | Commands arrive from | Events go to |
| --- | --- | --- |
| `crates/wasm` (Phases 4–6) | React, via Worker `postMessage` | React, via `postMessage` |
| `crates/server` (Phase 9) | API gateway, via gRPC | Event stream, then WebSocket to the browser |

### Rules that keep the core portable

1. Domain crates (`kernel` through `app`) have no browser, WASM, I/O or `serde` dependencies. CI builds them for a native target to enforce it.
2. Contexts never depend on each other. Only `app` orchestrates them.
3. All game rules live in Rust. React only renders events and sends commands; no quota, fee or risk logic in TypeScript.
4. Time and randomness are inputs: the simulation clock and seeded RNG streams, never `Date.now()` or system randomness. `clippy.toml` bans the escape hatches.
5. Money is integer. Floats only appear in agent decisions and statistics, and any transcendental maths (`ln`, `exp`, `powf`, ...) goes through the `libm` crate so native and WASM builds produce identical bits.
6. One `GameSession` owns all state. A server holds many sessions, one per player.
7. Log every command, including time advances. Seed + command log + `SIM_VERSION` must reproduce the run exactly; tests check this natively and in WASM.

## Phases 0–2: Foundations, engine, ledger

Estimates assume 8–10 hours a week. These three phases are pure Rust with no UI, and they are the part interviewers will read most closely.

### Phase 0: Rust foundations (1–2 weeks)

- [ ] Rust Book core chapters: ownership, borrowing, structs and enums, traits, `Result` error handling, modules
- [x] Cargo workspace with `kernel` and `exchange` crates, pinned toolchain, workspace lints, `clippy.toml` determinism bans
- [x] GitHub Actions CI running `cargo fmt --check`, `cargo clippy -D warnings` and `cargo test`
- [ ] Decision log entries in `docs/decisions/` for the "decide now" list in `docs/system-design.md`
- [ ] Write the Phase 1 spec

### Phase 1: Matching engine (2–3 weeks) — `kernel`, `exchange`

- [ ] `kernel`: `Price`, `Qty` and `Amount` as validated integer newtypes; `Side`, `OrderId`, `AccountId`, `InstrumentId`
- [ ] `kernel`: `InstrumentSpec` (tick size, lot size, min qty, decimals) and the single notional function with an explicit rounding mode
- [ ] Order book: one sorted map of price levels per side, a FIFO queue per level (price-time priority)
- [ ] Limit orders that rest or cross; market sells by quantity; market buys by quote amount; cancels
- [ ] Time in force (GTC, IOC) and self-trade prevention
- [ ] Orders off the price/quantity grid are rejected
- [ ] Engine emits `Fill` events instead of mutating balances directly
- [ ] Unit tests for crossing, partial fills, cancels and empty-book market orders
- [ ] Property tests (`proptest`): the book never stays crossed, filled quantity never exceeds order quantity
- [ ] Throughput benchmark (`criterion`), with the result in the README

### Phase 2: Ledger (1–2 weeks) — `ledger`, first `app`

- [ ] Double-entry accounts: every fill posts balanced debits and credits
- [ ] Reservations per order: reserve notional plus the maximum fee on place, consume on each fill, release the exact remainder when the order is done
- [ ] Trading fees posted to the house, rounded up
- [ ] Rounding dust posted to the house, so money is conserved
- [ ] `app`: place/cancel use case (validate → reserve → match → settle → release); settlement never fails, a ledger error after matching is a panic
- [ ] Invariant tests: total money is conserved, every balance has its normal sign, reserved balance equals the sum of open reservations
- [ ] Decision log entries: why integers, why reservations, why fills drive the ledger, why settlement can't fail

## Phases 3–5: Simulation to first playable

The goal of these phases is a live link by about month 3. Phase 3 carries the most risk, so tune the market before building any UI.

### Phase 3: Headless simulation (2–3 weeks) — `sim`, `market_data`, `cli`

- [ ] Seeded RNG (`rand_chacha`, version pinned exactly), with an independent stream per agent and per subsystem
- [ ] Tick-based simulation clock, with ticks grouped into in-game days
- [ ] Agents: market maker, noise traders, momentum traders. Agents see a public `MarketView` and return intents that go through the same use case as the player
- [ ] Agents have real ledger accounts; refills are explicit house transfers
- [ ] `market_data`: candles, depth, and a robust mark price (not last trade)
- [ ] `cli`: prints daily candles for a given seed
- [ ] Tune agent parameters until prices look plausible: no flatlining, no runaway prices
- [ ] Determinism tests: same seed gives identical output; `Advance(n)` equals `n × Advance(1)`; golden event-stream hashes per seed, with `SIM_VERSION` bumped on intentional changes

### Phase 4: In the browser (2 weeks) — `protocol`, `wasm`

- [ ] `protocol` crate: wire types, mapping from domain types, money as strings, generated TypeScript types
- [ ] `wasm-bindgen` bindings and a `wasm-pack` build in CI
- [ ] CI check that native and WASM builds produce the same golden hashes
- [ ] Run the simulation in a Web Worker; stream integration events and answer queries
- [ ] Candlestick chart (TradingView Lightweight Charts), order book view, order entry form
- [ ] Speed controls: pause, 1x, 4x

### Phase 5: First playable and deploy (1–2 weeks) — `risk`, `run`

- [ ] One quarter with a profit quota
- [ ] Max drawdown rule with an instant game over, measured on the mark price
- [ ] Win and lose screens showing the run summary
- [ ] Deploy as a static site (Cloudflare Pages or GitHub Pages)
- [ ] **Milestone:** LinkedIn post with the live link and a short technical write-up

## Phases 6–8: Roguelike loop, depth, AI news

Phase 6 completes the full game loop, which is the interview-ready version at around month 4–5. Everything after it is iteration.

### Phase 6: Roguelike loop (2–3 weeks)

- [ ] Four quarters with escalating quotas
- [ ] Draft between quarters: pick 1 of 3 from a pool of 6–10 upgrades. Upgrades become parameters (fee schedule, risk limits) passed into other contexts
- [ ] Risk-adjusted final score, computed deterministically
- [ ] Daily seed derived from the date, one attempt per day
- [ ] Shareable result line, for example: Daily #42 · Survived Q3 · 2.7x · Sharpe 1.4
- [ ] Save games as `SIM_VERSION` + seed + command log in IndexedDB; loading replays the log. Decide how saves from an older version are handled
- [ ] **Milestone:** full game loop live and ready to demo in interviews

### Phase 7: Depth (ongoing)

- [ ] Margin, leverage and a liquidation engine: borrowing modelled as explicit `Borrowed` and `Collateral` balances, not negative balances
- [ ] Liquidation cascades handled through the internal event queue, with an iteration limit
- [ ] Short selling
- [ ] More agents: mean-reversion traders, whales, panic sellers
- [ ] World events that shift agent behavior (hacks, crackdowns, hype cycles, listings)
- [ ] New coins with personalities (a stablecoin that can depeg, a volatile meme coin)
- [ ] Cursed upgrades with big upside and a drawback
- [ ] Difficulty tiers unlocked after a first win
- [ ] Journal checkpoints if memory or save size becomes a problem

### Phase 8: AI news (1 week)

- [ ] The simulation decides each event numerically; AI only writes the headline. Nothing from the headline flows back into the simulation
- [ ] Small serverless function that holds the API key and returns headline text
- [ ] Canned headlines as a fallback, so the game works offline and without the endpoint

## Phase 9: Two-service version (4–6 weeks, after Phase 6)

The domain crates move behind a server unchanged. This phase only adds new adapters and infrastructure around them.

- [ ] `server` crate: a `tonic` gRPC server wrapping `GameSession`, one session per player, running on `tokio`
- [ ] gRPC methods mapped from the protocol: `StartRun`, `PlaceOrder`, `Cancel`, `PickUpgrade`, plus queries
- [ ] Simulation advances server-side on a timer per session; each timer tick is logged as an `Advance` command
- [ ] Commands queue and apply at tick boundaries, in arrival order; duplicates are dropped by `request_id`
- [ ] Engine publishes events to a stream keyed by session (NATS or Redis Streams to start; Kafka if you want to show it)
- [ ] API gateway (Go or FastAPI): auth, accounts, input validation, rate limiting, WebSocket to the browser
- [ ] A reconnecting client resumes from the last event `seq` it saw
- [ ] PostgreSQL for users, saved runs and leaderboards
- [ ] Leaderboard anti-cheat: verify a submitted score by replaying its seed and command log server-side on the matching `SIM_VERSION`, with limits on log length and replay time
- [ ] Docker Compose for local development, then a small deployment
- [ ] Load test: sessions per engine instance and p99 order latency, with results in the README
- [ ] **Milestone:** write-up on running the same engine as WASM in the browser and as a networked service

## Phase 1–2 starter: types and module layout

Bodies are left as `todo!()` so you write the logic yourself. Domain types derive no `serde`; the `protocol` crate maps them to wire types in Phase 4.

```
crates/kernel/src/
  lib.rs          // re-exports
  money.rs        // Price, Qty, Amount, notional + rounding
  ids.rs          // OrderId, AccountId, InstrumentId
  instrument.rs   // InstrumentSpec, Asset, CoinId

crates/exchange/src/
  lib.rs          // re-exports
  order.rs        // Side, OrderKind, TimeInForce, Order, Fill
  book.rs         // OrderBook: submit, cancel, best bid/ask
  error.rs        // ExchangeError

crates/ledger/src/      (Phase 2)
  lib.rs
  ledger.rs       // Ledger, postings, journal
  reservation.rs  // Reservation per order
  error.rs        // LedgerError
```

### kernel

Newtypes stop you from mixing a price, a quantity and an amount of money by accident. Constructors validate, so an invalid value can't exist.

```rust
/// Quote minor units (e.g. USD cents) per one whole coin. Always > 0.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Price(i64);

/// Quantity in the coin's smallest unit. Always > 0 on an order.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Qty(u64);

/// Money in quote minor units. i128 leaves room for price × qty.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Amount(i128);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct OrderId(u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct AccountId(u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct InstrumentId(u16);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Rounding { Up, Down }

#[derive(Debug, Clone)]
pub struct InstrumentSpec {
    pub id: InstrumentId,
    pub tick_size: Price, // prices must be a multiple of this
    pub lot_size: Qty,    // quantities must be a multiple of this
    pub min_qty: Qty,
    pub qty_decimals: u8, // base units per whole coin = 10^qty_decimals
}

impl InstrumentSpec {
    /// The only place in the codebase that turns price × qty into money.
    /// The caller chooses the rounding: against whoever initiates.
    pub fn notional(&self, price: Price, qty: Qty, rounding: Rounding) -> Amount { todo!() }
}
```

### exchange

`BTreeMap` keeps price levels sorted; `VecDeque` keeps time order within a level. The book never touches balances. The index is a `BTreeMap` too: `clippy.toml` bans `HashMap` in the core for determinism.

```rust
use std::collections::{BTreeMap, VecDeque};
use kernel::*;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side { Buy, Sell }

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TimeInForce {
    /// Rests until filled or cancelled.
    Gtc,
    /// Fills what it can immediately; the rest is cancelled.
    Ioc,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderKind {
    Limit { price: Price, qty: Qty, tif: TimeInForce },
    /// Sells this quantity at any price.
    MarketSell { qty: Qty },
    /// Spends up to this amount; lets the ledger reserve an exact sum.
    MarketBuy { quote: Amount },
}

#[derive(Debug, Clone)]
pub struct Order {
    pub id: OrderId,
    pub account: AccountId,
    pub instrument: InstrumentId,
    pub side: Side,
    pub kind: OrderKind,
}

/// Emitted by the book; the ledger settles it.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Fill {
    pub instrument: InstrumentId,
    pub maker: OrderId,
    pub taker: OrderId,
    pub maker_account: AccountId,
    pub taker_account: AccountId,
    pub taker_side: Side,
    /// Always the resting (maker) order's price.
    pub price: Price,
    pub qty: Qty,
}

#[derive(Debug, Default)]
pub struct OrderBook {
    bids: BTreeMap<Price, VecDeque<Resting>>, // best = highest key
    asks: BTreeMap<Price, VecDeque<Resting>>, // best = lowest key
    index: BTreeMap<OrderId, (Side, Price)>,
}

impl OrderBook {
    /// Matches against the opposite side, rests any GTC limit remainder.
    pub fn submit(&mut self, order: Order) -> Result<SubmitOutcome, ExchangeError> { todo!() }

    /// Removes a resting order and returns its unfilled quantity.
    pub fn cancel(&mut self, id: OrderId) -> Result<Qty, ExchangeError> { todo!() }

    pub fn best_bid(&self) -> Option<Price> { self.bids.keys().next_back().copied() }
    pub fn best_ask(&self) -> Option<Price> { self.asks.keys().next().copied() }
}
```

### ledger (Phase 2)

Every journal entry must sum to zero per asset. Each balance has a normal sign: asset balances never go below zero, and liabilities (Phase 7) never go above it. Reservations are tracked per order, so funds behind an open order can't be spent twice and the exact remainder is released when it finishes.

```rust
use std::collections::BTreeMap;
use kernel::*;

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum Asset { Usd, Coin(InstrumentId) }

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum BalanceKind {
    Available,
    Reserved,
    // Phase 7: Collateral, Borrowed (a liability)
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct LedgerKey {
    pub owner: AccountId,
    pub asset: Asset,
    pub kind: BalanceKind,
}

#[derive(Debug, Clone)]
pub struct Posting {
    pub key: LedgerKey,
    /// Positive = credit, negative = debit.
    pub amount: i128,
}

#[derive(Debug, Clone)]
pub enum EntryReason {
    Deposit,
    Reserve(OrderId),
    Release(OrderId),
    Trade { maker: OrderId, taker: OrderId },
    Fee(OrderId),
    RoundingDust(OrderId),
}

#[derive(Debug, Clone)]
pub struct JournalEntry {
    pub reason: EntryReason,
    pub postings: Vec<Posting>,
}

/// AccountId(0) = the house: it issues starting funds and collects fees and dust.
pub const HOUSE: AccountId = AccountId(0);

#[derive(Debug, Default)]
pub struct Ledger {
    journal: Vec<JournalEntry>,
    balances: BTreeMap<LedgerKey, i128>,
    reservations: BTreeMap<OrderId, Reservation>,
}

impl Ledger {
    /// Rejects entries that do not balance per asset, or that would give a
    /// non-house balance the wrong sign. Applies all postings or none.
    pub fn post(&mut self, entry: JournalEntry) -> Result<(), LedgerError> { todo!() }

    pub fn balance(&self, key: LedgerKey) -> i128 {
        self.balances.get(&key).copied().unwrap_or(0)
    }
}
```

### First tests to write

- [ ] A price or quantity off the instrument's grid is rejected
- [ ] `notional` rounds in the requested direction, including at overflow edges
- [ ] A limit buy below the best ask rests and produces no fills
- [ ] A crossing order fills at the maker's price, best price first, oldest order first
- [ ] A partial fill leaves the correct remainder resting
- [ ] An IOC remainder is cancelled, not rested
- [ ] A market order against an empty book returns no fills and does not rest
- [ ] A market buy never spends more than its quote amount
- [ ] Self-trade prevention applies when both sides belong to one account
- [ ] Cancelling an unknown id returns an error
- [ ] An unbalanced journal entry is rejected and changes nothing
- [ ] Reserving more than the available balance is rejected
- [ ] Property test: after any sequence of orders, the best bid is below the best ask

## Working habits

These habits turn the project into interview material, not just a game.

- **Spec before each phase.** One page: goal, scope, out of scope, done criteria. The specs become the project docs.
- **Decision log.** A short entry per significant choice in `docs/decisions/`: the options, what you picked, why. Each entry is an interview answer.
- **Write the core yourself.** Use AI freely for UI, tooling, CI and debugging. Write the domain crates by hand so you can defend every line.
- **Tests before UI.** Phases 1–3 must be green in CI before Phase 4 starts.
- **Post at milestones.** A LinkedIn post at Phase 5 and Phase 6, each with the live link and one technical lesson.
- **Keep the README current.** Live link, a GIF of gameplay, architecture summary, benchmark numbers, and links to the decision log.
