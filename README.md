SILICON ORACLE
Compute Oracle & Multi-Venue Distribution Strategy
From benchmark methodology to a feed any perp DEX can liquidate against
Working draft · June 2026 · illustrative numbers only
Contents
Right-click and choose 'Update Field' to build the contents.
 
Executive summary
Two methodology drafts exist: the original v0.1 white paper and the revised v3 oracle methodology. v3 is strictly better — it keeps everything sound in v0.1 and fixes the one flaw that made it unusable for a perpetual: a completed-rental VWAP cannot tick fast enough to mark and liquidate a contract. v3 solves this by splitting the oracle into a fast Mark Price and a slow Index Price.
The strategic conclusion: you do not need to build a perp exchange. The matching engine, mark price, funding, liquidation and settlement are the venue's job. Your product is a continuous, manipulation-bounded, never-stale USD-per-GPU-hour price feed. To put that feed on many venues, you become a first-party publisher on the oracle networks perp DEXs already consume, and run your own signed feed for direct partners.
Bottom line: The company is the feed, not the exchange. Build one robust continuous price; rent the perp mechanics from venues; distribute through oracle networks so onboarding a new DEX is a config change, not an engineering project.

 
Part 1 — Which methodology is better
The two documents are the same project at two stages, so this is not really a contest. v3 wins because it contains the mark-price logic v0.1 structurally cannot support.
What v0.1 gets right
v0.1 is a solid benchmark / index paper. Its strengths carry forward unchanged into v3:
•	Binary verification gate — a rental is admitted or rejected by rule, never secretly volume-scaled.
•	Completed-rental VWAP as the truth layer, weighted by verified GPU-hours.
•	Thin-market controls: rental-type stratification, minimum counts, concentration caps, dispersion flags, carryforward limits.
•	Wash-lease defenses, a bounded (filter-then-weight) offer index, data-rights statuses, and benchmark-administrator governance.
The fatal flaw v3 fixes
The core problem: v0.1 specifies the price engine as a Completed-Rental VWAP. A completed rental only prints when a multi-hour lease finishes and telemetry confirms uptime. The feed updates on lease completions, not wall-clock ticks, and every print is stale by roughly the lease duration. You cannot liquidate a perp minute-by-minute off an input that arrives every few hours and looks backward.

v3 keeps the VWAP as the Index (truth) but adds a fast Mark Price from the order book to drive liquidations, plus the four things that turn a benchmark into an oracle: a published manipulation bound, basis disclosure, a volume-wash defense, and an additive (not multiplicative) composite.
Verdict
Use case	Better choice	Why
Settlement-only daily reference rate	v0.1 is sufficient	No fast path needed; VWAP cadence is fine.
Marking & liquidating a live perp	v3 (required)	Needs a continuous Mark tethered to the slow Index.

 
Part 2 — The two-tier oracle and mark-price math
v3's architecture is three feeds with three different jobs:
Tier	Feed	Role
A · fast	Mark Price	Multi-venue order-book microprice. Drives liquidations, intraday marks, funding. Updates every few seconds. Never null.
B · slow	Index Price (DLV)	Completed-Rental VWAP over a rolling window. The truth tether — the Mark may not stray beyond a clamp band from it. May carry-forward / null.
C · daily	Settlement Anchor	TradFi daily print (Ornn OCPI / Silicon Data). Once-a-day reconciliation; divergence beyond a band raises a governance flag.

Building the mark-price ticks
v3's depth-weighted microprice is under-specified and naive top-of-book is spoofable. The production-grade construction (dYdX / Binance / Hyperliquid lineage) is four layers:
1 · Per-venue impact price (not top-of-book)
Pick an impact notional Q (the size a real taker clears) and walk each venue's book so a single thin quote can't move the feed:
impact_bid_v = avg fill price to SELL Q into venue v's bids
impact_ask_v = avg fill price to BUY  Q from venue v's asks
impact_mid_v = (impact_bid_v + impact_ask_v) / 2

Size Q against your manipulation bound. Top-of-book microprice is the fallback only when depth is unavailable.
2 · Cross-venue aggregation by median, then depth-weight
A median of sources resists any single venue being pushed. Include the slow index as one voter to pull the mark toward verified truth:
candidates = { impact_mid_v for each live venue v } + { index_dlv_live }
raw_mark   = weighted_median(candidates, weight = depth_within_X%_of_mid)

3 · Smooth to kill tick jitter
A short EMA stops a single flickering quote from triggering liquidation cascades. tau is the explicit freshness-vs-resistance knob:
mark_ema_t = a * raw_mark_t + (1-a) * mark_ema_{t-1}
a = 1 - exp(-dt / tau_mark)      # tau_mark ~ 5-30s (lag budget)

4 · Clamp to the index, and guarantee non-null
This is the core liquidation-safety property — the Mark cannot run away from verified truth, and the liquidation path never returns null:
clamp_band = base_band + k*dispersion + j*(1/liquidity_score)  # ~5%..15%
mark_t = clamp(mark_ema_t,
               index_dlv_live * (1 - clamp_band),
               index_dlv_live * (1 + clamp_band))
 
if index_dlv_live is stale/null:
    widen clamp_band, raise mark_confidence_pct,
    fall back to median of venue impact_mids, keep publishing

What fires a tick: a book-change event on any venue -> recompute that venue's impact price -> re-run the median -> push through the EMA -> clamp -> emit. Funding and the liquidation engine read the clamped, smoothed value, never the raw microprice.
Caveat to stress-test first: the fast path assumes a live two-sided order book exists. In compute that's essentially Hyperbolic plus partnered books — Akash / io.net on-chain leases give you the Index, not a tickable book. If the book thins at 3am Sunday (the weekend window you're selling), the Mark degrades to a clamp band around a stale Index. Prove the feed survives the thin window before promising 24/7 liquidation coverage.

 
Part 3 — Do you need to build the exchange? (Hyperliquid / HIP-3)
No. You don't build the matching engine, mark price, funding, liquidation, margin, or settlement. The venue does. Your product is the index feed. But “just being the benchmark” does not escape the fast-tick problem — and on Hyperliquid the path is specific.
A compute perp on Hyperliquid must be HIP-3
Core (validator-listed) perps derive their oracle from a weighted median of Binance / OKX / Bybit / Gate spot prices. There is no CEX spot feed for “H100 GPU-hour,” so validators have nothing to median. A compute perp can therefore only exist as a HIP-3 builder-deployed perp, where the deployer supplies the oracle — i.e. you.
Concretely, being the benchmark means you (or a partner) deploy a HIP-3 market, stake HYPE, and call setOracle every 3 seconds with the price (plus the markPxs / externalPerpPxs inputs). Hyperliquid does the rest.
What Hyperliquid does for you — delete from v3
Their mark = your oracle price + a 150s EMA of (their book-mid minus oracle) + the median of best bid / best ask / last trade. Mark moves are clamped to 1% per tick; all prices capped at 10x start-of-day. Funding is computed off the oracle price, not the mark. So the entire mark / microprice / clamp-band apparatus is Hyperliquid's job here — you can drop the mark-as-built, the clamp band, and the whole liquidation / funding / margin stack.
What you still must build — this is the benchmark
•	A continuous, never-stale USD/GPU-hr price every 3 seconds. A few-hourly VWAP cannot satisfy setOracle; funding (keyed to your oracle) would compute against a stale price.
•	The manipulation bound. Hyperliquid's mark trusts your oracle; if it's pushable, the perp is pushable. The 1%/tick clamp limits speed, not sustained direction.
•	The externalPerpPxs guard (a slower robust reference — your DLV settlement VWAP or the daily anchor fits).
•	Data sourcing, verification, wash defenses, and clean rights — the foundation everything sits on.
The strategic fork
Model	You build	You capture
Pure benchmark / data licensor	Only the feed + oracle-grade SLAs. A third party deploys the perp.	Data-licensing revenue. Lowest infra.
HIP-3 deployer	The feed + deploy the perp yourself + stake HYPE.	Trading fees + control of the listing. More upside, more obligation.

Either way the kept engineering is identical: turn lagging completed-rental data plus a live order-book signal into a robust, manipulation-bounded, 3-second feed. The perp mechanics are rented.
 
Part 4 — Getting the feed onto many perp DEXs
You do not integrate venue-by-venue as bespoke jobs. The scalable mechanism: become a first-party publisher on the oracle networks DEXs already consume, and run your own signed feed for direct partners. Publish once per network; the network fans it out to every chain and venue downstream.
Step 0 — Choose a hybrid distribution model
•	Publisher-on-network (scale lever): push your price into Pyth / Stork / Chainlink / RedStone. Any DEX on any chain they serve references your feed ID as a config change.
•	Direct signed feed (control lever): your own low-latency endpoint + on-chain verifier for flagship venues (the HIP-3 setOracle path, app-chains like dYdX).
Build the direct feed first — everything else publishes from it.
Step 1 — Make the feed distributable, not just a number
•	Sign every tick so consumers verify provenance without trusting your API.
•	Emit a confidence interval, not a point price. Pull oracles carry price + confidence; DEX risk engines use the band as a liquidation clamp. In thin windows you widen the band instead of going stale — this is what lets you never go dark across all venues at once.
•	Fixed cadence + explicit staleness timestamp (sub-second to ~1s) so consumers can call the equivalent of getPriceNoOlderThan(age).
Step 2 — Architect as a pull oracle, not push
Perps are latency-sensitive and need on-demand prices. Publish signed updates continuously off-chain; consumers pull and verify the latest update on-chain only when they execute. This is the Pyth / Stork / Chainlink Data Streams model. Do not build a per-block push oracle — that's for lending markets, not perps.
Step 3 — Become a first-party publisher on the perp oracle networks
This is the actual “any perp DEX” lever. Target, in rough priority:
•	Stork — currently powers the largest share of perp-DEX volume. Highest-leverage perp distribution.
•	Pyth — 120+ institutional publishers, pull model, multi-chain. Contact the Pyth Data Association, stand up pyth-agent + a Pythnet validator + Solana RPC (they assist), and publish your H100 symbol; new symbols are added weekly.
•	Chainlink Data Streams — pull-based, sub-second, for venues standardized on Chainlink.
•	RedStone / Supra — extra coverage; cheap insurance since many DEXs run multi-oracle setups.
Step 4 — Stand up your own signed feed + per-chain verifier
For venues that won't take you via a network (HIP-3 deployers, dYdX-style app-chains): provide a low-latency WebSocket/REST signed feed, a small on-chain signature-verifier contract per target chain, and an SDK with docs. This also backs the HIP-3 setOracle integration.
Step 5 — Lean into multi-oracle as the strategy
Serious DEXs median two-plus providers (e.g. Chainlink + Pyth + RedStone + Supra) to avoid single-oracle dependency. Being present in two or three networks means you appear as a medianed source regardless of which stack a venue standardized on. Aim for Stork + Pyth + one more.
Step 6 — Publish the spec that makes you safe to settle on
A DEX risk team diligences a feed before listing. Have ready, as published numbers: the worst-weighted-input manipulation bound (“$X moves the index Y%”), the lag budget, confidence-interval semantics, the source/rights map, methodology versioning and governance, and your widen-confidence thin-window behavior. This converts “interesting feed” into “approved to mark a market.”
Step 7 — Go to market: prove on one venue, make the rest a config change
Launch a flagship compute perp (the HIP-3 deployment) marking against your feed as the live proof point. Then pitch other perp DEXs — and because you're already a publisher on the networks they consume, onboarding is “reference this feed ID,” not an engineering project. The network presence is the sales motion.
 
Cross-cutting constraints to design around
1.	First-party data requirement. Pyth and the credible networks only admit first-party data — exchanges, market makers, trading firms. You qualify only if your price is built from licensed marketplace data or your own venue's data with clean rights. Scraped / unclear-rights data is barred. Close at least one licensed completed-rental source (e.g. Hyperbolic, Vast.ai receipts) before you can publish reputably.
2.	One feed, one blast radius. “Publish once, reach everywhere” means a stale or manipulated print propagates to every DEX simultaneously. Your continuous engine and manipulation bound are now systemic, not local. The confidence interval is the pressure-release valve — widen it aggressively in thin windows.
3.	Basis disclosure. Verified hours are uptime-discounted, so price-per-usable-hour reads higher than contracted price — inflating the DePIN discount and reading high versus Ornn / Silicon Data. Publish both a contracted-price VWAP and the uptime-adjusted one, and state the basis loudly.
4.	Concentration vs your anchor partner. Hyperbolic would be your primary rental source, order-book source, and commercial partner at once — routinely breaching your own concentration caps. Decide whether the cap binds or is waived before the deal, and disclose it.
Sources
•	HIP-3: Builder-deployed perpetuals — Hyperliquid Docs
•	HIP-3 deployer actions — Hyperliquid Docs
•	Oracle — Hyperliquid Docs
•	Funding — Hyperliquid Docs
•	Become a Publisher — Pyth Network
•	Publish Data — Pyth Developer Hub
•	How Rapid Listings Power Perp DEX Growth at Paradex — Stork
•	Introducing Chainlink State Pricing for DEX-Traded Assets — Chainlink
•	How Oracles Keep Perp DEX Prices Fair (2026 Guide)
