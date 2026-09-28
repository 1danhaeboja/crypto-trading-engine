# PROJECT_CONTEXT.md

## 1. Project Overview

**Project name:** `crypto-trading-engine`

This repository is a risk-first cryptocurrency trading research,
backtesting, paper-trading, and eventually live-trading platform.

The initial research capital is **KRW 3,000,000**.

The original aspirational target discussed was growing KRW 3,000,000 to
KRW 30,000,000 in roughly two months. This is **not** an engineering
performance requirement or an expected return. A 10x return over about
60 days would require approximately 3.9% compounded daily before fees,
slippage, taxes, and execution losses, and should be treated as an
extremely aggressive aspiration rather than a system assumption.

The engineering objective is instead:

1.  survive,
2.  avoid large drawdowns,
3.  trade only when evidence indicates positive expectancy,
4.  compound profits when conditions are favorable,
5.  remain in `WAIT/CASH` when conditions are unfavorable or uncertain.

Capital preservation takes precedence over trade frequency and headline
return.

------------------------------------------------------------------------

## 2. Core Philosophy

### 2.1 No single strategy is assumed to work in every market regime

The system must not permanently bind itself to one fixed trading
strategy.

Market behavior changes between:

-   strong uptrend,
-   mild uptrend,
-   sideways/range-bound conditions,
-   panic/crash,
-   crash recovery,
-   overheated/parabolic conditions,
-   abnormal or uncertain conditions.

The system should identify the current regime and choose an appropriate
strategy or choose not to trade.

### 2.2 `WAIT/CASH` is a first-class decision

No trade is a valid trading decision.

The architecture must represent `WAIT` or `CASH` explicitly rather than
treating inactivity as a missing signal.

A strategy must never be forced to produce BUY/SELL signals merely
because the engine is running.

### 2.3 Loss avoidance matters because recovery is asymmetric

Examples:

    Drawdown   Gain required to recover
  ---------- --------------------------
        -10%                     +11.1%
        -20%                     +25.0%
        -30%                     +42.9%
        -50%                      +100%
        -70%                      +233%

The engine should therefore optimize not only for return but also for
drawdown, downside volatility, losing streaks, and risk of ruin.

------------------------------------------------------------------------

## 3. High-Level Architecture

``` text
DATA SOURCES
     |
     v
Market Data Collectors
     |
     v
Normalization / Storage
     |
     v
Market Intelligence
(price, volume, orderbook, sentiment, events, derivatives)
     |
     v
Market Regime Classifier
     |
     +--------------------+
     |                    |
     v                    v
TRADE                 WAIT / CASH
     |
     v
Strategy Selector
     |
     v
Strategy Signal
     |
     v
Risk Engine
     |
     +------ reject / reduce / kill -----> WAIT / CASH
     |
     v
Execution Engine
     |
     +----------+----------+
     |                     |
     v                     v
   Bithumb                Upbit
     |
     v
Orders / Fills / Trade DB
     |
     v
Performance Analyzer
     |
     +---- feedback to regime / strategy evaluation
```

The **Risk Engine must sit between strategies and execution**. A
strategy signal alone must never be sufficient to place a live order.

------------------------------------------------------------------------

## 4. Planned Data Sources

### Phase 1: core/free market data

#### Bithumb

Primary Korean execution venue candidate.

Collect:

-   KRW tickers,
-   trades,
-   order books,
-   candles where useful,
-   account/order information only when authenticated operation is
    introduced.

#### Upbit

Second Korean venue and cross-exchange reference/execution candidate.

Collect:

-   KRW tickers,
-   trades,
-   order books,
-   candles where useful.

#### Binance or another large global venue

Initially intended mainly as a **global market and derivatives data
source**, not necessarily an execution venue.

Potential signals:

-   global BTC/alt prices,
-   funding rates,
-   open interest,
-   liquidation activity,
-   global/local basis.

Before implementing exchange-specific clients, verify the **current
official API documentation, authentication scheme, WebSocket format,
rate limits, and supported order types**. Do not rely on stale API
assumptions.

### Phase 2: sentiment and event data

Candidates include:

-   crypto news/RSS/API,
-   Fear & Greed-style indicators,
-   Reddit,
-   X,
-   Telegram,
-   Google Trends,
-   mention volume,
-   mention velocity,
-   sentiment velocity.

Raw sentiment level alone is not assumed to be predictive. Rate of
change and lead/lag relationships should be measured.

### Phase 3: on-chain data

Possible signals:

-   exchange inflow/outflow,
-   whale transfers,
-   stablecoin flows,
-   other network activity.

Potential providers include commercial on-chain analytics services. Paid
data should only be adopted after demonstrating incremental predictive
value.

------------------------------------------------------------------------

## 5. Market Intelligence

The intelligence layer converts raw market events into features.

Candidate features include:

-   returns,
-   realized volatility,
-   volatility z-score,
-   volume z-score,
-   market breadth,
-   order-book imbalance,
-   microprice,
-   trade imbalance,
-   spread,
-   depth,
-   expected fill price,
-   cross-exchange basis,
-   funding rate,
-   open-interest delta,
-   liquidation z-score,
-   sentiment level,
-   sentiment velocity,
-   news/event intensity.

Features should be timestamped consistently so that lead/lag analysis
and realistic backtesting are possible.

------------------------------------------------------------------------

## 6. Market Regime Classification

Initial regime examples:

-   `STRONG_UPTREND`
-   `MILD_UPTREND`
-   `SIDEWAYS`
-   `PANIC_CRASH`
-   `CAPITULATION`
-   `EARLY_RECOVERY`
-   `OVERHEATED`
-   `ABNORMAL`
-   `UNKNOWN`

A possible crash/recovery sequence:

``` text
BTC falls sharply
        ↓
market volume expands
        ↓
alts sell off
        ↓
sell intensity becomes extreme
        ↓
sell wall / sell pressure begins decreasing
        ↓
BTC stabilizes
        ↓
alt volume remains elevated
        ↓
CAPITULATION -> EARLY_RECOVERY candidate
```

Illustrative structured output:

``` text
BTC 1h return        -2.8%
BTC volume Z         +3.4
market breadth       12%
liquidations         extreme
funding              -0.018%
sentiment            strongly negative
orderbook imbalance  -0.41 -> -0.08
sell pressure        falling

Regime: CAPITULATION -> EARLY_RECOVERY
Confidence: 0.73
```

These numbers are examples only, not production thresholds.

### Classification approach

Start with a transparent rules-based classifier.

After sufficient historical data has been collected and
labeled/evaluated, experiment with statistical/ML models such as:

-   Hidden Markov Models,
-   XGBoost,
-   LightGBM,
-   other calibrated classifiers.

Do **not** let an LLM directly place trades.

LLMs may assist research, summarization, event classification, or
development, but production order decisions should remain deterministic,
testable, logged, and reproducible.

------------------------------------------------------------------------

## 7. Strategy Candidates

The architecture should support multiple independent strategies.

### 7.1 Cross-exchange arbitrage

Example:

``` text
Upbit XRP       3,000 KRW
Bithumb XRP     3,030 KRW
gross spread        1.0%
```

Net opportunity must account for:

``` text
net spread =
price difference
- trading fees on both venues
- expected slippage
- rebalancing cost
- execution risk allowance
```

Preferred operational model is **pre-funded inventory on both
exchanges** so both legs can execute almost simultaneously.

Avoid depending on transferring assets between exchanges after every
arbitrage trade.

Important execution risks:

-   one leg fills while the other does not,
-   partial fills,
-   API latency,
-   order-book movement,
-   insufficient depth,
-   rebalancing cost,
-   exchange/network interruptions.

Potential controls:

-   IOC/FOK when supported and appropriate,
-   depth-aware expected fill calculations,
-   hedge timeout,
-   maximum unhedged exposure,
-   kill switch.

No spoofing, wash trading, matched manipulation, fake liquidity, or
other manipulative behavior is permitted.

### 7.2 Panic crash recovery

Look for evidence that forced/panic selling is exhausting and
stabilization/recovery is beginning.

Possible inputs:

-   extreme negative return,
-   volume spike,
-   liquidation spike,
-   breadth collapse,
-   funding stress,
-   order-book recovery,
-   decreasing sell imbalance,
-   BTC stabilization,
-   persistent alt volume.

Avoid trying to catch a falling market solely because price has fallen
substantially.

### 7.3 Volume breakout / momentum

Candidate when price structure, participation, and volume expansion
align.

Must be regime-aware and risk-controlled to avoid chasing overheated
moves.

### 7.4 Mean reversion

Candidate primarily in range-bound or statistically stable regimes.

Should normally be disabled in strong trends or structural breaks.

### 7.5 Relative-strength / market-neutral strategies

Potential later research area.

Examples may include pairs or relative-strength structures designed to
reduce broad market beta.

### 7.6 WAIT/CASH

This is explicitly a strategy/state and should be measurable.

Track how much loss was avoided and opportunity cost incurred by staying
out of the market.

------------------------------------------------------------------------

## 8. Risk Engine

Risk management is a mandatory system boundary.

No strategy may bypass it.

Initial **research defaults** may include:

-   per-trade maximum planned loss: approximately 0.5--1.0% of equity,
-   daily loss stop: approximately 2%,
-   reduce exposure after a losing streak,
-   suspend new entries during abnormal volatility or degraded data,
-   pause strategies around approximately 10--15% maximum drawdown,
-   cap unhedged cross-exchange exposure,
-   reject orders when market data is stale,
-   reject orders when expected slippage exceeds limits,
-   global kill switch.

These are starting parameters for paper testing, not proven optimal
values.

All risk limits should be configurable, logged, and testable.

### Fail-closed principle

If configuration, market data, account state, risk state, or execution
state is uncertain, the engine should default to **not opening new
risk**.

------------------------------------------------------------------------

## 9. Execution Modes

The system should clearly separate:

``` text
BACKTEST
PAPER
LIVE
```

### Default mode

`PAPER`

Live trading must never be enabled implicitly.

The repository must initially operate without real exchange credentials.

### Live-mode safety requirements

Before LIVE is implemented/enabled:

-   explicit configuration is required,
-   tests must verify accidental LIVE startup is impossible,
-   API credentials must never be committed,
-   withdrawal permission should not be granted to trading bots unless
    absolutely unavoidable,
-   kill switch must exist,
-   position/exposure limits must exist,
-   stale-data protection must exist,
-   order/fill reconciliation must exist,
-   failure/restart behavior must be defined.

------------------------------------------------------------------------

## 10. Backtesting

Backtesting must model realistic execution.

### Arbitrage

Use an **event-driven order-book backtest**, not candle-only simulation.

Model:

-   bid/ask depth,
-   maker/taker fees,
-   expected queue/fill assumptions,
-   slippage,
-   partial fills,
-   latency assumptions,
-   failed second leg,
-   rebalancing costs.

### Other strategies

Use event/candle granularity appropriate to the strategy, while
preventing look-ahead bias and data leakage.

### Validation sequence

``` text
historical backtest
        ↓
out-of-sample test
        ↓
paper trading with live data
        ↓
tiny real-capital experiment
        ↓
scale only if behavior remains valid
```

Do not scale because of a short profitable streak.

------------------------------------------------------------------------

## 11. Performance Metrics

Return alone is insufficient.

Track at least:

-   total return,
-   annualized/period-adjusted return where meaningful,
-   maximum drawdown,
-   Return / MDD,
-   Profit Factor,
-   expectancy per trade,
-   win rate,
-   average win,
-   average loss,
-   payoff ratio,
-   Sharpe ratio where statistically appropriate,
-   Sortino ratio,
-   downside deviation,
-   maximum consecutive losses,
-   turnover,
-   fees,
-   slippage,
-   strategy-level PnL,
-   regime-level PnL,
-   time spent in CASH,
-   risk-of-ruin estimates where practical.

Example report format:

``` text
Trades                 17
Win Rate             58.8%
Profit Factor         1.72
Return               +1.21%
Max Drawdown          -0.43%

Momentum             +0.8%
Recovery             +0.6%
Arbitrage            +0.2%
Mean Reversion       -0.4%

Market regime
09-12   SIDEWAYS
12-15   CRASH
15-18   RECOVERY
18-24   BULL
```

The values above are illustrative only and must never be presented as
expected performance.

------------------------------------------------------------------------

## 12. Storage Architecture

Proposed components:

### PostgreSQL / TimescaleDB

Normalized time-series and transactional records.

Potential entities:

-   raw market events,
-   normalized ticks,
-   order-book snapshots/deltas,
-   features,
-   regime outputs,
-   strategy signals,
-   risk decisions,
-   orders,
-   fills,
-   positions,
-   balances,
-   PnL,
-   system events.

### Redis

Optional current-state/cache layer for:

-   latest market state,
-   latest features,
-   regime state,
-   execution/risk state.

### Parquet

Raw or historical research datasets suitable for:

-   offline analysis,
-   reproducible backtests,
-   feature research.

Storage technology should remain replaceable through interfaces rather
than leaking directly into strategy code.

------------------------------------------------------------------------

## 13. Proposed Repository Structure

A modern Python `src/` layout is preferred.

``` text
crypto-trading-engine/
├── README.md
├── PROJECT_CONTEXT.md
├── AGENTS.md
├── pyproject.toml
├── .gitignore
├── .env.example
│
├── config/
│   └── strategy.yaml
│
├── src/
│   └── crypto_trading_engine/
│       ├── __init__.py
│       ├── __main__.py
│       ├── config/
│       ├── collectors/
│       │   ├── bithumb/
│       │   ├── upbit/
│       │   └── binance/
│       ├── data/
│       ├── intelligence/
│       │   ├── market_regime.py
│       │   ├── sentiment.py
│       │   ├── volume_anomaly.py
│       │   └── crash_detector.py
│       ├── strategies/
│       │   ├── wait_cash.py
│       │   ├── momentum.py
│       │   ├── recovery.py
│       │   ├── mean_reversion.py
│       │   └── arbitrage.py
│       ├── risk/
│       │   ├── engine.py
│       │   ├── position_size.py
│       │   ├── drawdown.py
│       │   └── kill_switch.py
│       ├── execution/
│       │   ├── paper.py
│       │   ├── bithumb.py
│       │   └── upbit.py
│       ├── backtest/
│       └── monitoring/
│
└── tests/
    ├── risk/
    ├── execution/
    └── strategies/
```

The exact structure may evolve. Prefer clear domain boundaries over
premature abstraction.

------------------------------------------------------------------------

## 14. Secrets and Security

Never commit secrets.

`.env.example` may contain names only:

``` dotenv
BITHUMB_API_KEY=
BITHUMB_API_SECRET=

UPBIT_ACCESS_KEY=
UPBIT_SECRET_KEY=

TRADING_MODE=PAPER
```

`.gitignore` must include at minimum:

``` text
.env
.venv/
__pycache__/
*.py[cod]
.pytest_cache/
.DS_Store
data/
logs/
```

Use local `.env`, OS secret storage, deployment secret managers, or
another appropriate secret mechanism.

Never request that API secrets be pasted into ChatGPT/Codex conversation
history.

Start exchange API access read-only where possible. Add trading
permission only after paper validation. Avoid withdrawal permission.

------------------------------------------------------------------------

## 15. Development Workflow

GitHub repository:

`1danhaeboja/crypto-trading-engine`

Suggested local macOS path:

``` text
~/projects/crypto-trading-engine
```

Suggested future server deployment path:

``` text
/opt/crypto-trading-engine
```

GitHub is the source of truth.

Prefer:

``` text
feature branch
    ↓
implementation
    ↓
tests
    ↓
review diff
    ↓
commit
    ↓
PR / merge
```

Avoid committing directly to `main` for normal feature work after
bootstrap.

------------------------------------------------------------------------

## 16. Development Roadmap

### v0.1 --- Foundation & Safety

Build:

-   Python project structure,
-   typed configuration,
-   PAPER default,
-   explicit execution modes,
-   core domain models,
-   Risk Engine skeleton,
-   kill switch,
-   paper execution skeleton,
-   structured logging,
-   `.env.example`,
-   `.gitignore`,
-   unit tests,
-   accidental-LIVE-start protection.

No real order placement.

### v0.2 --- Market Data & Intelligence

Build:

-   current official Bithumb collector,
-   current official Upbit collector,
-   global/Binance data collector where useful,
-   normalized event schema,
-   raw data persistence,
-   feature calculations,
-   initial rules-based regime classifier,
-   data-quality/staleness checks.

### v0.3 --- Strategy Lab & Backtesting

Build:

-   WAIT/CASH,
-   arbitrage research,
-   panic-recovery research,
-   momentum/breakout,
-   mean reversion,
-   strategy selection,
-   realistic backtesting,
-   fee/slippage modeling,
-   performance reports.

### v0.4 --- Live Paper Validation

Run continuously against live data without real orders.

Evaluate:

-   strategy expectancy,
-   drawdowns,
-   regime performance,
-   signal lead/lag,
-   data outages,
-   execution simulation quality,
-   opportunity frequency,
-   transaction-cost sensitivity.

### v0.5+ --- Controlled Live Trading

Only after explicit human approval and successful validation.

Start with tiny capital.

Do not automatically deploy the entire KRW 3,000,000.

Scaling must be evidence-driven.

------------------------------------------------------------------------

## 17. Initial Arbitrage Experiment

One early experiment is to monitor Bithumb and Upbit simultaneously
using live order books.

With virtual KRW 3,000,000 capital, measure:

-   gross cross-exchange spreads,
-   depth-adjusted spreads,
-   fee-adjusted spreads,
-   slippage-adjusted spreads,
-   opportunities above 0.3%,
-   opportunities above 0.5%,
-   opportunities above 1.0%,
-   opportunity duration,
-   fillable size,
-   one-leg execution risk.

Collect at least enough live observations to understand whether apparent
spreads survive realistic costs.

Compare the resulting opportunity quality with recovery, momentum, and
sentiment/regime strategies.

Thresholds above are research buckets, not automatic trade triggers.

------------------------------------------------------------------------

## 18. Signal Research Principles

Do not assume a signal works because it is intuitively plausible.

For each candidate feature, measure:

-   predictive horizon,
-   lead/lag,
-   stability over time,
-   behavior by market regime,
-   transaction-cost sensitivity,
-   false-positive rate.

Useful horizons may include, depending on data frequency:

``` text
30 seconds
2 minutes
5 minutes
15 minutes
1 hour
```

These are research horizons, not fixed production intervals.

Avoid feature leakage and optimize on out-of-sample performance rather
than in-sample fit.

------------------------------------------------------------------------

## 19. Observability

Every meaningful decision should be reconstructable.

Log:

-   input market timestamp,
-   feature snapshot/version,
-   regime and confidence,
-   strategy selected,
-   strategy signal,
-   risk-engine decision,
-   rejected-order reason,
-   intended order,
-   actual/simulated fill,
-   fees/slippage,
-   position before/after,
-   PnL,
-   kill-switch events,
-   configuration version.

A future incident should be explainable without guessing why the engine
traded.

------------------------------------------------------------------------

## 20. Legal / Operational Boundaries

The project is intended for proprietary trading using the user's own
capital.

Do not implement:

-   spoofing,
-   wash trading,
-   matched manipulation,
-   artificial volume generation,
-   intentional market manipulation,
-   circumvention of exchange identity/Travel Rule controls.

If the project later expands to managing other people's money, pooled
funds, investment advice, or related services, legal/regulatory
requirements must be reviewed separately before implementation.

Current Korean exchange, tax, Travel Rule, and virtual-asset regulations
can change. Verify current official rules before relying on them
operationally.

------------------------------------------------------------------------

## 21. Decisions Not Yet Finalized

Codex must **not invent final answers** for these without
research/review:

-   exact Bithumb API implementation,
-   exact Upbit API implementation,
-   exact Binance/global-data endpoints,
-   database deployment topology,
-   final strategy thresholds,
-   final position-sizing formula,
-   final stop-loss methodology,
-   exact arbitrage minimum spread,
-   exact regime classifier,
-   ML model selection,
-   live-server architecture,
-   monitoring/alerting provider,
-   final universe of tradable coins,
-   live capital allocation,
-   tax/accounting implementation.

Prefer small experiments and measured evidence.

------------------------------------------------------------------------

## 22. Instructions for Codex

When continuing development from this repository:

1.  Read `PROJECT_CONTEXT.md` before proposing architecture changes.
2.  Inspect the existing repository and Git history before modifying
    files.
3.  Keep `PAPER` as the default.
4.  Never add a real API secret to the repository.
5.  Never enable withdrawal capability by default.
6.  Never implement a path where a strategy can bypass the Risk Engine.
7.  Treat `WAIT/CASH` as a legitimate output.
8.  Fail closed when state or data is uncertain.
9.  Add tests for safety-critical behavior before live functionality.
10. Verify current official exchange API documentation before
    implementing exchange-specific behavior.
11. Separate research assumptions from production defaults.
12. Do not describe backtest performance as expected future return.
13. Keep execution decisions deterministic and auditable.
14. Prefer incremental, reviewable commits.
15. Before any LIVE implementation, present the safety changes and
    request explicit human review.

------------------------------------------------------------------------

## 23. First Codex Task

The recommended first task is:

``` text
Read PROJECT_CONTEXT.md and inspect the repository.

Create or update AGENTS.md so future Codex sessions preserve the project's
risk-first engineering rules.

Then implement v0.1 Foundation & Safety only:

- modern Python src layout
- configuration models
- BACKTEST/PAPER/LIVE execution-mode enum
- PAPER as mandatory default
- fail-closed live-mode guard
- basic domain models
- Risk Engine skeleton
- kill switch
- paper execution skeleton
- structured logging
- .env.example
- .gitignore
- unit tests for safety-critical behavior

Do not implement real exchange order placement yet.
Do not add secrets.
Run the tests and report the results before proposing the next phase.
```

------------------------------------------------------------------------

## 24. Guiding Principle

> **The system is not required to trade. It is required to protect
> capital and take risk only when the evidence and risk constraints
> justify doing so.**

Profitability must be demonstrated through data, realistic execution
modeling, out-of-sample testing, and live paper validation rather than
assumed in advance.
