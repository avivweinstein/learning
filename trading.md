# Trading & Stock Markets — A Primer

> A from-scratch mental model for someone who has never traded but wants to understand
> what's actually happening when you click "Buy" on Fidelity, how that compares to a
> high-frequency trading firm in New Jersey, and what a Wall Street trader actually does
> all day. Heavy on intuition, light on math.

---

## 1. What is a stock?

A **stock** (also "share" or "equity") is a tiny ownership slice of a company. If a company has issued 1,000,000 shares and you own 100, you own 0.01% of the company.

Owning a share entitles you to:
- A pro-rata claim on the company's future earnings (paid as **dividends**, or reinvested to grow the business).
- Voting rights at shareholder meetings (usually 1 vote/share — though some companies have dual-class structures, e.g. Meta, Google).
- A pro-rata claim on whatever's left if the company is liquidated (after debt holders are paid).

You do **not** own any specific physical asset of the company. You own a financial claim.

**Why do companies issue stock?** To raise money without taking on debt. They sell ownership to the public (via an **IPO** — Initial Public Offering) and use the cash to fund operations, R&D, acquisitions, etc. After the IPO, the shares trade between investors on the open market — the company itself is no longer involved in those transactions.

---

## 2. What is a stock market / stock exchange?

A **stock exchange** is a regulated marketplace where shares of public companies are bought and sold. Major U.S. exchanges:

- **NYSE** (New York Stock Exchange) — older, hybrid floor + electronic.
- **Nasdaq** — fully electronic, tech-heavy listings (AAPL, NVDA, MSFT, GOOG).

The "stock market" as a concept is the aggregate of all these exchanges plus off-exchange venues (more on those later).

### How a trade actually happens

Every exchange runs a **central limit order book (CLOB)** for each stock. Two sides:

- **Bids** — people who want to buy, sorted highest price first.
- **Asks** (or "offers") — people who want to sell, sorted lowest price first.

The highest bid and lowest ask are called the **best bid** and **best ask** (or the **NBBO** — National Best Bid and Offer, the best across all U.S. exchanges). The difference between them is the **bid-ask spread**.

```
Example order book for NVDA:

  Asks (sellers)
    $920.05  x 500
    $920.03  x 200
    $920.01  x 100  <- best ask
  --- spread = $0.02 ---
    $919.99  x 300  <- best bid
    $919.98  x 700
    $919.95  x 100
  Bids (buyers)
```

When a buy order arrives at $920.01 or higher, it **crosses the spread** and matches with the best ask — a trade executes. Price-time priority: best price wins; among same-price orders, earliest wins.

### Order types you'll meet

- **Market order** — "Buy/sell now at whatever price." Guaranteed execution, NOT guaranteed price. Crosses the spread immediately.
- **Limit order** — "Buy at $X or better, sell at $Y or better." Guaranteed price (or better), NOT guaranteed execution. Sits in the book until matched, canceled, or expired.
- **Stop order** — Becomes a market order once the stock crosses a trigger price. Used for stop-losses ("sell if it drops to $X to limit my downside").
- **Stop-limit** — Same trigger, but converts to a limit order instead of market.

**Default advice for retail**: use limit orders. Market orders on illiquid stocks can fill at terrible prices.

---

## 3. Long vs. Short

Two directions you can bet:

### Long (the normal way)
You buy a stock expecting it to go up. Your max loss is what you paid (it can only go to zero). Your max gain is unbounded.

> Buy NVDA at $900. Sell at $1000. Profit = $100/share.

### Short (the inverted way)
You **borrow** shares from someone else (your broker arranges this), sell them immediately at the current price, and hope the price drops so you can buy them back cheaper and return them. You pocket the difference.

> Borrow + sell NVDA at $900. Later buy back at $800. Return shares. Profit = $100/share.

**Risks of shorting**:
- Your max loss is **unbounded** — if NVDA goes from $900 to $3000, you lose $2100/share. The stock can keep going up forever.
- You pay a **borrow fee** (interest on the borrowed shares) for as long as you hold the short.
- The lender can demand the shares back at any time (a **forced buy-in**).
- A **short squeeze** can happen — if the stock spikes, shorts panic-cover, which drives the price up further, which forces more shorts to cover, etc. (See: GameStop, January 2021.)

Short selling is how you bet *against* a company, hedge a long portfolio, or profit from overvalued stocks.

---

## 4. Options: Calls and Puts

An **option** is a contract giving you the **right, but not the obligation**, to buy or sell a stock at a specific price (the **strike**) on or before a specific date (the **expiration**).

One option contract = 100 shares (in the U.S., for stock options).

You **pay a premium** upfront to buy the option. That premium is your max loss as a buyer.

### Call option — the right to BUY

A **call** gives you the right to *buy* 100 shares at the strike price. You buy calls when you think the stock will go **up**.

> NVDA is at $900. You buy 1 call with strike $950, expiring in 30 days, for a $20 premium ($20 × 100 = $2000 total).
>
> - If NVDA goes to $1000 by expiration: your call is worth $50/share intrinsic value ($1000 − $950). You make $50 − $20 = $30/share, or $3000 total. (150% return on your $2000.)
> - If NVDA stays below $950 at expiration: your call expires worthless. You lose the full $2000 premium. (-100%.)

Calls are leveraged bullish bets.

### Put option — the right to SELL

A **put** gives you the right to *sell* 100 shares at the strike price. You buy puts when you think the stock will go **down** (or to hedge a long position).

> NVDA is at $900. You buy 1 put with strike $850, expiring in 30 days, for a $15 premium.
>
> - If NVDA drops to $750: your put is worth $100/share intrinsic. Profit = $85/share = $8500. (~567%.)
> - If NVDA stays above $850: put expires worthless. Lose $1500 premium.

### Why options exist (use cases)

1. **Leverage** — small premium controls 100 shares' worth of exposure.
2. **Hedging** — own NVDA long, buy puts as insurance against a crash.
3. **Income** — sell (write) covered calls against shares you own to collect premium.
4. **Defined-risk speculation** — your max loss as a buyer is the premium, period.

### Why options are dangerous

- Most options expire **worthless**. The clock works against the buyer (**theta decay**).
- Pricing depends on **volatility (IV)**, time to expiry, interest rates, dividends — not just stock direction. You can be right on direction and still lose if IV crashes.
- **Selling** options (the other side of the trade) can have unbounded losses — selling a naked call has the same risk profile as shorting the stock, but worse.

If you're new: trade options only after you understand Black-Scholes intuition, the Greeks (delta, gamma, theta, vega), and have lost money paper-trading first.

---

## 5. The basic trading vocab cheat sheet

| Term | Meaning |
|---|---|
| **Bid** | Highest price a buyer will pay right now |
| **Ask / Offer** | Lowest price a seller will accept right now |
| **Spread** | Ask − Bid. Tight = liquid stock. Wide = illiquid. |
| **Volume** | Shares traded in a period |
| **Liquidity** | How easily you can buy/sell without moving the price |
| **Volatility** | How much the price swings (often quoted as annualized %) |
| **Market cap** | Share price × shares outstanding. The total value of the company. |
| **P/E ratio** | Price / Earnings per share. Rough valuation metric. |
| **Dividend yield** | Annual dividend / share price |
| **ETF** | Exchange-Traded Fund. A basket of stocks that trades like a single stock (e.g. SPY = S&P 500, QQQ = Nasdaq-100) |
| **Index** | Theoretical basket used as a benchmark (S&P 500, Dow, Nasdaq Composite) — not directly tradable; you trade ETFs that track it |
| **Margin** | Borrowing money from your broker to buy more stock than you have cash for. Amplifies gains AND losses. |
| **Short interest** | % of float being shorted. High SI = potential squeeze setup. |
| **Float** | Shares actually available to trade (excludes locked-up insider shares) |

---

## 6. How trading on Fidelity actually works

Fidelity is a **retail broker**. Here's what happens when you press "Buy 100 shares NVDA at market" in the app:

### Step 1 — Order leaves your phone
Your tap hits Fidelity's servers in the cloud. Fidelity does pre-trade risk checks: do you have cash/buying power? Is this stock allowed in your account type? Etc.

### Step 2 — Order routing
Fidelity does **NOT** send your order directly to the NYSE or Nasdaq. Instead, it routes the order through one of:

- **Market makers / wholesalers** (Citadel Securities, Virtu, Susquehanna, etc.) — these firms pay Fidelity for the right to fill retail orders. This is **payment for order flow (PFOF)**. The wholesaler executes your order internally (off-exchange) at a price equal to or better than the NBBO.
- **Dark pools** — private off-exchange venues for large orders.
- **Public exchanges** — NYSE, Nasdaq, IEX, etc.

Fidelity is somewhat unusual in that it does **not accept PFOF on equities** (it does on options). So for stocks, your order is more likely to go to exchanges or wholesalers without a kickback. You can read the routing breakdown in your monthly **Rule 606 disclosure**.

### Step 3 — Execution
The wholesaler or exchange matches your order against a counterparty. You get a **fill** at some price (e.g., 100 shares filled at $920.005 — slightly better than the $920.01 ask, because the wholesaler captured a fraction of the spread and gave you a tiny price improvement).

This whole round trip takes ~milliseconds, but to you it feels instant.

### Step 4 — Clearing & settlement
The trade clears through the **DTCC** (Depository Trust & Clearing Corp). Settlement is now **T+1** (one business day after trade date) — it used to be T+2 until May 2024. The shares appear in your account next business day; cash settles the same way.

In practice you can use the proceeds of a sale to buy something else immediately — Fidelity fronts you the buying power. But you cannot withdraw the cash to your bank until it's actually settled.

### What Fidelity is NOT doing
- Not "calling someone on the floor of the NYSE" — that hasn't really been a thing for 20+ years.
- Not letting you trade microseconds faster than anyone else.
- Not giving you exotic order types or leverage normally available to institutions (without specific account upgrades).

### What you pay
- **Commissions**: $0 for stocks/ETFs at Fidelity, Schwab, Robinhood, etc. since ~2019.
- **Spread cost**: implicit. You buy at the ask, sell at the bid — that gap is your friction.
- **Options contract fees**: ~$0.65/contract at Fidelity.
- **Margin interest**: if you borrow from Fidelity to trade on margin, you pay ~10%+ APR (negotiable for big balances).

---

## 7. How an HFT firm differs from you on Fidelity

A **high-frequency trading** (HFT) firm — Citadel Securities, Virtu, Jump, Jane Street, Hudson River Trading, Tower Research, Two Sigma Securities — operates at the *opposite end* of the food chain. Same stock market, fundamentally different game.

### What HFT firms actually do

Most HFTs are **market makers**. They continuously post both bids and asks on thousands of stocks, profiting from the spread. They are not trying to predict whether NVDA is "a good company" — they are trying to capture half a cent thousands of times a second across the entire market.

Other HFT strategies:
- **Statistical arbitrage** — exploit short-term mispricings between correlated assets (e.g. ETF vs. its underlying basket; one stock vs. its index futures).
- **Latency arbitrage** — see a price update on one venue microseconds before another and trade against the stale quote.
- **Index arbitrage** — when SPY drifts from the underlying S&P 500 basket, arb it back into line.

### What's different from you on Fidelity

| Dimension | You on Fidelity | An HFT firm |
|---|---|---|
| **Latency** | ~100ms+ phone-to-fill | **Microseconds**. Single-digit μs round-trip is normal. |
| **Co-location** | Wherever your phone is | Servers physically in **the same data center as the exchange's matching engine** (Equinix NY4 in Secaucus NJ, Mahwah for NYSE, Carteret NJ for Nasdaq). Cable lengths are equalized to the inch so no rack has an unfair advantage. |
| **Network** | Comcast / Verizon Wi-Fi | Microwave/millimeter-wave radio links between Chicago and NJ, FPGAs decoding market data on the wire, custom NICs that bypass the kernel |
| **Hardware** | iPhone | FPGAs, custom ASICs, kernel-bypass networking, hand-tuned C++/Rust on bare metal, sometimes pure hardware logic with no CPU in the hot path |
| **Order rate** | 1 order per coffee | Millions of orders/sec across the firm. Most are canceled microseconds later. |
| **Holding period** | Minutes to decades | Milliseconds to seconds, occasionally minutes. Frequently flat overnight. |
| **Edge per trade** | Hopes to make 5-50% on a thesis | Targets fractions of a cent per share, scaled to billions of shares/year |
| **Data feeds** | Delayed 15-min quotes (free) or real-time NBBO | **Direct feeds** from each exchange (ITCH, OUCH, PITCH) — raw, faster, with full order-book depth, costing $$$/month per venue |
| **Risk management** | Mental ("don't blow up my Roth") | Pre-trade risk gates in FPGA, kill switches, real-time P&L monitoring, drawdown limits |
| **Capital** | Your savings | Hundreds of millions to billions |
| **Regulation** | Reg T, PDT rule | Reg NMS, Reg SCI, MiFID II, exchange rules, CAT reporting, plus their own internal compliance |

### The PFOF connection

When your Fidelity buy order gets routed to Citadel Securities, the HFT firm is on the *other side* of your trade. They fill you at NBBO or slightly better, and they make money because:
- Retail flow is **uninformed** (no insider info), so it's safer to trade against than institutional flow.
- They can immediately hedge or offset your trade against their book and capture spread.

This is why retail brokers can offer "free" trading. The HFTs subsidize it via PFOF. Critics argue this creates conflicts of interest; defenders argue retail gets price improvement vs. just hitting the exchange.

### Can you compete with HFTs?
**No.** Not on speed, not on data, not on hardware. The good news is you don't have to. They make pennies; you can target percentages over weeks. Different timeframes, different games.

---

## 8. How a Wall Street trader differs from both

"Wall Street trader" is a catch-all. The job depends entirely on the desk.

### Sell-side traders (at investment banks: Goldman, Morgan Stanley, JPM, etc.)

These traders work at **broker-dealers**. Their clients are big institutions (pension funds, mutual funds, hedge funds, sovereign wealth funds).

Two main flavors:

**Flow trader / market maker (sell-side)**:
- Provides liquidity to clients. A pension fund wants to sell 5 million shares of MSFT — they call Morgan Stanley's trading desk.
- The trader quotes a price (bid/ask), takes the trade onto the bank's book, then has to **work it out** — gradually unload it without moving the market against them.
- Skills: pricing, risk management, relationships, knowing who's buying/selling what.
- Increasingly automated — humans oversee algos for vanilla equities; humans still dominate exotic products (corporate bonds, swaps, structured notes).

**Prop trader (now mostly extinct at banks post-Dodd-Frank/Volcker)**:
- Trades the bank's own capital. Mostly migrated to hedge funds and prop shops after 2010 regulation banned banks from doing this with deposits.

### Buy-side traders (at hedge funds, mutual funds, asset managers)

These traders **execute** trades on behalf of the firm's portfolio managers. The PM decides "buy 1M shares of NVDA over the next 3 days." The trader figures out **how**:
- Use a VWAP/TWAP algo to slice the order over time?
- Send chunks through a dark pool to avoid signaling?
- Negotiate a block trade with a sell-side desk?
- Time it around the market open vs. close?

Their job is **minimizing market impact** — buying 1M shares all at once would spike the price and cost millions in slippage.

### Hedge fund portfolio managers / discretionary traders

These are the "Wall Street traders" of pop culture — making bets, hours-to-months timeframes, fundamental research, macro views. The "Big Short" guys. The "Margin Call" guys. They're not pressing F12 to send orders — they tell their execution traders what they want and the execution traders work it.

### Quant traders / quant researchers

Increasingly the dominant model: build statistical models, backtest, deploy systematically. Renaissance, Two Sigma, DE Shaw, Citadel's Global Equities. Closer in spirit to HFT than to a 1980s floor trader, but timeframes are usually longer (hours to weeks).

### Comparison

| Dimension | You on Fidelity | Wall Street trader (sell-side flow) | HFT firm |
|---|---|---|---|
| **Goal** | Make money for yourself | Make money for the bank by serving clients + spread | Make money via tiny edges at huge scale |
| **Counterparties** | Whoever's on the other side | Specific named institutions you have phone numbers for | Anonymous flow on exchanges |
| **Capital** | Personal | Bank's balance sheet (billions) | Firm's capital (millions to billions) |
| **Decision speed** | Seconds to days | Seconds to hours | Microseconds |
| **What you watch** | Your portfolio, news, charts | Order flow, client positions, your inventory, your risk limits | Tick data, latency to each venue, signal P&L |
| **Tools** | Fidelity app, maybe a charting site | Bloomberg terminal, RFQ platforms, internal OMS, voice line, IB chat | Custom C++/FPGA stack, proprietary feeds, ML models |
| **Compensation** | You keep your gains | Salary + bonus tied to desk P&L | Salary + bonus tied to strategy P&L |

---

## 9. If you actually want to start trading

### Reasonable starting setup

1. **Open a brokerage account.** Fidelity, Schwab, or Vanguard are solid for U.S. retail. Robinhood works but has had operational issues (e.g. GME halt, 2021).
2. **Fund it with money you can afford to lose.** Seriously.
3. **Start with index ETFs** (VTI, VOO, QQQ) or individual blue-chip stocks before touching options or shorts.
4. **Use limit orders, not market orders.** Especially on anything that isn't AAPL/MSFT-tier liquid.
5. **Track everything.** Date, ticker, price, size, thesis. Review your trades quarterly. You will be surprised how often you're wrong but feel like you're right.

### Account types worth knowing

- **Cash account** — you can only buy with settled cash. Safest. No margin.
- **Margin account** — you can borrow from the broker. Required for shorting. Subject to **Reg T** (50% initial margin) and the **Pattern Day Trader (PDT) rule** (if you make 4+ day-trades in 5 business days with <$25k account, your margin account gets restricted).
- **Roth IRA / Traditional IRA** — tax-advantaged. Long-term holdings. Most brokers don't allow shorting or naked options in IRAs.
- **Options approval levels** — Fidelity gates options strategies behind tiers. Level 1: covered calls. Level 2: long calls/puts. Level 3+: spreads, naked options. You apply and they approve based on stated experience and account size.

### Tax basics (U.S.)

- **Short-term capital gains** (held < 1 year) — taxed as ordinary income. Brutal at high brackets.
- **Long-term capital gains** (held ≥ 1 year) — taxed at 0/15/20% federal. Massively better.
- **Wash sale rule** — if you sell at a loss and rebuy "substantially identical" security within 30 days, the loss is disallowed (added to basis). Easy to trip on accident.
- Brokers issue a **1099-B** in January for the prior year. Import to TurboTax / give to your CPA.

### Things that destroy retail traders, in rough order of frequency

1. **Overtrading** — every trade has a spread cost. Trading 50x/day means paying the spread 50x/day.
2. **Leverage / margin** — fine in theory; in practice, gets you margin-called at the worst time.
3. **Selling naked options** — limited upside, unlimited downside. Looks like easy income for months until one event blows up the account.
4. **Concentration** — putting >20% of net worth in any single stock, or borrowing against your house to buy stocks.
5. **Confusing a bull market with skill** — everyone is a genius from 2009-2021. Markets test you in the bad years.

### Books worth reading (and the reason)

- **"A Random Walk Down Wall Street"** — Burton Malkiel. The case for indexing. Read this first.
- **"The Intelligent Investor"** — Benjamin Graham. The classic value investing primer.
- **"Flash Boys"** — Michael Lewis. Pop-readable look at HFT and dark pools (somewhat one-sided but accessible).
- **"Trading and Exchanges"** — Larry Harris. Dense but the canonical market microstructure textbook.
- **"Options, Futures, and Other Derivatives"** — John Hull. The textbook on options pricing.
- **"More Money Than God"** — Sebastian Mallaby. History of hedge funds.

---

## 10. TL;DR

- A **stock** is a share of company ownership. A **stock market** is a regulated marketplace where shares trade via a continuous auction (the order book).
- **Long** = buy hoping it goes up. **Short** = borrow-and-sell hoping it goes down (unbounded risk).
- **Calls** = right to buy at a strike. **Puts** = right to sell at a strike. Limited risk for buyers, dangerous leverage for sellers.
- On **Fidelity**, you click Buy → order routes (often through wholesalers or exchanges) → fills in milliseconds → settles T+1. You pay $0 commission and an implicit spread cost.
- An **HFT firm** does the same thing but at microsecond latency, from a server colocated next to the exchange's matching engine, with custom hardware. Different game entirely.
- A **Wall Street trader** is somewhere in between, depending on the desk — sell-side market makers serve institutional clients, buy-side execution traders work big orders quietly, and discretionary PMs at hedge funds make directional bets.
- **Start simple**: index ETFs in a tax-advantaged account, limit orders only, no margin or naked options until you've watched a market downturn from the inside.
