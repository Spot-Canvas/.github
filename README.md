# Spot Canvas Trading — Architecture Overview

```
                        ┌─────────────────────────────────────────────┐
                        │             spot-canvas-app                 │
                        │                                             │
                        │   Ingestion ──── Backtesting                │
                        │       │                                     │        ╭──────────╮
                        │       ▼                                     │◄──────►│  Price   │
                        │   Trading Strategies                        │        │   Data   │
                        │       │                                     │        │   (DB)   │
                        │       │          Chart ──────►  Chart UI    │        ╰──────────╯
                        └───────┼─────────────────────────────────────┘
                                │
                                │ signals
                                ▼
                        ┌───────────────┐        trades        ┌──────────────┐
                        │               │─────────────────────►│              │
                        │     NATS      │                      │    Ledger    │
                        │  (JetStream)  │                      │  (positions, │
                        │               │◄╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌╌│   P&L, API)  │
                        └───────────────┘  account data (REST) └──────────────┘
                          │           ▲                              ▲
                 signals  │           │ trades                       │
                          ▼           │                              │
                        ┌───────────────┐      account data (REST)   │
                        │               │────────────────────────────╯
                        │   Trading Bot │
                        │               │
                        └───────┬───────┘
                                │
                                │ reports
                                ▼
                        ┌───────────────┐
                        │   Dashboard   │
                        │     (S3)      │
                        └───────────────┘
```

## Components

| Component | Repo | Description |
|---|---|---|
| **spot-canvas-app** | `spot-canvas-app/` | Ingestion from exchanges (Coinbase, Kraken, OKX, CoinDesk), backtesting engine, trading strategy signals, chart data serving |
| **Chart UI** | `spot-canvas-app/` | Web UI for candlestick charts, indicators, and strategy signals |
| **NATS (JetStream)** | Synadia NGS | Message broker — carries trading signals and trade events between services |
| **Ledger** | `ledger/` | Records trades, tracks positions, computes P&L. REST API for portfolio queries. |
| **Trading Bot** | TBD | Receives signals via NATS, executes trades, publishes trade events back to NATS, queries Ledger for position/account state |
| **Price Data (DB)** | Cloud SQL | PostgreSQL — candles, indicators, strategy signals, and ledger tables |
| **Dashboard** | S3 | Static reporting dashboard |

## Data Flow

1. **spot-canvas-app** ingests price data from exchanges and stores candles in the database
2. Trading strategies run on the ingested data and publish **signals** to NATS
3. The **Trading Bot** subscribes to signals, makes trading decisions, and executes orders
4. After execution, the bot publishes **trade events** to NATS (`ledger.trades.<account>.<market>`)
5. The **Ledger** consumes trade events, updates positions and P&L
6. The bot queries the **Ledger REST API** for current positions and account state
7. The bot generates **reports** to the Dashboard
