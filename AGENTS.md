# AGENTS.md — Market Making System

## Overview
Non-directional market-making system for Polymarket binary contracts (crypto UP/DOWN). Profits from liquidity provision, spread capture, and structural order-book mechanics.

## Tech Stack
- Python 3.11+ with asyncio + aiohttp
- websockets library — Binance/Coinbase real-time price feed ingestion
- py_clob_client — Polymarket CLOB API client for order management
- numpy/scipy — statistical computations (volatility, Bayesian updates, curve fitting)
- web3.py — Polygon on-chain interaction as fallback for position merging
- python-telegram-bot — alert and notification delivery
- SQLite or Redis — local state persistence for inventory, orders, and performance metrics
- Docker — containerized deployment for 24/7 uptime
- Claude API (anthropic SDK) — optional structural mispricing analysis layer
- Pydantic — configuration and data validation
- structlog — structured logging

## Architecture
| Component | Description |
|-----------|-------------|
| StoikovPricingEngine | Computes reservation price and optimal spread using Stoikov-Avellaneda model with inventory, volatility, and time-to-expiry inputs |
| BayesianProbabilityUpdater | Maintains internal fair-value probability via Bayesian updates from CEX reference prices and order flow |
| BidLadderManager | Places and maintains laddered limit orders across multiple price levels on both YES and NO sides |
| NonLinearCurveDistributor | Allocates capital across ladder levels using power-law distribution, concentrating volume near the mid-price |
| InventoryManager | Tracks net YES/NO inventory, enforces skew limits, and feeds position state into the pricing engine |
| PositionMergeEngine | Merges opposing YES+NO positions into $1.00 settlement to recycle locked capital back into the order book |
| SpreadCaptureTracker | Monitors and records every spread capture event with rolling profitability metrics |
| KellySizer | Computes risk-budget cap per ladder level using fractional Kelly criterion (0.25x) with hard position limits |
| RiskController | Top-level risk layer with daily loss limits, drawdown kill switches, per-market caps, and Telegram alerts |
| MarketDataFeed | Aggregates real-time data from Binance, Coinbase WebSockets and Polymarket CLOB for pricing and volatility |
| ContractWindowScheduler | Manages lifecycle of trading across multiple concurrent 5-min and 15-min contract windows |

## Data Flow
MarketDataFeed -> BayesianProbabilityUpdater -> StoikovPricingEngine -> NonLinearCurveDistributor -> KellySizer -> BidLadderManager -> (fills) -> InventoryManager + SpreadCaptureTracker -> PositionMergeEngine -> (capital recycled). RiskController monitors portfolio-level metrics and can halt the pipeline. ContractWindowScheduler orchestrates startup/shutdown across contract lifecycles.

## Conventions
- All source code in `src/` with absolute imports
- Tests in `tests/` mirroring src/ structure
- Config loaded from `config.yaml` via Pydantic Settings
- Async-first: all I/O operations use asyncio
- Logging via structlog
- Type hints required on all public functions
- Environment variables for secrets

## Directory Structure
```
src/
  pricing/        # StoikovPricingEngine, BayesianProbabilityUpdater
  orders/         # BidLadderManager, NonLinearCurveDistributor
  inventory/      # InventoryManager, PositionMergeEngine
  tracking/       # SpreadCaptureTracker, KellySizer
  risk/           # RiskController, kill switches
  feeds/          # MarketDataFeed (Binance, Coinbase, Polymarket)
  scheduler/      # ContractWindowScheduler
  config/         # Configuration loading
tests/
config.yaml
```
