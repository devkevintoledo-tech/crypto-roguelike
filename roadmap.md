# Crypto Roguelike — Build Roadmap

Oct 13, 2026 · @Kevin

## Overview

A single-player, browser-based roguelike where you trade fake crypto in a market of simulated agents. The goal is a live, playable portfolio piece for fintech roles, with a Rust core that shows matching, ledgers, risk and determinism.

**The run.** One run is a trading year of 4 quarters. Each quarter has a rising profit quota (roughly +20%, +50%, 2x, 4x). Between quarters, the player drafts 1 of 3 upgrades.

**Lose conditions.**

- Miss the quarter's quota: fired.
- Breach the max drawdown from peak equity: funding pulled instantly.
- Get liquidated on a leveraged position (once leverage is unlocked).

**Win condition.** Survive all 4 quarters. Score = final equity adjusted for volatility (a simplified Sharpe ratio). Endless mode continues with rising quotas.

**Daily run.** Same seed for everyone each day, one attempt, and a shareable result line. No backend required.

**Why it showcases fintech skills.** It needs a real order book, a double-entry ledger, integer money, double-spend prevention, margin and liquidation, and a deterministic simulation.

## Architecture

Everything runs client-side, so hosting is a free static site. The `engine` crate is the showcase: pure Rust, no browser dependencies, tested natively.

&#91;embedded content: architecture · 3 Rust crates in a Web Worker, React UI, optional AI endpoint\]

The UI sends commands to the worker and renders the snapshots it gets back. Only the optional headline call leaves the browser.

## Core contract

Everything outside the Rust core talks to it through one interface: a `Command` in, a list of `Event`s out. The browser now and a server later are just two adapters around the same core (ports and adapters).

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Command {
    PlaceOrder { side: Side, kind: OrderKind, qty: Qty },
    Cancel { id: OrderId },
    Advance { ticks: u32 },
    PickUpgrade { choice: u8 },
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum Event {
    OrderAccepted { id: OrderId },
    OrderRejected { reason: RejectReason },
    Filled(Fill),
    Candle { day: u32, open: Price, high: Price, low: Price, close: Price },
    QuarterEnded { quarter: u8, equity: i128, quota_met: bool },
    DraftOffered { options: Vec<UpgradeId> },
    RunWon { score: i64 },
    RunLost { reason: LoseReason },
}

/// Lives in crates/sim. Owns the book, ledger, agents, clock, RNG and run state.
pub struct GameSession { /* ... */ }

impl GameSession {
    pub fn new(seed: u64) -> Self { todo!() }
    pub fn handle(&mut self, cmd: Command) -> Vec<Event> { todo!() }
}
```

The types in `types.rs` also derive `Serialize` and `Deserialize`, since they appear in the contract.

| Adapter | Commands arrive from | Events go to |
| --- | --- | --- |
| `crates/wasm` (Phases 4–6) | React, via Worker `postMessage` | React, via `postMessage` |
| `engine-server` (Phase 9) | API gateway, via gRPC | Event stream, then WebSocket to the browser |

### Rules that keep the core portable

1. `engine` and `sim` have no browser or WASM dependencies. CI builds them for a native target to enforce it.
2. All game rules live in Rust. React only renders events and sends commands; no quota, fee or risk logic in TypeScript.
3. Time and randomness are inputs: the simulation clock and the seeded RNG, never `Date.now()` or system randomness.
4. `Command` and `Event` derive `serde`, so they are already the wire format.
5. One `GameSession` owns all state. A server holds many sessions, one per player.
6. Log every command. Replaying the seed plus the command log must reproduce the run exactly; a test checks this.

## Phases 0–2: Foundations, engine, ledger

Estimates assume 8–10 hours a week. These three phases are pure Rust with no UI, and they are the part interviewers will read most closely.

### Phase 0: Rust foundations (1–2 weeks)

- [ ] Rust Book core chapters: ownership, borrowing, structs and enums, traits, `Result` error handling, modules
- [ ] Create the Cargo workspace with `engine`, `sim` and `wasm` crates plus a `web/` folder
- [ ] GitHub Actions CI running `cargo fmt --check`, `cargo clippy` and `cargo test`
- [ ] Write the Phase 1 spec

### Phase 1: Matching engine (2–3 weeks)

- [ ] Core types: `Price` and `Qty` as integer newtypes, `Side`, `OrderId`, `Order`, `Fill`
- [ ] Order book: one sorted map of price levels per side, a FIFO queue per level (price-time priority)
- [ ] Limit orders that rest or cross, market orders, cancels
- [ ] Engine emits `Fill` events instead of mutating balances directly
- [ ] Unit tests for crossing, partial fills, cancels and empty-book market orders
- [ ] Property tests (`proptest`): the book never stays crossed, filled quantity never exceeds order quantity
- [ ] Throughput benchmark (`criterion`), with the result in the README

### Phase 2: Ledger (1–2 weeks)

- [ ] Double-entry accounts: every fill posts balanced debits and credits
- [ ] Reserve funds when an order is placed, release them on cancel (prevents double-spending)
- [ ] Trading fees posted to a fee account
- [ ] Invariant tests: total money is conserved, no balance goes negative
- [ ] Decision log entries: why integers, why reservations, why fills drive the ledger

## Phases 3–5: Simulation to first playable

The goal of these phases is a live link by about month 3. Phase 3 carries the most risk, so tune the market before building any UI.

### Phase 3: Headless simulation (2–3 weeks)

- [ ] Seeded random number generator (`rand_chacha`): the same seed always produces the same market
- [ ] Tick-based simulation clock, with ticks grouped into in-game days
- [ ] Agents: market maker, noise traders, momentum traders
- [ ] A command-line tool that prints daily candles (open, high, low, close) for a given seed
- [ ] Tune agent parameters until prices look plausible: no flatlining, no runaway prices
- [ ] Determinism test: two runs with the same seed produce identical output

### Phase 4: In the browser (2 weeks)

- [ ] `wasm-bindgen` bindings and a `wasm-pack` build in CI
- [ ] Run the simulation in a Web Worker and stream state snapshots to React
- [ ] Candlestick chart (TradingView Lightweight Charts), order book view, order entry form
- [ ] Speed controls: pause, 1x, 4x

### Phase 5: First playable and deploy (1–2 weeks)

- [ ] One quarter with a profit quota
- [ ] Max drawdown rule with an instant game over
- [ ] Win and lose screens showing the run summary
- [ ] Deploy as a static site (Cloudflare Pages or GitHub Pages)
- [ ] **Milestone:** LinkedIn post with the live link and a short technical write-up

## Phases 6–8: Roguelike loop, depth, AI news

Phase 6 completes the full game loop, which is the interview-ready version at around month 4–5. Everything after it is iteration.

### Phase 6: Roguelike loop (2–3 weeks)

- [ ] Four quarters with escalating quotas
- [ ] Draft between quarters: pick 1 of 3 from a pool of 6–10 upgrades
- [ ] Risk-adjusted final score
- [ ] Daily seed derived from the date, one attempt per day
- [ ] Shareable result line, for example: Daily #42 · Survived Q3 · 2.7x · Sharpe 1.4
- [ ] Save games: serialize state with `serde` into IndexedDB
- [ ] **Milestone:** full game loop live and ready to demo in interviews

### Phase 7: Depth (ongoing)

- [ ] Margin, leverage and a liquidation engine
- [ ] Short selling
- [ ] More agents: mean-reversion traders, whales, panic sellers
- [ ] World events that shift agent behavior (hacks, crackdowns, hype cycles, listings)
- [ ] New coins with personalities (a stablecoin that can depeg, a volatile meme coin)
- [ ] Cursed upgrades with big upside and a drawback
- [ ] Difficulty tiers unlocked after a first win

### Phase 8: AI news (1 week)

- [ ] The simulation decides each event numerically; AI only writes the headline
- [ ] Small serverless function that holds the API key and returns headline text
- [ ] Canned headlines as a fallback, so the game works offline and without the endpoint

## Phase 9: Two-service version (4–6 weeks, after Phase 6)

The `engine` and `sim` crates move behind a server unchanged. This phase only adds new adapters and infrastructure around them.

- [ ] `engine-server` crate: a `tonic` gRPC server wrapping `GameSession`, one session per player, running on `tokio`
- [ ] gRPC methods mapped from the `Command` contract: `StartRun`, `PlaceOrder`, `Cancel`, `PickUpgrade`
- [ ] Simulation advances server-side on a timer per session
- [ ] Engine publishes events to a stream keyed by session (NATS or Redis Streams to start; Kafka if you want to show it)
- [ ] API gateway (Go or FastAPI): auth, accounts, input validation, rate limiting, WebSocket to the browser
- [ ] Each event carries a sequence number; a reconnecting client resumes from the last one it saw
- [ ] PostgreSQL for users, saved runs and leaderboards
- [ ] Leaderboard anti-cheat: verify a submitted score by replaying its seed and command log server-side
- [ ] Docker Compose for local development, then a small deployment
- [ ] Load test: sessions per engine instance and p99 order latency, with results in the README
- [ ] **Milestone:** write-up on running the same engine as WASM in the browser and as a networked service

## Phase 1–2 starter: types and module layout

The engine crate starts with four modules. Bodies are left as `todo!()` so you write the logic yourself.

```
crates/engine/src/
  lib.rs      // re-exports
  types.rs    // Price, Qty, ids, Side, Order, Fill
  book.rs     // OrderBook: submit, cancel, best bid/ask
  ledger.rs   // double-entry accounts and journal
  error.rs    // EngineError, LedgerError
```

### types.rs

Newtypes stop you from mixing a price with a quantity by accident. Money is always integer.

```rust
/// Price in ticks (e.g. 1 tick = 0.01 USD).
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Price(pub i64);

/// Quantity in the coin's smallest unit.
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct Qty(pub u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct OrderId(pub u64);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct AccountId(pub u32);

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Side { Buy, Sell }

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum OrderKind {
    Limit { price: Price },
    /// Fills against the book; any unfilled rest is cancelled.
    Market,
}

#[derive(Debug, Clone)]
pub struct Order {
    pub id: OrderId,
    pub account: AccountId,
    pub side: Side,
    pub kind: OrderKind,
    pub qty: Qty,
}

/// Emitted by the book; the ledger settles it. The book never touches balances.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Fill {
    pub maker: OrderId,
    pub taker: OrderId,
    pub maker_account: AccountId,
    pub taker_account: AccountId,
    pub taker_side: Side,
    /// Always the resting (maker) order's price.
    pub price: Price,
    pub qty: Qty,
}
```

### book.rs

`BTreeMap` keeps price levels sorted; `VecDeque` keeps time order within a level.

```rust
use std::collections::{BTreeMap, HashMap, VecDeque};
use crate::{error::EngineError, types::*};

#[derive(Debug, Clone)]
struct Resting {
    id: OrderId,
    account: AccountId,
    qty: Qty,
}

#[derive(Debug, Default)]
pub struct OrderBook {
    bids: BTreeMap<Price, VecDeque<Resting>>, // best = highest key
    asks: BTreeMap<Price, VecDeque<Resting>>, // best = lowest key
    index: HashMap<OrderId, (Side, Price)>,   // lookup only, never iterated
}

impl OrderBook {
    /// Matches against the opposite side, rests any limit remainder.
    pub fn submit(&mut self, order: Order) -> Result<Vec<Fill>, EngineError> { todo!() }

    /// Removes a resting order and returns its unfilled quantity.
    pub fn cancel(&mut self, id: OrderId) -> Result<Qty, EngineError> { todo!() }

    pub fn best_bid(&self) -> Option<Price> { self.bids.keys().next_back().copied() }
    pub fn best_ask(&self) -> Option<Price> { self.asks.keys().next().copied() }
}
```

### ledger.rs

Every journal entry must sum to zero per asset. Balances are split into available and reserved, so funds behind an open order cannot be spent twice.

```rust
use std::collections::BTreeMap;
use crate::{error::LedgerError, types::*};

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub struct CoinId(pub u16);

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum Asset { Usd, Coin(CoinId) }

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord, Hash)]
pub enum Bucket { Available, Reserved }

#[derive(Debug, Clone)]
pub struct Posting {
    pub account: AccountId,
    pub asset: Asset,
    pub bucket: Bucket,
    /// Positive = credit, negative = debit. i128 leaves room for price × qty.
    pub amount: i128,
}

#[derive(Debug, Clone)]
pub enum EntryReason {
    Deposit,
    Reserve(OrderId),
    Release(OrderId),
    Trade { maker: OrderId, taker: OrderId },
    Fee(OrderId),
}

#[derive(Debug, Clone)]
pub struct JournalEntry {
    pub reason: EntryReason,
    pub postings: Vec<Posting>,
}

/// AccountId(0) = the house: it issues starting funds and collects fees.
pub const HOUSE: AccountId = AccountId(0);

#[derive(Debug, Default)]
pub struct Ledger {
    journal: Vec<JournalEntry>,
    // BTreeMap, not HashMap: deterministic iteration for replays.
    balances: BTreeMap<(AccountId, Asset, Bucket), i128>,
}

impl Ledger {
    /// Rejects entries that do not balance per asset, or that would push a
    /// non-house balance below zero. Applies all postings or none.
    pub fn post(&mut self, entry: JournalEntry) -> Result<(), LedgerError> { todo!() }

    pub fn balance(&self, account: AccountId, asset: Asset, bucket: Bucket) -> i128 {
        self.balances.get(&(account, asset, bucket)).copied().unwrap_or(0)
    }
}
```

### First tests to write

- [ ] A limit buy below the best ask rests and produces no fills
- [ ] A crossing order fills at the maker's price, best price first, oldest order first
- [ ] A partial fill leaves the correct remainder resting
- [ ] A market order against an empty book returns no fills and does not rest
- [ ] Cancelling an unknown id returns an error
- [ ] An unbalanced journal entry is rejected and changes nothing
- [ ] Reserving more than the available balance is rejected
- [ ] Property test: after any sequence of orders, the best bid is below the best ask

## Working habits

These habits turn the project into interview material, not just a game.

- **Spec before each phase.** One page: goal, scope, out of scope, done criteria. The specs become the project docs.
- **Decision log.** A short entry per significant choice: the options, what you picked, why. Each entry is an interview answer.
- **Write the core yourself.** Use AI freely for UI, tooling, CI and debugging. Write the engine, ledger and simulation by hand so you can defend every line.
- **Tests before UI.** Phases 1–3 must be green in CI before Phase 4 starts.
- **Post at milestones.** A LinkedIn post at Phase 5 and Phase 6, each with the live link and one technical lesson.
- **Keep the README current.** Live link, a GIF of gameplay, architecture summary, benchmark numbers, and links to the decision log.
