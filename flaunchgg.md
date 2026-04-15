Flaunch is a memecoin launchpad built on Base that uses Uniswap v4 hooks to do four things that v2/v3 simply could not do at the AMM layer: enforce a fixed-price fair launch, redirect 100% of swap fees to the creator and community (paid in ETH, not the memecoin), automate buybacks via "Progressive Bid Walls", and tokenize creator revenue rights as a transferable NFT. Let me walk through the architecture, then the math, then the historical numbers.

## 1. The architecture
<img width="1440" height="1524" alt="image" src="https://github.com/user-attachments/assets/b78713d9-236a-4b28-9829-2e9c83b74e96" />

When a swap happens on a Flaunch pool, ETH never touches the memecoin pool directly. There are two stacked v4 pools and two hook contracts wrapping them. Here is the full request lifecycle.A few things in this picture deserve emphasis because they are not obvious from the marketing copy:

**Two pools, not one.** Every memecoin lives in a `flETH/MEME` pool, never an `ETH/MEME` pool. The Universal Router silently routes every trade through the `ETH/flETH` pool first. flETH is Flaunch's wrapped-ETH primitive (contract `0x000000000d564d5be76f7f0d28fe52605afc7cf8` on Base). Idle ETH inside flETH gets deposited into Aave v3 via `AaveV3Strategy`, which is how Flaunch can run the launchpad with zero protocol fee — the protocol pays itself out of Aave yield (~2%) instead of taxing trades.

**One hook can serve infinite pools.** This is a v4 design point that the Flaunch team exploits hard. There is one `PositionManager` hook contract (`0x51Bba15255406Cfe7099a42183302640ba7dAFDC` on Base) and it is the hook for *every* memecoin pool launched on Flaunch — over 200,000 of them. The pool-key parameters (token addresses) differ, but the hook address is constant. This is why all the protocol-level invariants (fair launch, bid wall, fee routing) hold uniformly across every Flaunch coin.

**Single-sided fair-launch liquidity.** When you flaunch a token, no ETH is provided as initial liquidity. Only memecoin is deposited — between 1% and 69% of the 100B supply, in a concentrated tick range above the current price. As people buy, they convert flETH into MEME and the position single-sidedly absorbs the flETH. This is what makes a Flaunch launch possible without the creator having to seed any capital.

## 2. The math

There are three formulas worth knowing.

### (a) Initial market cap and starting price

The creator passes a USDC market cap into `initialPriceParams` (minimum 1,000 USDC, typical values 1,000 – 75,000). With a fixed total supply of 100,000,000,000 tokens (1e29 wei at 18 decimals):

$$P_0 = \frac{\text{MarketCap}_{USDC}}{100{,}000{,}000{,}000}$$

So a $10,000 market cap launch starts at $0.0000001 per token (0.0000001 USDC). That price is converted to ETH at the current ETH/USDC oracle rate, then to a `sqrtPriceX96` that initializes the v4 pool. The flaunching fee is **0.1% of that market cap, paid in ETH** — i.e. `flaunchingFee = MarketCap × 0.001 / ETH_price`.

### (b) The fair-launch fixed price

During the first 30 minutes (configurable up to 30 days for scheduled launches), every buyer receives tokens at $P_0$ regardless of how many have been bought before them. The hook's `beforeSwap` callback intercepts the trade, computes the output as `amountIn / P_0`, and bypasses the AMM curve entirely. Three constraints apply during this window:

- Per-wallet cap: 0.25% of total supply by default
- Tokens bought during fair launch are non-transferable until the window closes
- Refund mechanic: holders can sell back at exactly $P_0$ (minus the AMM fee), making the period effectively risk-free

Once the window closes, the remaining unsold fair-launch supply is dumped into the AMM at $P_0$ as a normal v4 position, and price discovery begins.

### (c) The Progressive Bid Wall — the actual interesting math

This is the part most explainers get hand-wavy about. Here is what is really happening.

Every swap pays a fee to the hook (currently ~1% via the static fee calculator). That fee is split per the creator's `creatorFeeAllocation` (a number from 0 to 100_00, where 100_00 = 100%). Define:

- $\alpha$ = creator share (e.g. 0.20 for 20%)
- $f_i$ = fee from swap $i$, denominated in flETH
- Community share per swap: $(1 - \alpha) \cdot f_i$

The hook accumulates the community share in an internal balance for each pool. **As soon as the running total crosses 0.1 ETH, the hook deploys that 0.1 ETH as a single-sided concentrated liquidity position** placed one tick spacing below the current spot price. In v4 tick math, a price moves by a factor of $1.0001^{\Delta\text{tick}}$ per tick, and Flaunch uses tickSpacing = 60, so the bid wall sits at:

$$P_{bid} = P_{spot} \cdot 1.0001^{-60} \approx P_{spot} \cdot 0.99402$$

That's roughly 0.6% below spot — close enough to absorb sells, far enough that ordinary noise doesn't trigger it.

The "progressive" part is what happens next. On each subsequent swap, the hook checks whether spot has moved up. If $P_{spot}$ has risen above the bid-wall position by more than one tick spacing, the hook **withdraws the existing position and re-deploys it one tick below the new spot**, plus any newly accumulated 0.1 ETH tranches. So the wall ratchets upward forever, but never down. Total ETH locked in the wall after $N$ tranches:

$$\text{WallSize} = 0.1 \text{ ETH} \cdot N$$

where $N = \lfloor \text{cumulative community fees} / 0.1 \rfloor$. For a coin doing $1M daily volume at a 1% fee with 80% to community, that's $1M × 0.01 × 0.8 = $8,000/day going into the wall — or about 80 new tranches per day at $100 ETH-equivalent each.

The mechanism is reflexive: rising price → bigger wall → harder to sell down → price holds → more volume → more fees → bigger wall. It's the same flywheel as an OlympusDAO PCV but driven by trading volume instead of bonds.

## 3. The Progressive Bid Wall, drawn
<img width="1440" height="974" alt="image" src="https://github.com/user-attachments/assets/3552122a-8c85-429f-b9ec-6047a1c540ed" />

## 4. The historical numbers

A few datapoints worth knowing, with the caveat that memecoin launchpad metrics move fast and you should treat any single snapshot as a checkpoint, not a steady state.

**The launch.** Flaunch went live on Base on **31 January 2025**, riding the broader Uniswap v4 mainnet launch the same week. Within roughly seven days of going live, Flaunch had launched 2,135 tokens, generated $75.6 million in v4 trading volumes, and paid back 51 ETH. Bankless reported $628,900+ in revenue had been returned to creators and communities just two days after launch.

**Hook dominance.** According to HookRank.io shortly after launch, 24 hooks had been deployed on v4 with the overwhelming majority of TVL ($2.3 million) sitting in Flaunch's hook contracts. Flaunch was the proof-of-concept that hooks weren't just an academic feature — they could anchor a billion-dollar product category.

**Phase 2 milestones.** Per Flaunch's own changelog in their docs, the protocol hit **$1M in cumulative creator revenue** by 23 August 2025, then launched **Groups** (a primitive that lets one anchor coin aggregate fee flows from multiple sub-coins) on 17 November 2025. Solana token bridging via Flaunch Wrap shipped 11 December 2025.

**Lineage.** Flaunch is built by Flayer Labs, which is itself a 2024 merger of **NFTX and FloorDAO**. That history matters because the FLAY token (the protocol governance token) inherited holders from both — the docs note 33.35% of FLAY supply was allocated to NFTX migrants and 16.65% to FloorDAO holders, with 20% controlled by the DAO. The treasury reportedly holds ~$25M including 5 CryptoPunks, 42 Lil Pudgys, an Autoglyph, and various NFTX vault positions.

**FLAY as a lever.** FLAY itself doesn't capture fees by default. There is a governance-controlled fee switch that, if enabled by FLAY voters, would route up to 10% of protocol trading fees to FLAY. As of the most recent search, that switch has not been flipped, and FLAY trades around $0.01–0.02 with a circulating market cap in the single-digit millions of dollars — a small fraction of the volume Flaunch processes. The bull case for FLAY is essentially "the fee switch eventually flips"; the bear case is "the protocol works fine without it ever flipping, so why would holders sacrifice volume for it."

## 5. Why this hook is genuinely novel

Three things Flaunch does that no pre-v4 launchpad could do at the AMM layer:

The first is **fee redirection at the pool level**. On v2/v3, swap fees go to LPs, period. The protocol cannot intercept them without putting a router/wrapper in front and convincing every aggregator to use it. With a v4 hook, the protocol *is* the LP rules — fees can be routed to a creator NFT, an Aave deposit, or a buyback queue before they ever accrue to a position. Flaunch uses this to pay creators in ETH instead of the memecoin, which is the single biggest UX win over pump.fun (where creators have to dump their token to realize value).

The second is **enforced launch mechanics**. The 30-minute fixed-price window, the no-sell rule, the per-wallet cap — these are AMM-level invariants that hold for every router, every aggregator, every contract that ever calls `swap()` on the pool. There is no "use the official frontend" disclaimer; the hook itself reverts trades that violate the rules.

The third is **automated single-sided LP management**. The Progressive Bid Wall is essentially a perpetual limit-order book maintained by the protocol, where the order size is funded by trading fees and the price level tracks spot. Building this without hooks would require an off-chain keeper running on a private RPC, paying for transactions, and being trusted not to MEV the wall. With hooks, the rebalance happens inside the same transaction as the swap that triggered it, atomically and trustlessly.

If you want to read the contracts directly, they're on GitHub at `flayerlabs/flaunchgg-contracts` (the `BidWall` and `FairLaunch` contracts in `src/contracts/` are the most interesting), and the canonical mainnet addresses are on Base — `PositionManager` at `0x51Bba15255406Cfe7099a42183302640ba7dAFDC` and `BidWall` at `0x66681f10BA90496241A25e33380004f30Dfd8aa8`. The repo has been audited three times by Enigma Dark plus an initial audit by Omniscia.
