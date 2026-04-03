# Market Making System — Development Roadmap

## Status Overview
| Status | Count |
|--------|-------|
| Not Started | 28 |
| In Progress | 0 |
| Done | 0 |

## Phase 1: Core Pricing & Data Infrastructure
Implement MarketDataFeed (Binance WebSocket + Polymarket CLOB integration), BayesianProbabilityUpdater, and StoikovPricingEngine. Validate that the system can ingest real-time price data, maintain an internal probability estimate, and compute Stoikov reservation prices correctly. Output: a running system that logs computed bid/ask prices in real-time without placing orders.

**Duration:** 3-5 days

| # | Item | Status | Depends On | Target Files |
|---|------|--------|------------|-------------|
| 1.1 | MarketDataFeed module with <50ms latency from Binance tick to internal state | Not Started | — | src/feeds/ |
| 1.2 | BayesianProbabilityUpdater with configurable prior, likelihood model, and update frequency | Not Started | 1.1 | src/pricing/ |
| 1.3 | StoikovPricingEngine computing reservation price r = s - q*gamma*sigma^2*(T-t) | Not Started | 1.1, 1.2 | src/pricing/ |
| 1.4 | Unit tests validating pricing output against known scenarios | Not Started | 1.3 | tests/ |
| 1.5 | Logging infrastructure for all computed prices and probability updates | Not Started | 1.1 | src/config/ |
| 1.6 | Latency profiling: instrument per-component processing time (data ingestion, Bayesian update, Stoikov computation, order serialization, API round-trip) to establish baseline latency budget and identify bottlenecks | Not Started | 1.1, 1.3 | src/feeds/, src/pricing/ |

## Phase 2: Order Management & Ladder Deployment
Implement BidLadderManager, NonLinearCurveDistributor, and KellySizer. Connect to Polymarket CLOB API for order placement. Deploy bid ladders on both YES and NO sides with non-linear volume distribution. Validate FIFO queue priority by measuring order placement latency at contract window open.

**Duration:** 3-5 days

| # | Item | Status | Depends On | Target Files |
|---|------|--------|------------|-------------|
| 2.1 | BidLadderManager with 1-cent spaced ladders, 10-15 levels deep on each side | Not Started | 1.3 | src/orders/ |
| 2.2 | NonLinearCurveDistributor with power-law (exponent 1.5) volume allocation | Not Started | 2.1 | src/orders/ |
| 2.3 | KellySizer enforcing quarter-Kelly with 8% max single position cap | Not Started | 1.2 | src/tracking/ |
| 2.4 | Order lifecycle management: place, amend, cancel with <100ms execution | Not Started | 2.1 | src/orders/ |
| 2.5 | Paper-mode order simulation for validation without real capital | Not Started | 2.1, 2.2, 2.3, 2.4 | src/orders/ |

## Phase 3: Inventory & Position Merge System
Implement InventoryManager with skew tracking and quote-skewing logic. Build PositionMergeEngine for automatic capital recycling. Implement SpreadCaptureTracker for real-time profitability monitoring. Validate that the system maintains market-neutral inventory and successfully merges opposing positions.

**Duration:** 3-4 days

| # | Item | Status | Depends On | Target Files |
|---|------|--------|------------|-------------|
| 3.1 | InventoryManager with soft (0.15) and hard (0.40) skew thresholds | Not Started | 1.3 | src/inventory/ |
| 3.2 | Quote-skewing logic: widen overweight side, tighten underweight side | Not Started | 3.1, 1.3 | src/inventory/, src/pricing/ |
| 3.3 | PositionMergeEngine: auto-merge when combined cost <= $1.02, batch size >= 10 | Not Started | 3.1 | src/inventory/ |
| 3.4 | SpreadCaptureTracker with rolling 1h and 24h cumulative metrics | Not Started | 2.4 | src/tracking/ |
| 3.5 | Integration test: full cycle of ladder placement -> fill -> inventory update -> merge -> capital recycle | Not Started | 3.1, 3.3, 2.1 | tests/ |

## Phase 4: Risk Management & Multi-Window Orchestration
Implement RiskController with all circuit breakers and kill switches. Build ContractWindowScheduler for managing concurrent contract lifecycles. Add Telegram notification integration. Validate graceful handling of drawdown events, contract expiry, and system failures.

**Duration:** 2-3 days

| # | Item | Status | Depends On | Target Files |
|---|------|--------|------------|-------------|
| 4.1 | RiskController: daily loss limit (-20%), drawdown kill switch (-40%), per-market caps (15%) | Not Started | 3.1 | src/risk/ |
| 4.2 | ContractWindowScheduler: handle 5-min and 15-min BTC/ETH windows concurrently | Not Started | 2.1 | src/scheduler/ |
| 4.3 | Telegram Bot integration for real-time alerts and daily summaries | Not Started | 4.1 | src/risk/ |
| 4.4 | Graceful wind-down: cancel all orders 30s before contract expiry | Not Started | 4.2, 2.1 | src/scheduler/, src/orders/ |
| 4.5 | Fault recovery: reconnect WebSocket feeds, resume ladder state after disconnection | Not Started | 1.1, 2.1 | src/feeds/, src/orders/ |
| 4.6 | Graceful shutdown handler: on SIGTERM/SIGINT, cancel all open orders across all active markets within 5 seconds, persist full inventory and order state to disk, log final position snapshot, and exit cleanly. On restart, detect and load persisted state before resuming trading | Not Started | 4.1, 4.2 | src/risk/, src/scheduler/ |
| 4.7 | State persistence layer: persist open orders, current inventory by market, running PnL, and merge history after every fill event and every 10s heartbeat. Use SQLite for initial deployment. On startup, load latest snapshot, reconcile with live Polymarket position query, flag discrepancies before resuming | Not Started | 3.1, 2.4 | src/config/ |

## Phase 5: Paper Trading Validation & Live Deployment
Run the complete system in paper-trade mode for minimum 7 days, targeting 200+ completed trades. Validate spread capture metrics, inventory neutrality, merge efficiency, and effective return. Compare paper results against expected 2-5% monthly return target. If paper win rate consistently above 60% and spread capture positive, proceed to live deployment with $1-5 per trade sizing. Scale gradually based on evidence.

**Duration:** 7-14 days (paper) + ongoing (live)

| # | Item | Status | Depends On | Target Files |
|---|------|--------|------------|-------------|
| 5.1 | 7+ days paper trading data with 200+ completed trade cycles | Not Started | 2.5, 4.2 | — |
| 5.2 | Performance report: realized spread per trade, daily PnL, inventory skew history, merge count and cost | Not Started | 5.1 | src/tracking/ |
| 5.3 | Live deployment with $1-5 initial trade size, scaling to full Kelly sizing over 2-4 weeks | Not Started | 5.1, 5.2 | — |
| 5.4 | Monitoring dashboard: real-time PnL, inventory state, spread capture rate, system health | Not Started | 3.4, 4.1 | src/tracking/ |
| 5.5 | Operational runbook: startup, shutdown, parameter tuning, emergency procedures | Not Started | 4.6 | — |
