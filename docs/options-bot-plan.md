# Automated Options Trading Bot — Plan

Status: **plan only, no code yet.** Prepared 2026-10-10. Broker fees and data prices below were checked on that date but change often. Check each one again before you open an account.

---

## 0. The facts this plan is built around

Start here, because these numbers drive every other decision.

> **Update (2026-10-10): starting capital is now $1,000, and paying for real-time data is approved.** The constraints below were written for $100. Section 11 covers what changes at $1,000 and answers the question "can we just reverse a losing algorithm?"

1. **$100 buys roughly one options contract.** A contract covers 100 shares, so a $0.60 quote costs $60. Each trade will use 30–90% of the account. That means the risk of losing everything is high whatever the strategy. The plan therefore puts survival ahead of upside.
2. **At $100, fixed monthly costs matter more than commissions.** Commissions are close to $0 at several brokers. Real-time options data, hosting and LLM calls are not. A $99/month data feed would cost 99% of the capital every month. The first job is to keep fixed costs near $0–5/month.
3. **The account will be a cash account, which allows only long calls and long puts (Level 2).**
   - Covered calls are out, because they need 100 shares.
   - Cash-secured puts are out, because they need strike × 100 in cash.
   - Spreads (Level 3) usually need a margin account, and margin accounts need at least $2,000.
   - The PDT $25k rule was removed (SEC approved April 2026; brokers have until Oct 2027 to implement). It never applied to cash accounts anyway. Cash accounts trade only with **settled funds**, and options settle T+1.
4. **Buying options loses money for the average trader.** You pay the bid/ask spread on every trade and lose time value every day. Most bots that buy options lose money. The realistic goal is a system that can show positive expectancy in paper trading before real money is risked, then grows slowly. A goal of "10x the $100" is not realistic.
5. **A long option that finishes in the money is auto-exercised.** A cash account with $100 cannot pay for 100 shares. The bot must **close every position before expiry**, with no exceptions.
6. **Good-faith violations.** Suppose the bot buys with money from a sale that hasn't settled yet, then sells that new position before the money settles. That is a good-faith violation. Three violations in 12 months gets the account restricted for 90 days. With $100 that means **about one round trip per settlement cycle**. The bot must keep its own count of settled cash, because the broker's "buying power" may include unsettled money.
7. **The break-even hurdle.** Fixed costs of about $3–9/month are 3–9% of capital per month. The strategy must earn roughly **50–100% a year just to break even**. Treat the live $100 as a **measurement budget you expect to lose**, not as growth capital.

---

## 1. Broker selection

Our profile: about 5–15 round trips per month, 1 contract each, API access required.

| Broker | Options commission | API quality | Real-time options data **via API** | Verdict |
|---|---|---|---|---|
| **Alpaca** | $0 commission; about $0.05/contract in regulatory/OCC fees | Best: REST + WebSocket, `alpaca-py`, free paper trading, multi-leg (`mleg`) supported | The free tier is an **"indicative" feed** (a randomized derivative of OPRA, not suitable for live pricing). Real OPRA data costs **$99/month** (Algo Trader Plus) | **Use for development and paper trading.** Live trading only if you accept the data problem (see below) |
| **Tradier** | $0.35/contract on Standard; $0 on Pro ($10/month) | Good REST + streaming, sandbox | Data is free for funded accounts. Reports say options quotes may be **delayed** unless subscribed. **Must verify** | Strong live candidate if real-time options quotes are free for funded accounts |
| **tastytrade** | $1/contract to open, $0 to close, plus about $0.10/contract clearing each way | Open API, Python SDKs exist | Real-time streamer for account holders (**verify**) | Good live fallback |
| **Webull** | $0 | OpenAPI Python SDK (application required) | API needs a **separate OPRA non-display subscription**; price not published | Possible, but data cost is unknown |
| Schwab / IBKR | about $0.65/contract | Schwab Trader API / IBKR TWS | Schwab is likely included; IBKR has paid data packages | More expensive per trade, still viable |
| Robinhood | $0 | **No official options API** | — | **Excluded.** Scraping its API breaks its terms of service |

**Cost formula for choosing a live broker:**
`monthly_cost = data_fee + platform_fee + round_trips × contracts × 2 × per_contract_fee`

At 8 round trips per month:

| Option | Monthly cost |
|---|---|
| Tradier Standard | ≈ $5.60 (if options data is free) |
| tastytrade | ≈ $9.60 |
| Schwab | ≈ $10.40 |
| Alpaca + OPRA | ≈ $99 |
| Alpaca, indicative data only | ≈ $0.80, but pricing blind |

**Recommendation:**
1. Open **Alpaca** now. It's free, and its paper trading is unlimited.
2. Build everything against a `Broker` interface, so changing brokers means adding one adapter.
3. Before going live, confirm Tradier's real-time options data terms. If the data is free for a funded account, go live on **Tradier Standard**. Otherwise use **tastytrade**.
4. Only use Alpaca live if the strategy sticks to SPY/QQQ, where spreads are a penny wide, and limit-order logic does not depend on exact quotes. This option comes with the data caveat above.

---

## 2. Strategy plan (what the bot actually trades)

### Universe
Only underlyings with very liquid options: **SPY, QQQ, IWM**, plus perhaps 3–5 mega-cap stocks.
- Bid/ask spread must be ≤ $0.02, or ≤ 5% of the mid price.
- Open interest must be > 1,000.
- On a $0.50 option, a $0.05 spread already costs 10% round trip. Liquidity filters are the most important part of the strategy.

### Contract selection (resolve with a chain census first)
**Problem:** with SPY and QQQ in the $600s, a 0.30–0.50 delta option with 1–7 days to expiry costs about $2–$6, which is $200–$600 per contract. That doesn't fit a $100 account.

**First task in the strategy work:** write `scripts/chain_census.py`. For each candidate underlying it logs, several times a day for 2 weeks:
- the contracts priced $0.30–$0.90,
- their delta, days to expiry, spread as % of mid, and open interest.

Then choose openly between two universes:

| Universe | Contracts | What's realistic / main risk |
|---|---|---|
| **(a) Index ETFs, low delta** | SPY/QQQ/IWM, delta about 0.10–0.25, 0–2 days to expiry | Behaves like a lottery ticket: needs a big move. Spreads are tight in absolute terms |
| **(b) Cheap liquid stocks** | Stocks priced about $10–$40 with penny-wide options, delta 0.30–0.50, 3–10 days to expiry | More reasonable delta. Wider spreads as a % of price, and more risk from company-specific news |

Either way:
- Premium of $0.30–$0.90 (≤ 90% of *settled* cash).
- Never hold into the final 30 minutes of the expiry session. Every time cutoff is **relative to the exchange calendar's close**, so it also works on early-close days such as Nov 27 and Dec 24.

### Candidate signals
Each signal is a pluggable `Strategy` class, and each must prove itself in paper trading.

| # | Signal | Idea | Why it might work / main risk |
|---|---|---|---|
| A | **Intraday trend continuation** | After the first 30 minutes, if price is above VWAP with rising volume and the opening range is broken, buy a call (mirror for puts) | Common and simple to test. Risk: chop, and the spread on false breakouts |
| B | **Volatility-cheap breakout** | Buy when implied volatility rank is low (<20) and realized volatility is expanding | Options are cheap relative to the move. Risk: time decay while waiting |
| C | **Mean-reversion fade** | Buy puts/calls after a large move away from VWAP when market breadth disagrees | Risk: fading a real trend |
| D | **0DTE lotto (experiment)** | Small, rare entries late in the day | **Usually loses money.** Tracked separately; it must clearly show positive expectancy in paper trading or it is cut |

### Exits (handled by deterministic code, never the LLM)
- Take profit at +50% to +100%.
- Stop loss at −35% to −50% of premium.
- Time stop: close if the target isn't hit within N minutes or by a set time of day.
- Hard flat by close − 30 min on expiry day. The watchdog repeats this independently of the engine (see §6).
- If the underlying is halted, suspend stop logic. Don't walk the limit price down into an empty order book.

### Growth path (unlocked by account size, not by the bot's judgment)
| Equity | What unlocks |
|---|---|
| < $500 | 1 contract, signals A–C only |
| $500–$2,000 | Up to 2 concurrent positions; risk 10–20% of equity per trade |
| > $2,000 | Margin account and Level 3: **defined-risk debit/credit spreads**, which cost less to hold over time and depend less on spread width |
| > $5,000–$10,000 | Cash-secured puts or a wheel on liquid stocks priced under about $50. A SPY cash-secured put needs about $65k |

---

## 3. Architecture (Python + FastAPI)

The trading loop runs **independently of the web server**. FastAPI is the control plane: status, start/stop, kill switch and config. It does not execute trades itself.
- The engine is a **separate process**: `python -m engine`, run under systemd.
- It holds a **single-instance lock** (a file lock plus a DB lease row). This prevents two engines from trading at once after a uvicorn reload or a second worker.
- FastAPI and the engine communicate only through SQLite tables (a commands table and a state table).

```
┌──────────────── FastAPI (control plane) ─────────────────┐
│ /health /status /positions /pnl /trades /config          │
│ POST /halt  POST /resume  POST /flatten   (auth required)│
└──────────────┬───────────────────────────────────────────┘
               │ SQLite only (commands + state tables); engine = separate process
┌──────────────▼──────────── Engine (asyncio) ──────────────┐
│ Scheduler (exchange calendar, ET) ─► Market data ingest   │
│        │                                  │               │
│        ▼                                  ▼               │
│ Strategy plugins ──signals──► RISK ENGINE (deterministic) │
│                                     │ approved orders     │
│                                     ▼                     │
│ Position monitor ◄──fills── Order manager ──► Broker API  │
│ (stops, time exits, expiry flatten)   (limit-at-mid,      │
│                                         walk, cancel)     │
└──────────────┬────────────────────────────────────────────┘
               │ read-only tools + structured outputs
┌──────────────▼─────────── Claude Agent SDK layer ─────────┐
│ Weekly analyst · Post-close reviewer · Incident triage    │
└───────────────────────────────────────────────────────────┘
        Watchdog process (separate): heartbeat → cancel all / alert
```

### Proposed layout
This is a new Python project. The current repo is a Rails app, so I recommend a **new repository**, or at least a self-contained `options_bot/` directory.

```
options_bot/
  app/main.py               # FastAPI app (control plane only; never starts the engine)
  engine/__main__.py        # engine entrypoint, single-instance lock
  scripts/chain_census.py   # logs which contracts actually fit the $100 filters
  app/api/                  # routes: status, control, journal
  engine/scheduler.py       # market calendar (pandas_market_calendars), session states
  engine/data.py            # quote/bar streams, option chain snapshots, greeks
  engine/strategies/        # base.py + trend.py, vol_breakout.py, fade.py
  engine/risk.py            # ALL hard limits live here; pure functions, 100% tested
  engine/orders.py          # limit-at-mid, price walking, idempotent client_order_id
  engine/positions.py       # exits, expiry flatten, reconciliation with broker
  brokers/base.py           # Broker protocol
  brokers/alpaca.py  brokers/tradier.py  brokers/paper_sim.py
  agents/analyst.py  agents/reviewer.py  agents/tools.py   # claude-agent-sdk
  backtest/                 # replay historical option bars through the same strategies
  db/models.py              # SQLModel/SQLite: trades, orders, signals, agent_runs, costs
  watchdog.py
  tests/
```

### Key libraries
`fastapi`, `uvicorn`, `pydantic` v2, `sqlmodel` (SQLite), `alpaca-py`, `httpx` (Tradier), `pandas`, `numpy`, `pandas_market_calendars`, `py_vollib` (greeks / implied volatility), `apscheduler` or plain asyncio, `claude-agent-sdk`, `structlog`, `pytest`.

### Order handling rules
- **Never send market orders on options.**
- Use a limit order at the mid price. Walk it toward the natural price in 1-cent steps every few seconds, up to a maximum slippage. Then cancel.
- Use a deterministic `client_order_id` per signal, so a retry cannot double-submit. Write it to the DB **before** submitting the order.
- A cancel stays *pending* until the broker confirms it. A fill can still arrive after the cancel request.
- Exit orders are **close-only** and sized from the broker's reported position, so a race can never open a new position in the opposite direction.
- **Reconcile** positions and orders with the broker at startup **and every minute**. The broker is the source of truth. Also handle exercise and contract-adjustment events.
- All times use `America/New_York` and come from the exchange calendar. Run NTP on the host. DST ends Nov 1.

---

## 4. Where Claude (claude-agent-sdk) fits, and where it must not

**Rule: the LLM never places an order and never relaxes a risk limit.** It can only *advise*, or *narrow* what the engine may do.

**No LLM in the trade path.** An intraday LLM veto can't be backtested honestly: the model's training data covers the test period, which is look-ahead bias. It would also add latency and make results non-reproducible. Event blackouts (FOMC, CPI, NFP, earnings) come from a **deterministic calendar**: published release dates plus a free earnings feed. They do not come from an LLM.

| Agent | When | Model | Tools | Output |
|---|---|---|---|---|
| **Weekly analyst** | Sunday evening | Haiku 5.5 (`claude-haiku-5-5`) | Limited web search, read-only `get_calendar` and `get_watchlist` | A Pydantic-validated `WeeklyNotes` covering anything the calendar missed. Its only effect is to **add** blackout tickers or dates, which only tightens limits |
| **Post-close reviewer** | After the close | Haiku 5.5. Use Sonnet 5.5 (`claude-sonnet-5-5`) only for a weekly deep review | Read-only journal and fills | A markdown trade journal plus *proposed* parameter changes, written to a file for **human approval**. Never applied automatically |
| **Incident triage** | On engine error or watchdog trip | Haiku 5.5 | Logs, read-only broker state | A summary pushed to your phone |

### Implementation notes
- Custom tools are exposed through an in-process SDK MCP server: `@tool` + `create_sdk_mcp_server` in `claude-agent-sdk`.
- Only read-only tools are allowed. There is no `place_order` tool.
- Restrict `allowed_tools` and disable Bash/Write in production runs.
- Every agent output is parsed into a Pydantic model. If parsing fails or the call times out, the engine uses **conservative defaults** (trade normally with base limits, or stand down on scheduled event days). It never falls back to more aggressive behavior.

### LLM budget
At $100, LLM costs count against returns.

The Agent SDK sends a large system prompt plus tool definitions on every turn, and web fetches pull in whole pages. Budget **100–300k input tokens per agent run**, not 30k.

| Item | Rough cost |
|---|---|
| Daily Haiku 5.5 reviewer, 200k input tokens ($0.10 / $0.50 per million tokens) | ≈ $0.02/day ≈ $0.45/month |
| Weekly Haiku analyst | < $0.20/month |
| Weekly Sonnet 5.5 deep review, about 300k input tokens ($2 / $10 per million tokens) | ≈ $0.65/run ≈ $2.60/month. Optional; skip while capital is under $500 |
| **Total** | **≈ $0.70/month without Sonnet, ≈ $3.30/month with it** |

- Track actual `usage` in an `agent_runs` table, and replace these estimates with measured numbers during paper trading.
- Set a **hard monthly cap of $3**. When the cap is hit, skip the agents. The engine never depends on them.

---

## 5. Risk engine (non-negotiable, enforced in code, unit-tested)

| Limit | Initial value |
|---|---|
| Max open positions | 1 |
| Max premium per trade | min($90, 90% of **settled** cash, tracked by the engine) |
| Unsettled funds | No entry may be funded by unsettled proceeds (prevents good-faith violations) |
| Max trades per day | 2, but settlement usually limits this to about 1 round trip per day |
| Daily loss limit | −$25 **realized + unrealized** → halt for the day |
| Drawdown circuit breaker | −40% from peak → **halt until a human runs `POST /resume`** |
| Floor | Equity < $30 → permanent halt (cannot buy a useful contract anyway) |
| Trading windows | Open + 15 min to close − 30 min, taken from the exchange calendar (handles early closes) |
| Event blackout | FOMC/CPI/NFP release minutes, and ticker earnings |
| Order type | Limit only, with max slippage of 2 cents or 5% |
| Expiry | Flatten everything by close − 30 min on expiry day |
| Data staleness | Quote older than 5 seconds → no new entries |
| Broker disconnect | No new entries; the watchdog cancels open orders after 60 s without a heartbeat |

---

## 6. Running "constantly and autonomously"

- **Active hours:** the engine trades 09:30–16:00 ET. The agents run pre-market and post-close. Outside those times it idles; nights and weekends are just heartbeats.
- **Hosting at near-zero cost:**
  - An always-on home machine or Raspberry Pi costs about $0.
  - A free-tier cloud VM costs $0.
  - A $4–5/month VPS works, but costs 4–5% of capital per month.
  - Run under `systemd` or Docker with `restart=always`.
- **Watchdog:** a separate process checks the engine's heartbeat. If the engine is stuck, the watchdog cancels open orders through the broker API and sends an alert. It also **flattens positions near the close by itself**, whether or not the engine is healthy.
- **External dead-man switch:** stops are client-side, because Alpaca options support only limited order types. So a free cloud cron job (for example GitHub Actions or a free-tier function) pings `/health` during market hours. If the check fails, it flattens through the broker API and alerts you. This covers the case where the home machine loses power and the watchdog goes down with it.
- **Alerts:** use ntfy.sh or Telegram (both free). Send one on fills, halts, errors and the daily summary.
- **Secrets:** broker keys and the Anthropic key go in env vars or a `.env` file outside the repo. The control API needs a bearer token. Never expose it publicly without auth (Tailscale or a Cloudflare Tunnel is a good option).
- **Observability:** use structured JSON logs, a `/status` page, and a daily P&L / expectancy report.

---

## 7. Phased roadmap with go/no-go gates

| Phase | Work | Exit gate |
|---|---|---|
| **0. Accounts** (days 1–2) | Open Alpaca with paper keys. Get Anthropic API keys. Verify Tradier/tastytrade options-data terms | Keys in `.env`; broker decision documented |
| **1. Skeleton** (week 1) | FastAPI + engine loop + SQLite + Alpaca paper adapter + scheduler + risk engine with full tests | Engine runs a full simulated day without errors; `/halt` works |
| **1b. Chain census** (weeks 1–2, in parallel) | Run `chain_census.py` to choose universe (a) or (b) from §2 | A documented universe in which ≥ 3 contracts per day pass every filter |
| **2. Backtest** (weeks 2–3) | Historical option bars (Alpaca historical options data) through the same strategy code. Bars have **no bid/ask**, so spreads are an *assumption*: run a sensitivity sweep (e.g. 2/5/10% round-trip cost) | Expectancy stays positive at the pessimistic spread setting. Out-of-sample split. Bootstrap 95% confidence interval on expectancy with lower bound > 0. Max drawdown < 40%. Adjust for testing several signals |
| **3. Paper trading** (weeks 4–9, ≥ 6 weeks) | Paper trading needs **real quotes**: Alpaca's free feed is randomized. Either fund the live broker early and run `paper_sim` on its real-time data, or pay for 1 month of Alpaca OPRA ($99) only for the final weeks. Track LLM cost. Run a monthly kill-switch / `/flatten` drill (with a dry-run mode) | ≥ 40 trades; confidence-interval check as in Phase 2; no risk-rule or good-faith violations; total costs (LLM + data + hosting) < 5% of capital per month |
| **4. Micro-live** ($100) | Same code, chosen live broker, 1 contract | Kill switch tested live. Compare live fills against paper. Stop and re-evaluate after −40% |
| **5. Scale** | Only when equity is higher, per the growth path in §2 | Re-run the gates for each new capability |

If no strategy passes Phase 2 or Phase 3, the right result is **don't go live**. The project has still succeeded as a learning exercise.

---

## 8. Other things to keep in mind
- **Taxes:** every gain is short-term. Wash-sale rules apply to rapid re-entries in the same contracts. Export the trade journal for tax filing.
- **Broker terms:** use only official APIs. Alpaca, Tradier and tastytrade allow automated trading.
- **Not financial advice:** this is an engineering plan. Options can lose 100% of the premium quickly.

## 9. Open questions for you
1. Where will it run: your own always-on machine, or a paid VPS?
2. Are you OK with paper trading for at least 6 weeks before real money?
3. Are you willing to spend money on real-time data during paper trading? Options: fund the live broker early, or pay $99 for one month of Alpaca OPRA.

Decided after advisor review:
- **No LLM in the trade path.**
- **Use a new repository** for the bot, not this Rails repo.

## 10. Advisor review log
An independent review (2026-10-10) found four blocking issues. All four are addressed above:
1. The contract filters didn't fit SPY/QQQ prices → added the chain census and the choice between universes (a) and (b).
2. Cash-account good-faith violations were missing → added settled-cash tracking.
3. Paper trading on Alpaca's randomized feed would validate nothing → real-data requirement in Phase 3, and a spread-sensitivity sweep in Phase 2.
4. Engine lifecycle and calendar bugs → separate engine process with a lock; all cutoffs relative to the calendar close.

It also raised several should-fix items, all applied: the risk-of-ruin and break-even framing, a higher LLM token estimate and dropping the veto, an external dead-man switch, and order-correctness rules.

---

## 11. Revisions for $1,000 capital and paid data

### What changes
- **Data:** $99/month for Alpaca's real-time feed is about 10% of $1,000 every month, or about 119% a year. Paying it permanently is still too expensive. Pay for **one month** to run Phase 3 paper trading on real quotes. Then trade live at a broker whose real-time data comes with a funded account (Tradier or tastytrade; confirm first), unless Alpaca's data turns out to be the only good option.
- **Position size:** risk 5–10% per trade ($50–$100 of premium). That means 1 contract per trade with up to 2–3 positions open. Each loss is no longer a large share of the account, so the drawdown breaker becomes −25% from peak, and the daily loss limit becomes −$60.
- **Universe:** the cheap-stock universe (b) stays viable. Low-delta index ETF options (a) can now be sized properly.
- **Still a cash account:** margin needs at least $2,000, so spreads still aren't available. **Getting to $2,000** is the most valuable upgrade, because it unlocks defined-risk credit spreads (see below).

### Can we reverse a losing algorithm?
Usually not. Each trade's result is split into two parts:

`net P&L = gross edge (did the signal predict direction?) − costs (spread + fees + time decay)`

Reversing the trade flips the **edge**, but you pay the **costs** again on the reversed trade. So:
- **The original's net is about −costs (the signal is just noise):** most losing bots look like this. Reversing it also loses about −costs. Two losers.
- **The original loses *more than twice the costs*:** the signal is reliably *wrong*. Reversing it can win. To check this, log **gross P&L before costs** separately in the trade journal. Only consider reversing if that gross loss is statistically significant (bootstrap confidence interval below −costs).

Three more catches:
1. **Exits aren't symmetric.** A +100% target / −50% stop on a long call does not mirror into a sensible long-put rule. The reversed strategy has to be re-specified and re-tested as a new strategy.
2. **Overfitting.** Reversing after seeing the results is one more data-snooping step. The reversed version must pass the Phase 2 and Phase 3 gates on **fresh, unseen data** before any money goes into it.
3. **"Loses more often than it wins" is not the same as "loses money."** Option buyers often win only 30–40% of trades and still come out ahead when the winners are big. Judge strategies by **expectancy (average $ per trade after costs)**, never by win rate.

### Reversing the side (buying → selling)
This is the "reversal" that has a known reason to work. Option sellers collect time decay and, on average, the **volatility risk premium**: implied volatility tends to price in bigger moves than actually happen.

The trade-off is the opposite payoff shape. Sellers win often and in small amounts, and occasionally take a large loss. It also needs collateral:

| Capital | Selling strategy available |
|---|---|
| $1,000 (cash) | **Cash-secured puts / covered calls** on liquid stocks priced under about $10 (collateral up to about $1,000 per contract). One position at a time, so concentration risk is high |
| $2,000+ (margin, Level 3) | **Defined-risk put credit spreads / iron condors** on SPY/QQQ/IWM, e.g. $1–$2-wide spreads risking $100–$200 each. This is the standard small-account setup for a seller |

**Recommendation:** run **both** in Phase 2 backtests and Phase 3 paper trading: (1) the buying signals, with gross-vs-net logging, and (2) a mechanical selling strategy. Use the selling strategy only once capital reaches $2,000 so it can use defined-risk spreads, or use cash-secured puts at $1,000 if the census finds suitable liquid stocks under $10. Let the gates decide which one goes live.

---

## 12. Continuous trading, micro-scalping, and adding to losers

### The proposed idea
"Take about $1 profits at high volume, hold losers until they come back, and buy more when a trade goes against us."

### Why it fails as stated
1. **A $1 target is the same size as the trading costs.** One options tick is $0.01 × 100 = **$1 per contract**. Even SPY options have at least a 1-tick spread, so buying at the ask and selling at the bid costs about $1 before any fees. Fees add $0.10 at Alpaca and $0.70 at Tradier Standard. Netting $1 needs the price to move about 2 ticks your way on every trade, and the other side of each trade is a market maker whose business is exactly this.
2. **"Volume" isn't possible in a cash account.** Settled-cash rules (§0, item 6) let $1,000 turn over about once per day. Frequent trading in and out needs a margin account, which needs at least $2,000, and must follow the broker's intraday margin rules.
3. **"Wait until it comes back" means no stop loss on an asset that loses value every day.** A losing option often never recovers before it expires worthless. The math:
   - Suppose 95% of trades win +$1.
   - Suppose 5% lose an average of −$40.
   - Expectancy = 0.95 × $1 − 0.05 × $40 = **−$1.05 per trade**.
   - Breaking even needs a **97.6% win rate**. One bad trend day wipes out weeks of $1 wins.
4. **Buying more is not a hedge. It is averaging down, a form of martingale.** It *increases* exposure in the direction that is losing. In a cash account it runs out of money just as the losing streak gets longest, which ends in ruin.
   - A real hedge takes the *opposite* position, for example buying a put against a long call. That locks in the loss rather than recovering it, and it costs another spread.

### Where "many small wins" legitimately comes from
**Defined-risk option selling** (put credit spreads / iron condors, §11). It has a high win rate and small gains, and the *maximum loss is fixed when the trade is opened*. That fixed maximum replaces "hope it comes back". It needs $2,000 and a margin account (Level 3).

### Building for continuous trading (any strategy)
- **Streaming, event-driven engine:**
  - Use WebSocket quote and trade-update streams, not polling.
  - Strategies react to events (quote, bar close, fill).
  - Use one asyncio task per subsystem, connected by queues.
- **Each position is a state machine:**
  `SIGNAL → PENDING_ENTRY → OPEN → EXITING → CLOSED`, plus `CANCEL_PENDING` / `ERROR`.
  - Each transition is persisted to SQLite *before* the broker is called.
  - Allowed transitions are listed explicitly, so an illegal one raises an error.
- **Every position opens with its exit plan already set:** profit target, stop loss, time stop and expiry flatten. Changes after entry can only *tighten* it.
- **Adding to a position (if ever):** only as a *pre-planned* second entry that is counted against the trade's risk budget at entry time.
  - At most 1 add.
  - The combined stop is unchanged.
  - **Never an add triggered by a loss.**
  - The risk engine rejects any order that would increase total risk on a losing position.
- **Rate controls:** max orders per minute, max round trips per day (settlement-aware), and a cooldown after each loss.
- **Recovery:** after any restart, reconcile with the broker, rebuild the state machines from the DB and the broker's records, and resume. Positions the engine doesn't recognize trigger an alert and are only allowed to exit.

### How to test the user's idea anyway
Add it as **Strategy E: micro-scalp** in the backtest and in paper trading only, with:
- a +$1 net target,
- a hard stop,
- at most 1 pre-planned add,
- the full spread and fees modeled.

If it passes the Phase 2 and Phase 3 gates on real quotes, it can go live like any other strategy. The data decides. Strategy E without a stop and with unlimited adds will **not** be built into the live engine.
