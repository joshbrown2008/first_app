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

---

## 13. Margin account setup notes
- **Alpaca opens every account as a margin account.** Under $2,000 of equity the account has no borrowing power. However, margin accounts usually allow trading with unsettled funds, which would remove the good-faith-violation limit from §0, item 6. **Confirm this with Alpaca support before relying on it.** Spreads still need **options Level 3** approval, which is a separate application. You get two tries; if denied, you can reapply after 60 days.
- **tastytrade:** choose "margin" when opening the account. Full margin privileges need at least $2,000 of equity. Spreads need options approval, which takes 1–3 business days.
- **Engine rule:** the margin account is for *features* (spreads, using unsettled funds), **not borrowing.** The risk engine caps gross exposure at 1× equity (cash), and every spread counts its maximum loss against that cap.

---

## 14. Recommended setup ("the one plan")

"Maximize gains" here means **maximizing long-run account growth while staying in the game**. Sizing for the biggest possible upside maximizes the chance of losing everything.

### Accounts
| Step | Action |
|---|---|
| Now | Open **Alpaca** (paper + live) for development. **Apply for options Level 3 now**, because approval can take a while and a denial means waiting 60 days to reapply |
| Now | Open the live broker: **Tradier** if a funded account gets real-time options quotes through the API, otherwise **tastytrade**. Choose a margin account and apply for spreads approval |
| Paper (weeks 4–9) | Pay for **one month** of Alpaca OPRA ($99) *or* use the live broker's data with `paper_sim` |
| Go-live | Fund **$2,000** (the margin threshold, which unlocks defined-risk spreads). Fund only *after* the paper gates pass |

**Why not Alpaca live:** at about 6 spreads a month, Alpaca costs about $0.20 in fees plus **$99 of data, about $100/month (5% of $2,000 every month)**. tastytrade costs about $2.40 per spread round trip, about $15/month, with data included (verify). Tradier Standard costs about $1.40 per spread.

### Core strategy (about 80–90% of the risk budget): mechanical put credit spreads
- **Underlyings:** SPY, QQQ, IWM. Treat them as **one correlated bet**.
- **Entry:**
  - 30–45 days to expiry.
  - Short put at about 16–20 delta. Long put $2–$5 below it.
  - Credit ≥ 1/4 to 1/3 of the spread width.
  - Enter 10:00–11:30 ET, when spreads are tightest.
  - Only when IV rank > 25.
- **Order:** a single multi-leg limit order at mid, walked $0.01 at a time toward the natural price. **Never leg in.**
- **Exits:**
  - Take profit at **50% of credit**.
  - Stop at **2× credit** (spread value = 3× credit).
  - Close at **21 days to expiry** whatever the P&L.
  - Close immediately if the short strike is in the money with little extrinsic value left (early-assignment risk; one assigned SPY put = about $65k of stock).
- **Sizing at $2,000:**
  - Max loss per spread ≤ $200 (10%).
  - At most 3 open at once.
  - Total defined risk ≤ 30% of equity.
  - Re-size from equity every week.
- **Honest math:** a $2-wide spread for $0.40 credit wins +$20 at 50% and loses −$80 at the stop. That needs an **~80% win rate** just to break even. A 16-delta short put expires out of the money about 84% of the time, so the edge is real but **thin**. That is why the Phase 2 and Phase 3 gates exist.

### Satellite (about 10–20% of the risk budget, only if gates pass)
The long-option signals (A–C) and Strategy E, each with its own budget and kill rule.

### Execution and operations
- The engine checks positions every minute while the market is open. This strategy isn't latency-sensitive.
- Profit-take orders are re-placed every morning. Many brokers make options orders day-only; verify per broker.
- Stops are enforced by the engine. The watchdog and the external dead-man switch force a close if the engine is down.
- A weekly report covers expectancy, win rate versus break-even win rate, drawdown and costs as % of equity.

### Expectations
A *good* result is beating SPY's return with drawdowns under about 25%. Any setup that promises 5–10% a month is taking tail risk that hasn't shown up yet.

---

## 15. Higher-return strategy set (replaces §14 as the default candidate list)

**Ground rule:** every extra % of return below comes from taking a risk that §14 avoided. That can mean more leverage, faster turnover, or more exposure to sudden moves. All dollar figures are **illustrative mechanics on $2,000, not backtest results**. Phase 2 replaces them with measured numbers.

### Structural upgrade: trade XSP instead of SPY for premium selling
XSP is the Mini-SPX index option, 1/10 the size of SPX.
- **Cash-settled and European-style:** no early assignment, and no risk of receiving $65k of stock.
- **Expirations every day**, so 0DTE is available.
- **Section 1256 tax treatment:** 60% long-term / 40% short-term, whatever the holding period.
- **Broker check:** Alpaca may not support index options, while tastytrade and Tradier do. Verify API support for XSP before choosing the live broker.

### S1. 0DTE XSP put credit spreads with a regime filter (high turnover)
- **Entry:**
  - Every qualifying day, about 10:30–11:00 ET.
  - Sell a ~10-delta put and buy one $2 lower ($200 width). Credit about $0.20, so about $20 per spread.
  - Run only if both are true: (a) SPX is above its 20-day moving average, and (b) VIX term structure is in contango (VIX < VIX3M).
- **Exits:** stop when the spread trades at 3× credit. Otherwise let it expire; it is cash-settled.
- **Size:** 2 spreads per day; max loss is about $360 (18%) on a crash day.
- **Mechanics:**
  - Ideal (21 winning days): **≈ +$800/month (≈ 40%)**.
  - One stop-out costs about −$80, a gap through the stop up to −$360.
  - Break-even win rate is about 75–80% with stops.
- **Honest view:** academic and practitioner evidence on 0DTE premium selling is **mixed, and costs matter a great deal**. This is the highest-upside candidate and also the most likely to fail its gates.

### S2. Trend-filtered leveraged index exposure (deep ITM calls)
- **Entry:**
  - When SPX is above its 200-day moving average, hold 1 XSP call at about 80 delta, 3–6 months to expiry. That's about $900–1,200 of premium controlling about $6,600 of index.
  - Total exposure: about 2.5–3× the account's equity.
- **Exits:** exit to cash when SPX closes below its 200-day moving average. Roll at 60 days to expiry.
- **Mechanics:**
  - An average month (index +0.8%) gives about **+2%/month** after time decay.
  - A strong month (+3%) gives about **+8%**.
  - A fast crash before the filter exits can cost **−30 to −50%**.
- **Why it's credible:** it harvests the equity risk premium with leverage. The trend filter has historically helped avoid long bear markets, though it gets whipsawed in choppy ones.

### S3. Earnings implied-move overpricing (seasonal)
- **Entry:** 15–30 minutes before the close on the day before earnings, sell a defined-risk iron condor at about 1× the implied move on liquid large caps. Stocks under about $150 or with $1–$2 strike widths, so they fit the account.
- **Exits:** close at the next morning's open.
- **Size:** max loss ≤ 5% of equity per event, ≤ 3 events per week.
- **Mechanics:** implied moves tend to exceed realized moves on average, but the losers are big. The edge per event is small, and the main value is low correlation with S1 and S2.

### S4. Regime overlay (decides which of S1–S3 may run)
| Regime | Condition | S1 | S2 | S3 |
|---|---|---|---|---|
| Calm uptrend | SPX > 200d moving average, VIX < VIX3M | ✅ full | ✅ full | ✅ |
| Choppy | SPX > 200d moving average, VIX ≥ VIX3M | ❌ | ✅ half | ✅ half |
| Downtrend | SPX < 200d moving average | ❌ | ❌ (cash) | ✅ half |

The overlay switches strategies off when their known failure conditions appear. It never increases size beyond the base limits.

### Portfolio (growth mode)
| Sleeve | Risk budget |
|---|---|
| S2 | about 50% of equity in premium |
| S1 | max daily loss ≤ 10% of equity |
| S3 | ≤ 5% of equity per event |

| Outcome | Rough monthly result |
|---|---|
| **Ideal month** (all signals win, index +3%) | **≈ +25–40%** |
| **Realistic, if the edges hold** | **≈ +1.5–3%/month (about 20–40%/year)**, with **drawdowns of 30–50%** |
| **If S1 has no edge** | S2 alone, about 15–25%/year with deep drawdowns |

**Gates still apply:** each sleeve must pass Phase 2 (with a cost sensitivity sweep) and Phase 3 on its own before it gets capital. The drawdown breaker moves to **−35%** (human resume required) in growth mode.

---

## 16. Scalping track

### What has to be true for scalping to work
1. **Cost per round trip must be small compared with the target.** It must be under about 20–25% of the average win.
2. **The signal must have a measurable edge** that survives realistic fills, judged by expectancy on quote-level data.
3. **Day-trading permissions.** That means a margin account with ≥ $2,000 under the new FINRA intraday-margin rules. A cash account can't scalp because of settlement.

Evidence warning: large studies of retail day traders (Taiwan, Brazil) find that only a very small fraction are profitable after costs over time. The bot has to *prove* an edge before it gets capital.

### Instrument choice
| Instrument | Typical cost per round trip | Example target | Cost / target | Notes |
|---|---|---|---|---|
| **MES (Micro E-mini S&P futures)** | 1 tick spread ($1.25) + about $1–2 fees ≈ $2.50–3.50 | 4–8 ticks ($5–10) | about 35–50% at small targets, better at 8+ ticks | **The standard scalping instrument**: nearly 24h, deep liquidity, 60/40 tax, low day margin. Needs a futures-enabled broker (tastytrade, IBKR, Tradier futures). **Not options** |
| **0DTE SPY options, ATM (~0.50 delta, ~$1–3)** | $0.01 spread ($1) + fees $0.10–0.70 ≈ $1.10–1.70 | $0.15–0.25 ($15–25) | about 6–10% | Workable: the targets are much bigger than the $1 scalp in §12. Fast time decay and gamma risk |
| **0DTE XSP options** | Often $0.02–0.05 spreads | $0.20–0.30 | about 10–20% | Cash-settled, but less liquid than SPY. Use only if spreads are measured tight |
| Low-delta OTM options | 1 tick on $0.10–0.30 = 3–10% | — | **Bad** | Avoid for scalping |

**Default:** 0DTE SPY ATM options (stays within "options trading"). Run an **MES futures variant in parallel paper trading** as a benchmark. If MES does better, consider switching instruments.

### Scalping strategies (each is its own pluggable `Strategy`, each gated separately)
All on 0DTE SPY options (or MES), 1 contract, entries only 09:45–15:30 ET.

| ID | Setup | Entry | Exit |
|---|---|---|---|
| **SC1 Opening Range Breakout** | 15-min opening range on SPY | Buy ATM call (put) on a 1-min close above (below) the range with relative volume > 1.5 | +$0.20 target / −$0.10 stop (on option price) / 10-min time stop. At most 2 per day |
| **SC2 VWAP pullback** | SPY above VWAP with rising 20-EMA (trend day) | Buy call on a pullback that touches VWAP or the 9-EMA and the next 1-min bar closes back up (mirror for puts) | +$0.15 / −$0.10 / 8 min. Disabled if price crosses VWAP 3+ times (chop) |
| **SC3 Key-level reaction** | Prior-day high/low, overnight high/low, round numbers | Two modes. *Fade*: first touch, rejection wick, and volume falling. *Break*: 1-min close through the level with volume | +$0.15 / −$0.08 / 5 min. At most 1 trade per level per day |
| **SC4 Momentum burst** | 1-min bar range > 2× the 20-bar average and volume > 3× | Enter in the burst direction on the first 1-min pause that holds | +$0.20 / −$0.10 / 5 min. No entries within 2 min of scheduled releases |

**Example math (SC1):** win +$20, loss −$10, cost about $1.50 per trade.
- Break-even win rate ≈ **38%**.
- At a 45% win rate: expectancy ≈ 0.45×20 − 0.55×10 − 1.5 = **+$2.00 per trade**.
- 8 trades/day × 21 days ≈ **+$340/month (≈ 17% on $2,000)**.
- At a 35% win rate: ≈ −$1.50/trade ≈ **−$250/month**.

The whole outcome turns on a few percentage points of win rate, and only quote-level testing can measure that.

### Scalping risk rules (risk.py)
| Rule | Value |
|---|---|
| Risk per trade | ≤ 1–2% of equity (a $10–20 stop on 1 contract at $2,000) |
| Daily loss limit | −3% (−$60) → flat and done for the day |
| Consecutive losses | 3 in a row → 30-min cooldown. A second time → done for the day |
| Max trades per day | 10 |
| Adding to losers | **Never** |
| Spread filter | Option spread > $0.02 or quote > 1 s old → no entry |
| News | No entries 2 min either side of scheduled releases (CPI, FOMC, NFP) |

### Engine changes for scalping
- **Streaming everything:** WebSocket quotes, trades and order updates (paid real-time OPRA required). Build 1-second and 1-minute bars locally.
- **Client-side brackets:** after each fill, the engine manages target, stop and time stop in a sub-second loop. A stop triggers a marketable limit (the natural price + $0.02 cap), never a plain market order.
- **Latency and slippage telemetry:** log signal → submit → ack → fill times and fill vs mid. Halt the strategy if median slippage exceeds the modeled value.
- **Hosting:** a cloud VM in a US-East region close to the broker (about $5–10/month). This beats a home connection for reliability. Python stays fine at these 100 ms timescales.

### Testing scalping (stricter than §7)
- **Data:** bars aren't enough. You need **historical options quotes, not just bars** (ThetaData or Polygon-class options data; check pricing) for at least 6–12 months of 0DTE SPY.
- **Backtest fills:** model fills at the far side of the spread plus 1 tick of slippage, and test sensitivity.
- **Validation:** walk-forward (optimize on one month, test on the next) to prevent overfitting.
- **Paper:** ≥ 300 trades per strategy, with the bootstrap expectancy check (lower bound > 0).
- **Live:** 1 contract, comparing live slippage to paper. If live expectancy falls more than 50% below paper, stop.

---

## 17. News-driven single-stock track with an LLM proposal reviewer

This **supersedes §10's "no LLM in the trade path"** decision, at the user's request, under three conditions:
1. The reviewer can only **reject or reduce** a trade.
2. Its value is **measured against counterfactuals**.
3. It is validated **forward-only** (see Testing below).

### Core idea: trade the gap between *predicted* and *implied* move
High-volatility stocks have expensive options, because the market already prices a big move. Buying them pays off only if the realized move **exceeds the implied move**. Every proposal must state:

`edge = predicted |move| − implied |move|` (from the ATM straddle, for the trade's horizon)

| Edge | Structure |
|---|---|
| Strongly positive, with direction | **Debit spread** in the predicted direction. This is cheaper than a single long option when IV is high, and its risk is defined |
| Strongly positive, no direction | Long straddle or strangle (rare; costly) |
| Strongly negative (overpriced) | Defined-risk **iron condor / credit spread** (S3-style) |
| Near zero | **No trade** |

### Universe (rebuilt daily)
- Stocks priced $10–$200 with average daily option volume > 5,000 contracts.
- ATM option spread ≤ 5% of mid, and open interest > 500 at the chosen strikes.
- **IV rank > 50 or IV/HV > 1.3** (the "high-volatility" screen).
- Exclusions: stocks under $5, recent IPOs (< 90 days), pending M&A deals, names with hard-to-borrow or halt-prone history (some small biotechs).
- Target about 50–150 names.

### Where an edge can plausibly exist (and where it can't)
- ❌ **Racing the headline.** Algorithms react within milliseconds, and an LLM pipeline takes seconds. **Never enter in the first ~5 minutes after a headline.**
- ✅ **Slow diffusion.** Post-earnings announcement drift, guidance/analyst revision drift, and news on less-covered mid caps. Academic work (e.g. LLM headline sentiment predicting next-day returns) found effects concentrated in **smaller stocks**, and they **shrink as the approach spreads**.
- ✅ **Interpreting complex text.** 8-K filings, guidance nuance, FDA/regulatory language, where a reader adds value over keyword sentiment.
- ✅ **Overreaction fades.** A large move on news the model scores as low-materiality.

### Pipeline
```
 News (Alpaca/Benzinga WS, ~600–900/day) + SEC EDGAR 8-K feed + earnings calendar
        │ dedupe, map to tickers, drop non-universe
        ▼
 [1] Triage — claude-haiku-5-5, structured output (strict JSON schema)
     {material: bool, event_type, direction: -1..1, surprise: 0..1, horizon, confidence}
        │ material only (~20–60/day)
        ▼
 [2] Predictor — deterministic quant model (gradient boosting / logistic)
     features: triage scores, pre-news move, relative volume, gap vs implied move,
               IV rank, sector/SPY move, time of day, days to earnings
     → forecast distribution of move over horizon h; compute edge vs implied move
        │ |edge| > threshold
        ▼
 [3] Proposal builder — deterministic: picks the structure from the table above,
     strikes, size (≤ 2–3% equity max loss), entry limit, exits
        ▼
 [4] Reviewer ("the advisor") — claude-opus-5-5, effort medium, structured output
     input: proposal, raw news text, predictor features, recent price/IV, open book, risk state
     output: {verdict: APPROVE|REDUCE|REJECT, size_multiplier ≤ 1.0, reasons[], red_flags[]}
     role: adversarial — "find why this trade is wrong" (stale news, already priced in,
           ambiguous ticker mapping, binary event inside horizon, liquidity, correlation)
        ▼
 [5] Risk engine — deterministic final gate (unchanged; can still reject anything)
        ▼
 Order manager → broker
```

**Why a separate review call instead of the API's built-in advisor tool:** the Claude API has an advisor tool (a cheaper executor model consults Claude Opus 5.5 mid-generation). With current advisor models, though, the advice comes back **encrypted**: it can be passed back to the model but not read or logged. Trade decisions need an **auditable, readable reason**. So the reviewer is its own call with a strict output schema, and every verdict is stored. The advisor tool can optionally be used *inside* the proposal step, but never as the decision record.

**Failure handling:** if the reviewer errors, times out (> 20 s) or refuses, the default is **REJECT**. The reviewer can never raise size or create a trade.

### Measuring whether the reviewer helps ("advisor lift")
- Every proposal is recorded. **Rejected proposals are paper-traded as a shadow book.**
- Compare expectancy of approved trades against all proposals, and against rejected ones.
- If the reviewer doesn't improve expectancy, net of its cost, over ≥ 100 proposals, switch it off and keep the predictor alone. Apply the same test to the Haiku triage against a plain keyword/sentiment baseline.

### Testing (stricter: LLM look-ahead bias)
- An LLM has read how historical events turned out. **Backtests of LLM-scored historical news are contaminated.**
- Mitigations:
  - (a) Train the predictor's non-LLM features on history.
  - (b) For LLM features, use only news **after the model's training cutoff**, or mask entity names and dates. Masking only partly helps.
  - (c) Rely mainly on **forward paper trading**: ≥ 3 months and ≥ 150 proposals, with the bootstrap expectancy check.

### Cost (estimated, verify with measured usage)
| Component | Volume | Cost |
|---|---|---|
| Haiku 5.5 triage | about 900 headlines/day × ~1.5k tokens ≈ 1.4M input tokens/day | ≈ **$0.15/day** |
| Opus 5.5 reviewer | about 5–20 proposals/day × ~15k input + 1.5k output | ≈ **$0.10 each → $0.50–2/day** |
| **Total** | | ≈ **$15–45/month**, under a monthly cap (e.g. $40) and a per-day cap |

At $2,000 this is about 1–2% of capital per month. The track must earn that before it adds anything.

### Risk rules specific to this track
- Max loss per trade ≤ 2–3% of equity. Defined-risk structures only (debit/credit spreads).
- ≤ 3 concurrent single-stock positions, ≤ 1 per sector.
- No holding through a scheduled binary event (earnings, FDA date) unless the trade *is* that event and is sized for total loss.
- Halt-prone names are excluded. If a holding is halted, alert and close when it reopens.
- News-track daily loss limit −4%, separate from the scalping and index tracks. A global limit sits across all tracks.

### Data sources
- **Alpaca news API (Benzinga):** real-time WebSocket (`wss://stream.data.alpaca.markets/v1beta1/news`) plus REST history back to 2015.
  - **Price: appears to be $0 with any Alpaca account.** It launched free, with rate limits set by the market-data plan: 200/min on Basic, 10,000/min on Algo Trader Plus.
  - Alpaca's current pricing page lists no news fee, but it doesn't list news as an item either.
  - Price check 2026-10-10. **Confirm by connecting a free/paper key to the news stream in Phase 0.**
- **Fallbacks if Alpaca news is gated:**
  - Massive (formerly Polygon) Benzinga add-on: $99/month, plus a base plan.
  - Finnhub: free tier at 60 calls/min (REST, adequate for a 100-name universe), or about $50/month paid.
  - Benzinga direct: quote-only, no published price.
  - Benzinga Pro ($37/month) has **no API**, so it's not usable.
- **SEC EDGAR:** free 8-K / filings feed.
- **Earnings calendar:** a free or low-cost provider.
- **Options IV / implied move:** from the paid OPRA feed already planned.

---

## 18. Research swarm (100 researchers → evaluators → final evaluator) and how it drives scalping

### Two reality checks that shape the design
1. **Latency.** A researcher → evaluator → final chain takes about **10–45 s**. Each hop is a model call (Haiku 1–3 s, Sonnet 3–10 s, Opus with thinking 5–30 s). A scalping setup lasts seconds. **The hierarchy cannot approve individual scalps.**
2. **Cost is cheap per call, not per month at $2,000.** Suppose 100 researchers "think" every minute with about 20k tokens of context on Haiku 5.5 ($0.10 per million input tokens):
   - 100 × 390 min × 20k ≈ 780M tokens/day ≈ **$78/day ≈ $1,600/month**, which is 80% of capital every month.
   - So the swarm must be **event-driven** (wake on news or price events), not a constant polling loop.

### Design: "slow brain, fast hands"
The LLM swarm **pre-decides conditions**. Deterministic code **executes in milliseconds**.

```
 EVENT BUS (deterministic, cheap): news headlines · 8-K filings · volume spikes ·
   level breaks · IV jumps · halts · sector moves
        │ routed by ticker
        ▼
 L1 RESEARCHERS (100 logical agents × ~5 tickers each = ~500-name universe)
   claude-haiku-5-5, woken only by events for their tickers; per-ticker notes in DB
   → SignalCard {ticker, thesis, direction, horizon, confidence 0–1,
                 key levels, invalidation price, evidence ids}
        ▼
 L2 SECTOR EVALUATORS (~8, one per sector), claude-sonnet-5-5, batch every 2–5 min
   dedupe, resolve conflicting cards, add sector context, rank, promote top N
        ▼
 L3 FINAL EVALUATOR, claude-opus-5-5
   portfolio view: correlation, risk budget, regime (S4), open positions
   → ARMS a PlaybookEntry (or rejects)
        ▼
 PLAYBOOK (DB table, each entry has a TTL of 15–120 min):
   {ticker, allowed side (long/short/none), setups allowed (SC1–SC4),
    levels, invalidation, max contracts, expires_at}
        ▼
 FAST SCALPING ENGINE (deterministic, sub-second): trades SC1–SC4 triggers
   ONLY on armed tickers, in the armed direction, until TTL or invalidation
        ▼
 RISK ENGINE (unchanged final gate) → order manager → broker
```

- The LLMs **arm and disarm**; they never send orders.
- L3 can only arm within the risk engine's caps. Expiry, an invalidation-price hit, or an L3 "disarm" removes an entry instantly.
- **Playbook example:** `NVDA · long only · SC2 (VWAP pullback) & SC3 (break) · above 182.40 · invalidate < 179.80 · 1 contract · expires 11:45 ET` (after a positive guidance headline that L1 judged under-priced).

### Making the hierarchy add information, not noise
- **100 copies of one model are not 100 independent opinions.** Their errors are correlated, so a majority vote overstates confidence. Get diversity from **different inputs and roles**:
  - news reader,
  - technicals/levels reader,
  - options-flow/IV reader,
  - a dedicated skeptic.
- **Keep it to 2–3 layers.** Each layer loses detail (the "telephone game"). Pass the structured cards and evidence ids up the chain, not summaries of summaries.
- **Researcher leaderboard:** score every SignalCard after its horizon ends. Use a calibration (Brier) score and the hit rate of the predicted direction against the move. L2 weights cards by each researcher's track record. Every month, retire or re-prompt the bottom 10%.
- **Prove each layer pays for itself:** shadow-trade what each layer rejected (as in §17). Drop any layer that doesn't improve expectancy by more than its token cost.

### Implementation
- Researchers are **stateless workers** pulled from a queue (an asyncio pool with a concurrency limit and API rate-limit backoff). "100 bots" means 100 prompt/ticker configurations, not 100 running processes.
- Use the plain Anthropic SDK (Messages API + strict structured outputs) for L1–L3: fast, cheap and auditable.
- Use `claude-agent-sdk` for slower, deeper research jobs, such as reading a 10-K or building the pre-market dossier on a ticker.
- **Prompt caching:** each researcher's stable prefix (role, rules, ticker notes) is cached. Only the new event is fresh input.
- **Batch API (50% off)** for non-urgent work: overnight dossiers and the leaderboard post-mortems.

### Cost (event-driven, estimated; replace with measured usage)
| Layer | Volume per day | Model | Est. $/day |
|---|---|---|---|
| L1 researchers | about 3,000 wakes (≈900 news + ≈2,000 price events) × 8k input / 0.5k output | Haiku 5.5 | about $3 |
| L2 evaluators | 8 sectors × about 80 batches × 10k input / 1k output | Sonnet 5.5 | about $8–20 |
| L3 final | about 30–60 decisions × 15k input / 2k output | Opus 5.5 | about $4–7 |
| **Total** | | | **≈ $15–30/day ≈ $300–600/month** |

**At $2,000 capital, that's 15–30% of the account per month.**

**Lean configuration (start here):** L1 on Haiku, L2 also on Haiku, L3 on Sonnet 5.5, and arm-only decisions. That's ≈ **$3–6/day ≈ $60–120/month** (3–6% per month). Upgrade a layer's model only when the shadow-book measurement shows its lift exceeds its cost. The full swarm makes economic sense at roughly **$20k+ capital**.

### Scalping universe for single stocks
- ATM option spread ≤ $0.03 at the nearest weekly expiry, and option volume > 20k contracts/day. This is typically mega-caps and the highest-volume momentum names; the list is refreshed daily.
- 1 contract (about $100–500 of premium at the ATM nearest weekly) is sized against §16's 1–2% risk-per-trade rule.
- The armed-playbook TTL keeps scalps tied to *fresh* reasoning. Nothing trades on a thesis older than its expiry.

### Build order
1. Event bus + playbook table + fast engine running SC1–SC4 on hand-armed tickers (paper).
2. L3 only (one evaluator arming from raw events).
3. Add L1 researchers.
4. Add L2.

Each step must beat the previous one in the shadow-book comparison before the next is built.

---

## 19. Probability engine (meta-labeling): score every idea with P(win) and expected value

### Concept
The "primary" sources propose a trade and its side: researcher cards, playbook triggers, and SC/S strategies. A separate **statistical model** then estimates whether *this particular* proposal will win and how much is at stake. This is **meta-labeling** (López de Prado). The primary decides direction; the meta-model decides **take/skip and size**.

A trained model scores in **microseconds**, unlike an LLM (§18). So it can gate **every individual scalp in real time**, which is the piece the LLM hierarchy couldn't do.

### What it outputs per proposal
```
p_win            calibrated probability the trade hits target before stop/time
p_win_lower      lower bound (e.g. 10th percentile) given sample uncertainty
avg_win, avg_loss  expected $ given win / loss (from the same setup's history)
EV = p·avg_win − (1−p)·avg_loss − costs        (also EV_lower using p_win_lower)
```
**Decision rule (deterministic):**
- Take the trade only if **EV_lower > 0**.
- Size = **¼ Kelly**, capped by the risk engine.
- For a win/loss ratio b = avg_win/avg_loss: Kelly f\* = p − (1−p)/b.
  - Example: SC1 with p = 0.45, b = 2 gives f\* = 0.175, so ¼ Kelly is 4.4% of equity at risk.
  - The §16 cap of 1–2% binds first. **Caps always win.**

### Labels: define "win" exactly as live trading does
**Triple-barrier labeling** (target / stop / time limit) uses the same bracket parameters as live exits. Every proposal gets an outcome, including ones we **didn't** take, because shadow trades (§17/§18) are labeled too. Without that, the model only learns from trades that were already filtered (selection bias).

### Models (start simple, add complexity only if it beats the simpler one out-of-sample)
| Stage | Model | Why |
|---|---|---|
| Cold start | **Bayesian Beta-binomial win rates** per (setup × ticker group × regime), shrunk toward the setup's backtest prior | Honest with tiny samples, and gives credible intervals for free |
| Once there are ≥ 500 labeled outcomes | **Logistic regression → gradient boosting (LightGBM)** on features, then **isotonic calibration** | Captures interactions such as time of day × IV rank × relative volume |
| Always | Walk-forward retraining (weekly) with time-decayed weights (e.g. 60-day half-life) | Markets change. Old edges fade |

**Features** must be known at decision time, with no leakage. They include:
- setup id; ticker liquidity/spread
- relative volume; distance to VWAP and levels
- IV rank and implied move; time of day; regime (S4)
- SPY/sector move
- news triage scores and researcher confidence
- **the researcher's leaderboard score**, and L2/L3 verdicts (as features, not overrides)

### Sample size: why the lower bound matters
| Trades observed | Win rate seen | 95% interval |
|---|---|---|
| 30 | 60% | about 42–77% |
| 300 | 60% | about 54–66% |

Early on the lower bound sits near break-even, so the engine trades **small or not at all** until evidence builds. That is the intended behavior.

### Monitoring and self-protection
- **Calibration check:** live Brier score and a reliability plot. When the model says 60%, do those trades win about 60%?
- If live calibration drifts beyond tolerance over the last N trades, **automatically halve size**. If it persists, disable that setup until retrained.
- Track EV-predicted vs realized P&L per setup weekly. A setup whose realized EV is below predicted for 4 straight weeks is paused.

### Where it plugs in (also cuts LLM cost)
```
L1 SignalCard ──► probability engine ──(EV_lower > 0 only)──► L2 ──► L3 arms playbook
Fast engine trigger on armed ticker ──► probability engine ──► size ──► risk engine ──► order
```
- Scoring L1 cards **before** L2/L3 filters out low-EV ideas, so the expensive Sonnet/Opus layers only see promising ones.
- L2 and L3 receive `p_win`, `EV` and the interval, and rank by EV.

### Libraries
`scikit-learn` (logistic, isotonic calibration), `lightgbm`, `scipy.stats` (Beta posteriors), `mapie` or bootstrap (intervals), `mlflow` or a simple `models/` registry (versioned models; every trade records which model version scored it).

---

## 20. Funding and growth targets

### Monthly operating costs (the hurdle)
| Item | Lean system | Full swarm |
|---|---|---|
| Real-time OPRA data (Alpaca Algo Trader Plus) | $99 | $99 |
| LLM (§18 lean vs full) | $60–120 | $300–600 |
| Hosting (US-East VM) | $5–10 | $10–20 |
| News | $0 (verify) | $0 (verify) |
| **Total** | **≈ $170–230/month** | **≈ $410–720/month** |

During development, add historical options quote data for backtests (estimated $30–80/month, unverified).

### Hurdle as % of capital (lean system, about $200/month)
| Capital | Costs per month | Return needed per year just to break even |
|---|---|---|
| $2,000 | 10% | about 120% → **not viable** |
| $5,000 | 4% | about 48% → very hard |
| $10,000 | 2% | about 24% → hard but plausible |
| $25,000 | 0.8% | about 10% → reasonable |

The full swarm needs **about $50k+** for its costs to drop to about 1% per month.

### Recommendation
1. **Development and paper trading (about 3–4 months): fund $0.** Budget about **$600–1,000 total** for data, LLM and backtest data. This is the real first spend.
2. **Go-live (only if the gates pass): fund $10,000** with the lean system. The **minimum is $5,000**:
   - $2,000 is the legal floor for margin and day trading.
   - The extra keeps a 35–50% drawdown from pushing the account below $2,000, which would switch off day trading.
   - Only fund money you can afford to lose entirely.
3. **Ramp:** live risk starts at **25% of the planned size**. Size doubles after each block of 100 trades whose realized expectancy is ≥ 70% of the paper/model prediction, up to 100%.
4. **Scale the system with capital:** add L2/L3 model upgrades and the full swarm only once capital and measured lift justify the cost (§18).

### Growth targets (net of all costs)
| Tier | Annual | Monthly average | Comment |
|---|---|---|---|
| Baseline goal | 10–20% | about 1–1.5% | Beats or matches the index after costs. A real success |
| Stretch | 30–50% | about 2–3.5% | Elite if *sustained*. Expect drawdowns of 25–40% along the way |
| Not a target | 10%+ per month | — | Implies hidden tail risk. If paper results show this, suspect a bug or overfitting |

Expect **30–40% of months to be losing months**, even in the good cases.

### What $10,000 becomes (illustrative compounding, no deposits)
| Net annual return | Year 1 | Year 2 | Year 3 |
|---|---|---|---|
| +15% | $11,500 | $13,225 | $15,209 |
| +30% | $13,000 | $16,900 | $21,970 |
| +50% | $15,000 | $22,500 | $33,750 |
| −30% (no edge plus costs) | $7,000 | $4,900 | $3,430 (kill switch would stop it long before this) |

**Stop rule:** if live results put the account at −35% from peak, the system halts (§15). If the probability engine's live calibration fails (§19), the strategy track is retired and does not get re-funded.

---

## 21. Running locally (i9 / 128 GB RAM / RTX 4090) with a minimal-spend ramp

### What the hardware changes
- **The 4090 (24 GB VRAM) can run open-weight LLMs locally**: 8–32B instruction models, 4-bit quantized, served with vLLM or Ollama. Batched throughput is easily enough for about 3,000 L1 researcher wakes per day.
  - Move **L1 researchers and news triage to the local model ($0 per token)**. Keep Claude for L2/L3 judgment.
  - This cuts the LLM bill from about $60–120 to **about $20–60/month** in the lean configuration.
  - Local-model quality is lower. The §18 leaderboard and shadow books decide whether it's good enough, and any researcher slot can be switched to Haiku 5.5 if local underperforms.
- **128 GB RAM + polars/Parquet:** months of options quote history can be backtested in memory.
- **CPU:** LightGBM training and walk-forward runs in minutes. No cloud compute needed.
- **Storage:** budget 1–2 TB of NVMe for quote history (Parquet, partitioned by date and underlying).

### LLM interface (provider-agnostic)
`agents/llm.py` defines a single `LLMClient` protocol with backends:
- `anthropic`: Anthropic SDK; `claude-agent-sdk` for deep-research jobs.
- `local`: an OpenAI-compatible endpoint served by vLLM or Ollama on `localhost`.

Every call records backend, model, tokens, latency and cost in `agent_runs`, so the leaderboard compares backends on equal terms.

### Software stack
| Layer | Choice |
|---|---|
| OS | **Windows 11 + WSL2 Ubuntu 24.04 with systemd enabled**, or native Linux (dual-boot) for the most reliability |
| Python | 3.12, managed with `uv`. The repo is new, not this Rails repo |
| Services | `docker compose` with `restart: unless-stopped`: `engine`, `api` (FastAPI, bound to 127.0.0.1), `watchdog`, `llm` (vLLM container with the NVIDIA container toolkit, or Ollama) |
| DB | SQLite (WAL mode) for trading state. Parquet files for market history |
| Remote access | Tailscale to reach the dashboard and `/halt` from your phone. **Never open a port on the router** |
| Alerts | ntfy or Telegram push for fills, halts, errors and the daily summary |

### Home-machine reliability checklist (a desktop isn't a server)
- **Windows Update:** set Active Hours to 08:00–17:00 ET and pause updates during live phases. Apply updates on weekends.
- **Power:** "never sleep" and the high-performance plan. A **UPS (about $150)** rides through short outages and allows a clean shutdown.
- **Network:** wired Ethernet. Optionally, a phone hotspot as failover.
- **Clock:** enable NTP sync (chrony in WSL, `w32tm` resync on Windows). The staleness checks depend on it.
- **External dead-man switch:** a free cloud cron job (e.g. a GitHub Actions schedule) checks a heartbeat every 5 minutes during market hours. If the heartbeat is missing, it **flattens positions through the broker API** and alerts you. This is essential for a home setup.
- **Secrets:** paper and live API keys live in separate `.env` files that are never committed. Live keys are loaded only when `MODE=live` *and* a confirmation flag is set.
- **Backups:** nightly copy of the SQLite DB and models to cloud storage or another drive.

### Minimal-spend ramp
| Phase | Duration | What runs | Spend |
|---|---|---|---|
| **A. Build + plumbing** | weeks 1–4 | Engine, risk engine, Alpaca **paper** account on the **free** data plan (fine for testing code, *not* for judging strategies). Local LLM. Backtests on Alpaca historical bars | **≈ $0–20/month** (a little Claude API for L3 tests) |
| **B. Validation paper** | 6–12 weeks | Same engine on **real-time OPRA ($99)**. All strategies run in paper with shadow books. The probability engine collects labels. Optional historical options-quote data for quote-level backtests (est. $30–80, unverified) | **≈ $120–200/month** |
| **C. Small real trades ("confidence")** | 2–3 months | Fund **$500–1,000** at Alpaca. Its accounts are margin accounts, and since June 2026 non-leverage accounts can day trade with no PDT designation (see §22). **1 contract, at most $20–40 risk per trade, at most 3 trades/day.** Only setups that passed the B gates. Live is compared against paper side by side | Data + LLM ≈ $120–160/month. **Capital at risk: treat losses here as tuition** |
| **D. Scale** | after C passes | Add capital toward $5k–10k (§20). Ramp size 25% → 100% per §20 | Per §20 |

**Gate C → D:** ≥ 100 live trades. Live expectancy ≥ 70% of paper expectancy. Median slippage within the model. No risk-rule violations. Max drawdown within the tested range.

**With local LLMs, the lean monthly hurdle drops to about $130–180.** On $10k that's about 1.5% per month (§20 recalculated).

---

## 22. Correction: $2,000 is not an options minimum

- **There is no regulatory minimum to trade options.** Long calls and puts (Level 2) can be traded with any balance that covers the premium.
- **$2,000 is the margin threshold for borrowing / leverage.** At Alpaca, $2,000 is now the minimum equity for **4× intraday buying power** on leverage-enabled accounts. That replaced $25,000 when PDT was retired on 2026-06-04. **This plan doesn't use leverage**, so it doesn't need it.
- **Alpaca implemented the PDT replacement on 2026-06-04.** For non-leverage margin accounts the PDT designation is abolished, and orders that would create a margin deficit are rejected. So a **$500–1,000 Alpaca account can day trade and scalp long options** with buying power equal to equity. This supersedes the earlier "a cash account can't scalp" and "$2,000 needed to day trade" statements (§0, §16, §21) **for Alpaca specifically**. Other brokers have until 2027-10-20 to implement, so check each one.
- **Where $2,000 may still matter:** Level 3 (spreads) approval criteria aren't published and may consider equity. Apply and see. The §20 cost-hurdle math, not regulation, is what argues for $5k–10k before scaling.
- **Note for the order manager:** Alpaca removed the `pattern_day_trader`, `daytrade_count` and `daytrading_buying_power` API fields. Use `buying_power`.

---

## 23. Other instruments: where each fits

**No instrument "yields more" by itself.** Return = edge × size × turnover − costs. What differs is the **cost per unit of edge**, the **available leverage** (which amplifies wins and losses alike), and how well the instrument **matches each strategy track**.

### Comparison (fees approximate; verify per broker)
| Instrument | Costs | Capital / leverage | Hours | Fit with our tracks |
|---|---|---|---|---|
| **Equity/ETF options** (current) | $0–0.70/contract + spread. Time decay and IV crush for buyers | Premium only (defined risk for buyers) | 9:30–16:00 ET | **Best for premium selling** (S1, S3, credit spreads), the only way to harvest the volatility risk premium |
| **Micro index futures** (MES = $5/S&P pt, MNQ = $2/Nasdaq pt) | About $0.39–0.75/side commission, about $1.50–2.30 round-turn all-in, plus a 1-tick spread ($1.25 on MES) | Day margin as low as about $50 at some brokers, about $2.9k overnight. **Very high leverage**: 1 MES ≈ $33k of S&P exposure | Nearly 23h, Sun–Fri | **Best for the scalping track.** Costs are about equal to ATM 0DTE options for the same S&P move, but there's **no time decay, no strike/IV selection, deeper liquidity**, it can rest passive limit orders, and gets 60/40 tax treatment |
| **Micro commodity futures** (MCL crude = $1/tick, MGC gold = $1/tick) | Similar to MES | Similar | Nearly 23h | **News/event track:** EIA inventory (Wednesdays), OPEC and geopolitics are LLM-readable catalysts. Noisier, with overnight gap risk |
| **Stocks / ETFs** | $0 commission, penny spreads, fractional shares (Alpaca) | 1×, up to 4× intraday at ≥ $2k (not used) | Plus extended hours | **Often better than options for the news-drift track** (§17). No time decay or IV crush, the full move counts, and costs are tiny. Use options only when the predicted move is much larger than the implied move |
| **Crypto (spot)** | Taker fees typically ≈ 0.1%+ per side at retail, too high for scalping | 1× spot (US perps limited) | 24/7 | Low priority. Needs an always-on bot, has weaker news structure, and higher fees |
| **Spot FX** | Spread-based (about 1 pip majors) | Up to 50:1 in the US | 24/5 | Low priority. Very efficient market, hard edge for retail |
| **Event contracts (Kalshi)** | Taker fee ≈ ⌈0.07 × contracts × P × (1−P)⌉ (verify) | Fully paid, max loss = price | Market-specific | **Experimental fit for the LLM swarm.** Its core skill is *forecasting* questions (CPI, Fed, jobs prints). Smaller liquidity, and the fee is highest near 50¢ |

### Recommendation: re-map tracks to their best instrument
| Track | Primary instrument | Why |
|---|---|---|
| Scalping (§16, §18 fast engine) | **MES/MNQ micro futures** (options variant kept as a paper benchmark) | No decay, simplest execution, deepest liquidity, nearly 24h |
| News-driven single stocks (§17) | **Shares** by default. Options only when edge = predicted − implied move is large | No IV crush, near-zero cost |
| Premium selling / regime (§14, §15) | **Options (XSP/SPY/IWM)** | The only way to harvest the volatility risk premium |
| Commodities news | **MCL/MGC micros** (phase 2, after MES is proven) | Scheduled, readable catalysts |
| LLM forecasting experiment | **Kalshi** economic contracts (paper first) | Tests the swarm's forecasting directly, separate from execution skill |

### Futures-specific risk rules (leverage is the danger)
- **Size by stop distance, never by margin.** Example: with $1,000 equity and 2% risk, the stop budget is $20, so 1 MES with a 4-point stop. Margin may *allow* 20 contracts. The risk engine allows **1**.
- **No overnight futures positions** in the scalping track: flat by 15:55 ET (or the session-specific cutoff).
- **Max contracts = floor(equity × risk% / (stop pts × $/pt)).** Hard cap of 2 MES until equity is above $5k.
- **Avoid contract roll weeks** (quarterly). Trade the front month after roll date.
- **No trading 2 minutes either side of CPI/FOMC/NFP**, unless a dedicated event strategy has passed its gates.

### Broker impact
- **Alpaca doesn't cover futures.** Keep Alpaca for paper trading, stocks/options data and news.
- **tastytrade** covers options and futures (and has an API and sandbox), so it could be the **single live broker**.
- Tradovate, NinjaTrader and IBKR have lower futures commissions (about $0.39–0.62/side vs about $0.75) and are worth it once volume is high.
- **Futures data:** non-professional CME real-time data through brokers typically costs a few dollars/month (verify), far less than OPRA's $99.
- The `Broker` interface (§3) gets a `FuturesBroker` adapter. `risk.py` gains contract specs (point value, tick, session times, roll calendar).
