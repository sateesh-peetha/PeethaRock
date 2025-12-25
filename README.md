## SYSTEM VISION

Design and implement a **distributed, modular, low-latency, event-driven trading platform** where each functional capability runs as an **independent Node.js service**, communicating through **WebSockets (real-time transport)** and **durable event logs (source of truth)**.  
The platform is deployable across **multiple personal laptops**, supports **full replayability**, **deterministic backtesting**, **safe machine learning integration**, and a **centralized control-plane UI**.

The system must be **robust against crashes, restarts, partial failures, data loss, and logic errors**, while remaining **simple enough to operate without cloud infrastructure, Kafka, or heavy databases**.

---

## GLOBAL NON-NEGOTIABLE ARCHITECTURAL PRINCIPLES

- Every module runs as an **independent Node.js process**
- All inter-module communication is **event-driven**
- **WebSockets = transport**, **Durable Event Logs = source of truth**
- No shared memory and **no direct module-to-module calls**
- Every event is **immutable, schema-validated, and append-only**
- Any module may crash and **fully recover via replay**
- **Live trading, backtesting, and replay share the same code paths**
- Machine learning is **advisory only** and **never executes trades directly**
- Capital protection is **centralized and non-bypassable**
- The entire system is **observable and controllable from a single UI**


# Phase-Wise Detailed Technical Execution Plan  
**Distributed, Event-Driven Trading Platform**

This document is a step-by-step **implementation manual** for building a distributed, event-driven trading system. Each phase defines **clear functional objectives**, **concrete technical steps**, and **solid test cases** (functional, failure, recovery, and correctness). The plan is designed for multi-laptop deployment using independent Node.js services, WebSockets, and durable event logs.

---

## PHASE 0 — SYSTEM BASELINE & STANDARDS (MANDATORY BEFORE CODING)

### Functional Objectives
- Establish shared contracts and standards across all modules
- Prevent integration failures later
- Guarantee replayability, debuggability, and correctness

### Technical Steps
1. Define event topic namespaces:  
   `md.*`, `indicator.*`, `strategy.*`, `risk.*`, `instrument.*`, `order.*`, `pnl.*`, `ml.*`, `system.*`
2. Define the global event envelope schema (Zod / JSON Schema)
3. Define versioning rules:
   - Event schema version
   - Strategy version
   - ML model version
4. Define directory conventions:
   - `/schemas`
   - `/events`
   - `/logs`
   - `/replay`
   - `/snapshots`
5. Define service lifecycle:
   - Start → Replay → Live subscription
6. Define hard constraints:
   - Strategy cannot send orders
   - Order engine only accepts `risk.approved`
   - ML cannot execute trades

### Test Cases
- Invalid schema → rejected
- Backward-compatible schema → accepted
- Version mismatch → logged, not fatal
- Service starts empty and rebuilds fully from replay

---

## PHASE 1 — CORE INFRASTRUCTURE (SYSTEM SPINE)

### Functional Objectives
- Zero data loss
- Deterministic ordering
- Safe distributed communication
- Independent service restartability

---

### 1.1 Message Router Service

#### Technical Steps
1. Implement WebSocket server
2. Implement topic-based pub/sub
3. Maintain subscriber registry
4. Validate incoming events against schema
5. Fan out events to subscribers
6. Emit heartbeats and latency metrics
7. Detect and signal backpressure
8. Ensure stateless, restart-safe design

#### Test Cases
**Functional**
- Publish event → all subscribers receive
- Topic isolation works correctly

**Failure**
- Router crash → restart → clients reconnect automatically
- Slow consumer does not block others

**Performance**
- Sustained high event rate
- Latency remains within acceptable bounds

---

### 1.2 Durable Event Log Service

#### Technical Steps
1. Implement append-only JSONL writer
2. Rotate files by topic and date
3. Ensure atomic writes (fsync)
4. Maintain offset indexes
5. Implement replay API (by offset or timestamp)
6. Add optional compression for old logs

#### Test Cases
- Kill process mid-write → no file corruption
- Replay reproduces exact event sequence
- Partial replay works correctly
- Log rotation and retention verified

---

### 1.3 Time Authority Service

#### Technical Steps
1. Implement LIVE mode (system clock)
2. Implement BACKTEST mode (simulated clock)
3. Implement REPLAY mode (event_time driven)
4. Provide REST/WS time query interface
5. Enforce monotonic time guarantees

#### Test Cases
- Time never moves backward
- Backtest clock advances deterministically
- Replay time matches event timestamps

---

## PHASE 2 — MARKET DATA TRUTH LAYER

### Functional Objectives
- Single source of market truth
- Fully replayable raw data
- No indicator or strategy logic

---

### 2.1 Market Data Service

#### Technical Steps
1. Implement exchange WebSocket adapters
2. Normalize all ticks into a canonical schema
3. Detect missing sequences and gaps
4. Request snapshots on reconnect
5. Emit:
   - `md.tick`
   - `md.depth`
   - `md.option.tick`
   - `md.option.chain.snapshot`
6. Persist all emitted events to durable logs

#### Test Cases
**Functional**
- Correct normalization across exchanges
- Option chain completeness

**Failure**
- WS disconnect → reconnect → snapshot recovery
- Exchange rate-limit handling

**Replay**
- Historical replay matches live feed ordering

---

## PHASE 3 — INDICATOR COMPUTATION LAYER

### Functional Objectives
- Deterministic indicator values
- Rebuildable state
- No trading logic

---

### 3.1 Indicator Engine

#### Technical Steps
1. Subscribe to `md.*` topics
2. Maintain sliding window buffers per symbol
3. Implement pure indicator functions (RSI, EMA, ADX, etc.)
4. Version indicator outputs
5. Optionally persist minimal state checkpoints
6. Emit:
   - `indicator.rsi.v1`
   - `indicator.ema.v1`
   - `indicator.adx.v1`

#### Test Cases
**Correctness**
- Known data → known indicator output
- Boundary and window edge cases

**Recovery**
- Restart → replay → identical indicator values

**Performance**
- High-frequency tick handling without lag

---

## PHASE 4 — STRATEGY DECISION LAYER

### Functional Objectives
- Convert signals to intent only
- No symbol, exchange, or order logic
- Hot-reloadable strategies

---

### 4.1 Strategy Engine

#### Technical Steps
1. Implement YAML/JSON strategy DSL
2. Validate and load strategy configs
3. Subscribe to indicator topics
4. Evaluate rules deterministically
5. Emit `strategy.intent`
6. Implement per-strategy rate limiting
7. Implement strategy-level kill switch

#### Test Cases
**Functional**
- Intent emitted only when conditions are met
- Multiple strategies operate independently

**Safety**
- Kill switch halts immediately
- Rate limiting prevents over-trading

**Replay**
- Same replay input → same intents

---

## PHASE 5 — CAPITAL SAFETY & RISK CONTROL

### Functional Objectives
- Absolute capital protection
- Centralized risk authority
- No bypass possible

---

### 5.1 Risk Gate (Critical)

#### Technical Steps
1. Subscribe to `strategy.intent`
2. Load current exposure and PnL
3. Validate:
   - Available margin
   - Position limits
   - Drawdown thresholds
4. Emit `risk.approved` or `risk.rejected`
5. Enforce global trading halt when required

#### Test Cases
**Functional**
- Valid trade → approved
- Invalid trade → rejected

**Failure**
- Risk service restart → rebuild from replay
- Race-condition handling under load

---

### 5.2 PnL & Risk Engine

#### Technical Steps
1. Subscribe to `order.fill`
2. Perform mark-to-market using `md.tick`
3. Maintain per-strategy and global PnL
4. Persist periodic snapshots
5. Emit `pnl.update` and `risk.violation`

#### Test Cases
- Correct realized/unrealized PnL
- Restart recovery accuracy
- Multi-instrument aggregation correctness

---

## PHASE 6 — INSTRUMENT RESOLUTION & EXECUTION

### Functional Objectives
- Correct instrument selection
- Accurate execution
- No ghost or orphan orders

---

### 6.1 Instrument Resolver

#### Technical Steps
1. Convert `strategy.intent` to tradable instruments
2. Apply ATM/OTM logic
3. Select expiry rules
4. Filter by liquidity
5. Emit `instrument.resolved`

#### Test Cases
- Correct strike and expiry selection
- Illiquid instrument rejection

---

### 6.2 Order Engine

#### Technical Steps
1. Subscribe to `risk.approved`
2. Generate idempotent order IDs
3. Send orders to exchange
4. Track full order lifecycle
5. Reconcile open orders on restart
6. Enforce kill switches

#### Test Cases
**Functional**
- Partial fills handled correctly
- Exchange rejects processed properly

**Failure**
- Restart with open orders
- Exchange outage handling

---

## PHASE 7 — BACKTESTING ENGINE (LIVE PARITY)

### Functional Objectives
- Same logic as live trading
- Deterministic outcomes
- Trustworthy statistics

---

### 7.1 Replay Engine

#### Technical Steps
1. Load historical event logs
2. Control Time Authority
3. Feed events in deterministic order
4. Support pause, resume, and seek

#### Test Cases
- Same input → same output
- Deterministic ordering guaranteed

---

### 7.2 Simulated Exchange

#### Technical Steps
1. Implement order queue model
2. Inject latency
3. Apply slippage models
4. Simulate partial fills

#### Test Cases
- Realistic fill behavior
- Sensitivity to latency and slippage

---

## PHASE 8 — MACHINE LEARNING SYSTEM (SAFE BY DESIGN)

### Functional Objectives
- Learn from historical data
- Advise strategies safely
- Enable long-term evaluation

---

### 8.1 Feature Store & Training

#### Technical Steps
1. Extract features from event logs
2. Store versioned feature datasets
3. Label outcomes
4. Train models offline
5. Persist models with metadata

#### Test Cases
- Feature consistency across replays
- Model reproducibility

---

### 8.2 ML Inference Engine

#### Technical Steps
1. Load trained model
2. Score incoming events
3. Emit advisory signals only
4. Enforce confidence thresholds
5. Implement ML kill switch

#### Test Cases
- ML disabled → no trading impact
- Advisory-only enforcement
- Version rollback safety

---

## PHASE 9 — UNIFIED CONTROL PLANE UI

### Functional Objectives
- Single operational view
- Safe manual intervention
- Complete observability

---

### Technical Steps
1. Build WebSocket aggregation layer
2. Implement system health dashboard
3. Strategy management UI
4. Risk and PnL visualization
5. Order lifecycle monitoring
6. Backtesting controls
7. ML insights and regime analysis panels

### Test Cases
- UI reflects real-time system state
- Manual kill switch propagates instantly
- Historical replay visualized correctly

---

## FINAL GUARANTEES

- Any module can restart at any time
- Zero silent data loss
- Backtesting equals live behavior
- ML cannot cause financial damage
- Scales from personal laptops to cloud without redesign

This plan is implementation-ready, audit-safe, and institution-grade.
