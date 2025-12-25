# PeethaRock

# Distributed Event-Driven Trading Platform  
**Institution-Grade, Laptop-Deployable, Replayable, and Safe-by-Design**

---

## 📌 System Vision

Design and implement a **distributed, modular, low-latency, event-driven trading platform** where each capability runs as an **independent Node.js service**, communicating via **WebSockets** with a **durable event log** as the source of truth.  
The system is deployable across **multiple personal laptops**, supports **deterministic replay & backtesting**, integrates **machine learning safely (advisory only)**, and is fully observable and controllable from a **single UI control plane**.

---

## 🧱 Global Architectural Principles (Non-Negotiable)

1. Each module runs as an **independent Node.js process**
2. All communication is **event-driven**
3. **WebSockets = transport**, **Durable Event Logs = truth**
4. No shared memory, no direct service calls
5. Events are **immutable, append-only, schema-validated**
6. Any module can crash and recover via **replay**
7. Live trading, backtesting, and replay use the **same code paths**
8. ML is **advisory only**, never executes trades
9. Capital protection is **centralized and non-bypassable**
10. Everything is **observable and controllable from one UI**

---

## 📦 System-Wide Event Contract

All messages in the system must follow this immutable structure:

```json
{
  "event_id": "uuid-v7",
  "topic": "md.tick",
  "source": "market-data",
  "schema_version": "1.0",
  "event_time": 1735123456789,
  "ingest_time": 1735123456999,
  "correlation_id": "strategy_run_id",
  "payload": {}
}
