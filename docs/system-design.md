# Crypto Roguelike — System Design and Risk Analysis

Oct 14, 2026 · companion to `roadmap.md`

This document has two parts:

1. **Risk register.** Decisions that are cheap to make now and expensive to change later. Each item says what breaks, when it starts to hurt, and what to do now.
2. **System design.** A domain-driven layout for the Rust core, so contexts stay separate as the game grows (more coins, margin, shorting, a server).

Game progression (quotas, upgrades, balance) is deliberately out of scope. It appears here only as a bounded context with a clear boundary.

---

# Part 1 — Risk register

Severity: **Critical** = corrupts money or breaks determinism/replay. **High** = forces a rewrite of a context. **Medium** = painful but local.

## 1. Money and numbers

### 1.1 Reservations are harder than "reserve on place, release on cancel" — Critical

The roadmap's model only works cleanly for limit orders on the exact fill price. The cases that break it:

| Case | Problem | Fix |
| --- | --- | --- |
| Market buy | Cost is unknown until it fills; nothing to reserve | Specify market buys by **quote amount** ("spend up to 500 USD"), as Binance does with `quoteOrderQty`. Reserve exactly that. Market sells are fine: reserve the coin qty. |
| Fees | If the fee isn't reserved, a fill can settle but the fee posting fails, mid-match | Reserve notional **plus the maximum fee** (taker rate) up front |
| Price improvement | Limit buy at 105 fills at 100: 5/unit is over-reserved | On each fill, consume `fill_price × qty + fee` from the reservation; release the rest when the order is done |
| Partial fills, then cancel | Remainder must be released exactly, with rounding | Track a `Reservation { order, remaining }` entity, not just a bucket total |
| Rounding | Proportional release of a reservation creates dust | Never compute release proportionally: release `remaining` at the end |

**Invariant to test:** for every account and asset, `Reserved balance == Σ remaining over that account's open reservations`.

**The key design consequence:** because funds are reserved before matching, **settling a fill must never fail**. If the ledger rejects a fill posting, that is a bug, not a business error, and the right response is a panic (the book has already changed and can't be rolled back cleanly). Build the types so this is true by construction: `settle_fill` consumes a reservation that is guaranteed to cover it.

### 1.2 Units, scales and rounding — Critical

`Price(i64)` and `Qty(u64)` don't say what they're measured in. Once there's more than one coin, this causes silent errors.

- `price × qty` is not money. With price in *quote minor units per whole coin* and qty in *base units* (e.g. 10⁻⁸ coin), notional is `price × qty / 10^qty_decimals`. That division has to round.
- Give each instrument an `InstrumentSpec`: `tick_size`, `lot_size`, `min_qty`, `qty_decimals`, `quote_decimals`. Reject orders that aren't on the grid. This stops "0.0000001 coin" orders and makes rounding predictable.
- Put **one** function in the whole codebase that turns price × qty into money, with an explicit rounding mode. Rule: round **against the account that initiates** (buyer pays rounded up, seller receives rounded down), and post the dust to the house so money is still conserved.
- Fees: `fee = ceil(notional × bps / 10_000)`. Rounding down lets players dodge fees by splitting orders into tiny pieces.
- Introduce a separate `Amount(i128)` type for money in quote minor units. `Price`, `Qty` and `Amount` should not be interchangeable.
- Validate `Price > 0` and `Qty > 0` at construction. `Price(i64)` currently allows negative prices.
- Use checked arithmetic in money code, and set `overflow-checks = true` in the release profile so overflow panics instead of wrapping.

### 1.3 Floats break determinism across platforms — Critical (from Phase 3, fatal in Phase 9)

Replay (rule 6) and server-side anti-cheat both need the browser build (WASM) and the server build (native) to produce identical numbers.

- `+ − × ÷` and `sqrt` on `f64` are IEEE-exact and give the same result everywhere. Rust doesn't fuse operations into FMA on its own.
- `ln`, `exp`, `pow`, `sin` and similar **are not guaranteed**. Native builds call the system maths library (glibc, Apple libm). WASM uses Rust's own port. They can differ in the last bit, and in a feedback-loop simulation one bit becomes a different market a few hundred ticks later.
- Random distributions (normal, exponential, log-normal for agent sizes and returns) use exactly these functions.
- Summing floats in a different order gives a different result. Iterate in a fixed order.

**What to do:**
- Use money and prices only as integers. Floats are allowed only inside agent decision-making and statistics.
- Every transcendental function goes through the `libm` crate (pure Rust, same output on every target). Ban `f64::ln`, `f64::exp` and similar in the simulation crates with clippy `disallowed_methods`.
- Compute the score (Sharpe) the same way, or in fixed point.
- Add a CI job that runs the same seeds natively and under WASM (e.g. `wasm-pack test --node`) and compares event-stream hashes.

### 1.4 Integers don't survive the trip to JavaScript — High (Phase 4)

JS numbers are exact only up to 2⁵³. `u64` order IDs and `i128` balances will lose precision silently, or fail to serialize, when they cross `postMessage`.

- In the **protocol** (wire) types, send money as decimal strings (`"1234.56"`) or as integers already scaled for display. Formatting belongs in Rust (rule 2), so TS just shows strings.
- Generate the TypeScript types from Rust (`ts-rs`, `tsify` or `specta`) so the two sides can't drift apart.
- Keep domain types and wire types separate (see Part 2). Domain types shouldn't change shape because JS has a number problem.

## 2. Determinism and replay

### 2.1 One shared RNG couples everything — High

If every agent draws from the same RNG, adding one agent, or changing the order of two `rng.gen()` calls, changes the whole market for every seed. Daily seeds, saved replays and golden tests all break on every balance tweak.

- Derive **independent streams**: `ChaCha8Rng::seed_from_u64(seed)` then `set_stream(stream_id)`, one stream per agent and one per subsystem (world events, draft offers). Adding a new agent then doesn't disturb existing ones.
- `rand` guarantees stable output for ChaCha itself, but **not** for distributions like `gen_range` across versions. Pin `rand` exactly, or write the few samplers you need by hand on top of raw `u64`s. Treat a `rand` upgrade as a simulation version bump.

### 2.2 Replays need a simulation version — High

Any change to agents, fees or rounding changes what a given seed + command log produces. Old saves and old leaderboard entries then replay to different results.

- Add `SIM_VERSION` to every save, replay and score submission.
- Keep golden tests: for fixed seeds and command logs, CI checks the hash of the event stream. When a change is intentional, update the hash and bump `SIM_VERSION`. When it isn't, the test just caught a determinism bug.
- In Phase 9, the anti-cheat verifier has to run the version the run was played on, so keep old engine versions available, or only accept submissions from the current version.
- The static site can update during a player's run. Decide now what happens: refuse to load saves from another version, or keep old WASM bundles.

### 2.3 Hidden sources of non-determinism — High

- `HashMap` and `HashSet` iterate in random order. Ban them in domain crates with clippy `disallowed_types`, or allow them only for lookups (as `book.rs` already notes). Note that dependencies can use them internally too.
- Agent turn order inside a tick changes outcomes. Define it explicitly, e.g. shuffle with the world RNG stream each tick.
- **`Advance { ticks: 4 }` must equal four `Advance { ticks: 1 }`.** If it doesn't, the result depends on UI speed settings and frame rate. Write a property test for this.
- Every input goes into the command log: player commands, draft picks, and in Phase 9 the server's timer ticks. A server that advances time without logging `Advance` commands can't be replayed.
- `std::time`, `Instant`, `thread_rng` and `getrandom` must never be reachable from domain crates. Ban them with clippy.

## 3. Order book and matching

### 3.1 Missing matching rules — High

- **Self-trade prevention.** A market maker quoting both sides, or the player, will match against themselves. Pick a rule (cancel the resting order, cancel the incoming one, or both) and apply it in the book.
- **Time in force.** Without expiry, agents leave stale orders forever and the book grows without limit. Support at least GTC (good till cancelled) and IOC (immediate or cancel), and have agents cancel or replace their own quotes.
- **Order IDs** are assigned by the session from one monotonic counter that is part of state (so replays give the same IDs). The player and the agents share that ID space.

### 3.2 Performance traps — Medium

- Cancelling from the middle of a `VecDeque` is O(n). With market makers re-quoting every tick, this becomes the hot path. Options: lazy deletion (mark cancelled, skip on match) or a slab with an intrusive linked list per level. Benchmark before choosing, but keep the book API free of `VecDeque` so you can swap later.
- Remove empty price levels from the `BTreeMap`, or best bid/ask will return a level with nothing in it.
- A market order on a thin book can sweep to absurd prices. Add price bands (reject or stop matching beyond X% from the mark price). This also protects agents from each other.

### 3.3 Cancel races — Medium (Phase 4, real in Phase 9)

The UI shows a resting order, the player clicks cancel, but the order filled in the tick that just ran. That is a normal rejection (`OrderNotFound` / `AlreadyFilled`) shown to the player, not an error. In Phase 9, commands arriving mid-tick need one rule: **commands queue and apply at tick boundaries, in arrival order**, and the log records that order.

## 4. Ledger and accounting

### 4.1 The journal grows without limit — High

If every agent has real ledger accounts (they should, or money isn't conserved), every agent fill posts journal entries. Thousands of entries per in-game day, kept in browser memory and written to IndexedDB on every save.

- Keep balances as the source of truth for the current state, plus a bounded journal: periodic **checkpoints** (a balance snapshot), keeping only entries after the last one. The player's own entries can be kept in full for the trade history.
- The conservation invariant still holds: checkpoint balances + the entries after it = current balances.

### 4.2 Margin and shorting break "no negative balances" — High (Phase 7)

Borrowing makes a balance negative, which `post` currently rejects. Don't relax the invariant. Model borrowing as **explicit accounts**: a `Loan` liability per account and asset, a `Collateral` bucket, and interest postings. Design the account key now so it can grow:

```rust
// Instead of (AccountId, Asset, Bucket) with a fixed Bucket enum:
pub struct LedgerKey { owner: AccountId, asset: Asset, kind: BalanceKind }
pub enum BalanceKind { Available, Reserved, Collateral, Borrowed /* liability */ }
```

and give each kind a normal sign (assets are ≥ 0; liabilities are ≤ 0), so the "no negative" check becomes "every balance has its normal sign".

### 4.3 Agents running out of money — Medium

Agents that lose everything stop trading and the market dries up. Refill them with explicit, logged house transfers (`EntryReason::AgentRefill`) rather than letting them have infinite money. Infinite money breaks conservation and makes invariant tests meaningless.

### 4.4 Mark price can be manipulated — High

Equity = cash + holdings × **some price**. If that price is the last trade, the player can buy a tiny amount at a high ask just before the quarter ends and inflate their equity (this is "marking the close", a real form of market manipulation). The reverse also hurts the player: one bad agent trade on a thin book can trigger the max-drawdown game over.

Use a **robust mark price**: e.g. a time-weighted mid price over the last N ticks, or the median of recent mids. All risk checks (drawdown, quota, liquidation in Phase 7) use the mark price, never the last trade. It's a good interview topic too.

## 5. Contract and protocol

### 5.1 One event stream is doing two jobs — High (Phase 4)

`Event` currently mixes **domain events** (every agent fill) with **what the UI needs** (the player's fills, candles, quarter results). Sending every agent fill across `postMessage` at 4x speed will flood the UI.

- Keep internal domain events inside the core.
- Expose **integration events** (player-relevant, small) plus **queries** that return views (book depth top N, portfolio, recent candles). This is CQRS: commands change state, queries read projections. The current contract has no query side.

### 5.2 Protocol versioning and Phase 9 needs — Medium, cheap now

- Give every outgoing event a `seq: u64` now. Phase 9 reconnect/resume needs it, and adding it later changes every message.
- Give every command a client `request_id` so retries after a reconnect can be deduplicated.
- Mark wire enums `#[non_exhaustive]` and include a protocol version in the handshake.

### 5.3 Saves: snapshot or replay — Medium

Two options:

- **Snapshot** the whole state with serde. Fast to load, but every domain type needs `Serialize`, and any change to a struct breaks old saves.
- **Replay**: save `(SIM_VERSION, seed, command log)`. Tiny, checks determinism every time you load, and domain types stay free of serde. Loading gets slower as runs get longer.

Recommendation: replay-based saves, adding snapshots later only if load time becomes a problem. This uses the same mechanism as anti-cheat, so it gets tested constantly.

## 6. Later phases

- **Liquidation cascades (Phase 7).** A liquidation submits a market order, which moves the price, which triggers another liquidation, within the same tick. If contexts call each other directly, you get re-entrancy bugs. The in-process event queue in Part 2 handles this: each step runs until the queue is empty, with a maximum iteration count as a safety net.
- **The player as an adversary.** Agents with simple rules can be exploited (spoofing momentum traders, sweeping a thin book). Decide what agents can see: the public book only, never order owners. Treat "the player found a money printer" as a test case in Phase 3 tuning.
- **AI headlines (Phase 8).** They must never feed back into the simulation, or determinism is gone. Headlines are produced after the fact from events, outside the core.
- **Anti-cheat cost (Phase 9).** Replaying a submitted run uses server CPU. Limit the command-log length and the replay time per submission, or it becomes a denial-of-service vector.

---

# Part 2 — System design

## Principles

1. **Bounded contexts are crates.** Cargo enforces dependency direction, so a context *cannot* reach into another one by accident.
2. **The domain is pure.** No serde, no WASM, no I/O, no clock, no system randomness. It takes inputs and returns decisions and events.
3. **One application layer orchestrates.** Contexts never call each other. The application layer runs each use case, in a fixed order, inside one session.
4. **Adapters are thin.** WASM, CLI and the gRPC server translate between the protocol and the application layer, nothing else.
5. **Nobody is special.** The exchange and the ledger don't know which account is the player. Only the Run and Risk contexts do.

## Subdomains

| Subdomain | Type | Why |
| --- | --- | --- |
| Exchange (matching) | Core | The fintech showcase |
| Accounting (ledger) | Core | The fintech showcase |
| Risk (equity, drawdown, later margin/liquidation) | Core | Where money and game rules meet |
| Run (quarters, quotas, draft, score) | Core (for the game) | Defined later, but its boundary is defined now |
| Market simulation (agents, world events) | Supporting | Generates the market; replaceable |
| Market data (candles, depth, mark price) | Supporting | Read models built from fills |
| Narrative (headlines) | Generic | Outside the core entirely |

## Bounded contexts

### Shared kernel — `kernel`
Tiny on purpose: `Price`, `Qty`, `Amount`, `AccountId`, `OrderId`, `InstrumentId`, `Asset`, `Side`, `InstrumentSpec`, the single notional/rounding function, and `Tick`. Everything depends on it, so changes here are expensive. Keep it to value types with no behaviour beyond arithmetic and validation.

### Exchange — `exchange`
- **Aggregate:** `OrderBook`, one per instrument.
- **Owns:** price-time priority, order types, time in force, self-trade prevention, price bands.
- **In:** `submit(order)`, `cancel(id)`. **Out:** `Fill`s, `OrderRested`, `OrderCancelled`, `OrderExpired`.
- **Does not know about:** balances, fees, the player, quotas.

### Accounting — `ledger`
- **Aggregate:** `Ledger` (balances + journal + reservations).
- **Owns:** double-entry postings, reservations, fee postings, conservation, checkpoints.
- **In:** `reserve(order, amount)`, `settle(fill, fee_schedule)`, `release(order)`, `transfer(...)`.
- **Out:** `Posted` entries, balance queries.
- **Does not know about:** order books, prices beyond what a fill tells it.
- Fee rates come in as a `FeeSchedule` value. The ledger doesn't know that an upgrade lowered them.

### Market data — `market_data`
- Projections built from fills and the book's top levels: candles, depth snapshots, **mark price**.
- Read-only for everyone else. Risk takes the mark price from here.

### Risk — `risk`
- **Owns:** equity calculation, peak equity, drawdown, and later margin requirements and liquidation decisions.
- **In:** balances (as a read-only view), mark prices, a `RiskLimits` value.
- **Out:** decisions, e.g. `PreTradeCheck::Rejected(reason)`, `DrawdownBreached`, `LiquidationRequired { account, ... }`.
- It **decides**; it doesn't act. The application layer turns a liquidation decision into orders.

### Market simulation — `sim`
- **Owns:** agents, behaviours, world events, per-agent RNG streams.
- Agents see a `MarketView` (public book, recent candles, their own orders and balances) and return **intents** (`Place`, `Cancel`). They never touch the book directly.
- Agent intents go through **the same use case as player commands**, so agents get the same reservations, fees, risk checks and validation. Fair, and one code path to test.

### Run — `run`
- **Owns:** quarters, quotas, draft, upgrades, score, win/lose conditions.
- **In:** clock ticks, equity snapshots, risk decisions. **Out:** `QuarterEnded`, `DraftOffered`, `RunWon`, `RunLost`.
- Upgrades produce a `Modifiers` value (fee discount, risk limits, leverage cap). The application layer passes the relevant parts into other contexts as parameters. **No context reads Run's state.**

### Narrative — outside the core
Consumes integration events in the adapter layer (or the serverless function) and returns text. Nothing flows back.

## Context map

```
                        ┌──────────────┐
                        │    kernel    │  shared value types
                        └──────▲───────┘
         ┌──────────┬──────────┼──────────┬──────────┬──────────┐
     exchange     ledger   market_data   risk       sim        run
         ▲          ▲          ▲          ▲          ▲          ▲
         └──────────┴──────────┴────┬─────┴──────────┴──────────┘
                                    │
                              ┌─────┴─────┐
                              │    app    │  GameSession, use cases, event queue
                              └─────▲─────┘
                              ┌─────┴─────┐
                              │ protocol  │  wire Command/Event/Query DTOs, serde, versioning
                              └─────▲─────┘
                 ┌──────────────────┼──────────────────┐
               wasm                cli               server (Phase 9)
```

Rules: arrows point only upward. Contexts at the same level never depend on each other. Only `protocol` and the adapters use serde.

## Workspace layout

```
crates/
  kernel/        shared value types, notional + rounding
  exchange/      order book, matching
  ledger/        double-entry, reservations
  market_data/   candles, depth, mark price
  risk/          equity, drawdown, (margin, liquidation)
  sim/           agents, world events, RNG streams
  run/           quarters, quotas, draft, score
  app/           GameSession, use cases, internal event queue, views
  protocol/      wire types, mapping, versioning, TS type generation
  wasm/          wasm-bindgen adapter (Worker)
  cli/           headless runner: candles for a seed, replay, golden hashes
  server/        Phase 9 gRPC adapter
web/             React UI
```

This replaces the roadmap's `engine` / `sim` / `wasm` split. You don't need all of these on day one; see "Adoption order" below.

## The application layer

`GameSession` is the **unit of work**, not a giant aggregate. It owns one instance of each context plus the clock, ID generators and RNG streams, and runs use cases.

### Example: place an order (player or agent)

```
1. validate        kernel: on the instrument's tick/lot grid, qty > 0
2. pre-trade risk  risk:   allowed for this account under current limits?
3. reserve         ledger: notional + max fee (or quote amount for market buys)
4. match           exchange: submit → fills, rested/expired remainder
5. settle          ledger: settle each fill (infallible: panic = bug)
6. release         ledger: release the remaining reservation if the order is done
7. project         market_data: update candles, depth, mark price
8. drain queue     risk / run policies react to the new state (see below)
```

If step 2 or 3 fails, nothing has changed and the order is rejected. After step 4, nothing is allowed to fail.

### Internal event queue

Contexts react to each other through **policies** run by the application layer:

```
loop {
    let Some(event) = queue.pop_front() else { break };
    for policy in policies { queue.extend(policy.react(&event, &state)) }
    guard.iterations += 1;   // panic past a limit: a cascade that never ends is a bug
}
```

- Synchronous, single-threaded, FIFO, so it's deterministic.
- Handles cascades (a liquidation's market order moves the mark price, which triggers another check) without contexts calling each other.
- Examples: `MarkPriceUpdated → risk checks drawdown`; `DrawdownBreached → run ends with RunLost`; `QuarterBoundary → run evaluates quota`.

### Commands, events, queries

| | Direction | Examples |
| --- | --- | --- |
| Commands | adapter → app | `PlaceOrder`, `Cancel`, `Advance`, `PickUpgrade` |
| Integration events | app → adapter | `PlayerFill`, `OrderRejected`, `Candle`, `QuarterEnded`, `RunLost` — each with `seq` |
| Queries | adapter → app → view | `BookDepth { levels }`, `Portfolio`, `Candles { from }`, `OpenOrders` |

Domain events (every agent fill, every posting) stay inside `app`.

## Enforcing the boundaries

Write these down now; they're cheap and they keep the design honest.

- **Cargo dependencies** enforce the context map. A context crate that lists another context crate in `Cargo.toml` is a review failure.
- **clippy.toml** in domain crates:
  - `disallowed_types`: `std::collections::HashMap`, `HashSet` (except where explicitly allowed for lookups)
  - `disallowed_methods`: `f64::ln`, `f64::exp`, `f64::powf`, ... (use `libm`), `std::time::*`, `rand::thread_rng`
- `#![forbid(unsafe_code)]` in every domain crate.
- `overflow-checks = true` in the release profile.
- **CI builds domain crates for a native target** (roadmap rule 1) **and** runs the cross-target determinism check (1.3).

## Test strategy by layer

| Layer | Tests |
| --- | --- |
| kernel | Rounding table tests; notional overflow; grid validation |
| exchange | Unit tests (roadmap list); proptest: book never crossed, filled ≤ ordered, IDs unique |
| ledger | Every entry balances; normal signs; `Reserved == Σ open reservations`; conservation across checkpoints |
| app | Whole-session invariants after random command sequences: money conserved across *all* accounts including agents and house; settlement never fails |
| determinism | Golden hashes per seed; `Advance(n) == n × Advance(1)`; native vs WASM hashes match; save → load → continue equals uninterrupted run |
| sim | Market health checks per seed: no flatline, no runaway, spread within bounds |

## Adoption order

Don't build the whole structure in week one. Each phase adds the crates it needs:

| Phase | Crates |
| --- | --- |
| 1 | `kernel`, `exchange` |
| 2 | `ledger`, and `app` with just the place/cancel use case to prove orchestration |
| 3 | `sim`, `market_data`, `cli`, golden hashes, RNG streams |
| 4 | `protocol`, `wasm` |
| 5–6 | `risk` (drawdown), `run` |
| 7 | `risk` grows margin and liquidation; ledger grows `Borrowed`/`Collateral` |
| 9 | `server` |

## Decisions to make now vs later

| Decide now (Phase 0–1 spec) | Can wait |
| --- | --- |
| Units and scales per instrument; the one notional/rounding function | Book internals (VecDeque vs slab) |
| Rounding policy (against the initiator, dust to house) | Journal checkpoint frequency |
| Market buys by quote amount; reserve max fee | Margin/collateral model details |
| Settlement can't fail; fills panic on ledger error | Snapshot saves |
| Self-trade prevention rule; time in force set | Price band percentages |
| Integers for money; `libm` for any float maths | Agent behaviour models |
| Per-agent RNG streams; pinned `rand`; `SIM_VERSION` | Server transport details |
| Domain types without serde; separate protocol crate | |
| `seq` on events, `request_id` on commands | |
| Mark price, not last trade, for all risk checks | |

Each "decide now" row is a decision log entry.
