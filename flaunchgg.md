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

## 5. How the v4 hook callbacks actually fire

Uniswap v4 lets a hook contract subscribe to up to 14 callback flags fired by the `PoolManager` at specific moments in the lifecycle of a pool. Most hooks subscribe to one or two of these. Flaunch's `PositionManager` subscribes to **nine**, which is unusually aggressive — it is one of the most callback-dense hooks in production. Here is the exact `getHookPermissions()` from the deployed source:

```solidity
return Hooks.Permissions({
    beforeInitialize:                true,   // prevents external pool init
    afterInitialize:                 false,
    beforeAddLiquidity:              true,   // FairLaunch + InternalSwapPool gate
    afterAddLiquidity:               true,   // event tracking
    beforeRemoveLiquidity:           true,   // FairLaunch + ISP gate
    afterRemoveLiquidity:            true,   // event tracking
    beforeSwap:                      true,   // FairLaunch + ISP fill + premine
    afterSwap:                       true,   // FeeDistributor + BidWall deposit
    beforeDonate:                    false,
    afterDonate:                     true,   // event tracking
    beforeSwapReturnDelta:           true,   // hook can return a custom delta (ISP)
    afterSwapReturnDelta:            true,   // hook can take swap fees
    afterAddLiquidityReturnDelta:    false,
    afterRemoveLiquidityReturnDelta: false
});
```

A subtle but important v4 detail: **the hook contract's address must encode this exact permission set in its low bits**. The dev comment in the source notes the bitmap as `1011 1111 0111 00` (the trailing `00` reflects the two unused flags). Hook deployers use CREATE2 with a mined salt to find an address whose last 14 bits match the desired flags — if they don't, the v4 `PoolManager` rejects the contract at pool initialization. This is why all Flaunch pools share one `PositionManager` address: re-mining a CREATE2 salt for every new memecoin would be prohibitively expensive, and a single hook can serve infinite pools.

The diagram below shows the order callbacks fire during a single buy.A few callback details that aren't obvious from the diagram alone:
<img width="1440" height="1864" alt="image" src="https://github.com/user-attachments/assets/2c6198cb-654c-4f3f-9d3d-aafea6091153" />


**`beforeInitialize` is a deny-all guard.** The hook permanently reverts with `CannotBeInitializedDirectly()`. The only way to create a pool with this hook is to call `flaunch()` on the `PositionManager` itself — which then internally calls `poolManager.initialize()`, bypassing its own guard via the implicit trust path. This stops anyone from creating a "fake" Flaunch pool with arbitrary token pairs.

**`beforeSwap` returns a `BeforeSwapDelta`** — a packed struct of two int128s representing how much of each currency the hook is taking out of (or putting into) the swap before the AMM math runs. Flaunch uses this to prepend fills from the `FairLaunch` position and the `InternalSwapPool` (ISP) before the user's order ever touches the v4 curve. The `beforeSwapReturnDelta` permission flag is what allows this — without it, v4 ignores any returned delta. The ISP is where memecoins paid as fees get sold back into ETH-equivalent inside the same transaction, so they never become sell pressure on the public pool.

**`beforeAddLiquidity` and `beforeRemoveLiquidity` are gates, not extensions.** During the fair-launch window they revert. After the window closes, they let normal v4 LPs add full-range or concentrated positions, but the hook itself remains the privileged actor that can manage the BidWall and the FairLaunch single-sided positions.

**`afterSwap` is where the value-routing happens.** It captures the swap fee in whatever currency was "unspecified" by the trade direction, runs `_distributeFees`, splits the haul into four buckets per pool — creator, BidWall, memecoin treasury, protocol — and conditionally pushes the BidWall slice into a new tick position if the running balance crossed the 0.1 ETH threshold. The `afterSwapReturnDelta` flag is what lets the hook actually take currency from the swap rather than merely observe it.

**Transient storage is used for premine.** When the creator calls `flaunch()` with a non-zero `premineAmount`, the contract writes to EIP-1153 transient storage keyed by `poolId`. The very next `beforeSwap` call in the same transaction reads that transient slot and recognizes the swap as the legitimate creator premine; otherwise the swap reverts because the pool is still scheduled. This is how premining is atomic and uncensorable — there is no separate "premine queue" contract; the data lives only for the duration of the `flaunch()` call.

## 6. The smart-contract architecture

Flaunch is built as a layered set of contracts where the `PositionManager` inherits from several specialized base contracts and orchestrates several more by composition. The key inheritance chain in the source:

```
PositionManager is BaseHook, FeeDistributor, InternalSwapPool, StoreKeys
```

Each parent owns a coherent piece of behavior, and the `PositionManager` is mostly a thin glue layer that wires them into the v4 lifecycle. Here is what each piece does and how the satellite contracts plug in.
<img width="1440" height="1228" alt="image" src="https://github.com/user-attachments/assets/bf9ccf8e-93b5-4bf9-be3d-e358f0d25957" />

Walking through the contracts that matter, in roughly the order they get touched in a launch:

**`Flaunch`** is the ERC-721 collection that mints the **Memestream NFT** for every new memecoin. Each tokenId in this contract represents the right to claim creator revenue from one specific pool. There is a 1:1 mapping between tokens minted here and pools created in v4. Transferring the NFT transfers the income stream — there is no separate "set new beneficiary" call.

**`Memecoin`** is the ERC-20 implementation. Every memecoin deployed via Flaunch is a clone of this template (or, post-v1.1, a direct deployment) with a fixed supply of 100,000,000,000 (1e29 with 18 decimals). The supply is non-inflatable, no mint function is exposed, and the contract holds a reference to its `MemecoinTreasury` and creator NFT.

**`MemecoinTreasury`** is a per-coin vault that holds ETH on behalf of the coin's community. When the BidWall is disabled (a creator choice), the community share of fees lands here instead of being deployed as buyback liquidity. It can only be drained via whitelisted actions in `TreasuryActionManager` — initially just `BuyBackAction` (market buy the coin) and `BurnTokensAction` (burn what's been bought). New action types require governance.

**`FairLaunch`** owns the single-sided concentrated liquidity position that defines the fixed-price window. It exposes `createPosition`, `fillFromPosition`, and `closePosition`. The `fillFromPosition` call is invoked from inside the hook's `beforeSwap` and returns a `BalanceDelta` indicating how much of the user's swap was satisfied at the fixed price; whatever remains spills over into normal AMM trading. When the window closes, unsold supply is sent to the dEaD address — this is the only path through which Flaunch supply ever decreases.

**`BidWall`** owns the buyback positions. Its key entry point is `deposit(poolKey, ethAmount, currentTick, nativeIsZero)`, called from `_distributeFees`. Internally it accumulates ETH per pool until a configurable threshold (default 0.1 ETH), then unlocks the `PoolManager`, removes its previous position, and mints a fresh single-sided ETH position one tick spacing below the current tick. The `checkStalePosition` call on every `beforeSwap` is the ratchet — if spot has moved up far enough that the old wall is stranded too far below, the wall is rebuilt at the new level. The contract also exposes `closeBidWall`, callable only by the Memestream NFT owner via the `PositionManager`, which liquidates the wall back to ETH and routes it to the treasury.

**`InternalSwapPool`** is a base contract that gives the `PositionManager` an internal "shadow order book." Whenever someone sells a memecoin and pays fees in the memecoin, those fee tokens are stored in `_poolFees[poolId].amount1`. The next buyer's swap is partially filled from this stash *before* the AMM is touched. Net effect: token-denominated fees are converted into ETH-equivalent on the way past the next opposing swap, which is why the protocol can promise creators "fees in ETH, never in your token."

**`FeeDistributor`** is the accounting layer. Its `feeSplit(poolId, amount)` returns `(bidWallFee, creatorFee, protocolFee)` based on the pool's `creatorFee[poolId]` value (set at launch from `creatorFeeAllocation`) and the global protocol fee parameters. The `_allocateFees` calls credit each recipient's claimable balance, and recipients pull via `claim()` calls.

**`StaticFeeCalculator`** is currently the only deployed fee calculator and just returns a constant 1% fee. The interface (`IFeeCalculator`) was deliberately left abstract — Flaunch has built in support for *dynamic* fee calculators (volatility-scaled fees, time-decaying fees, etc.) but ships only the static one for now. Each pool can be assigned a different calculator at flaunch time via the `feeCalculatorParams` blob.

**`Notifier`** is a thin event-fan-out contract. The hook emits `PoolStateUpdated`, `PoolSwap`, and other events through Notifier so that subgraph indexers and the Flaunch frontend can subscribe to a single source rather than every pool.

**`FlaunchPremineZap`** is the recommended entry point for creators. Calling `PositionManager.flaunch()` directly works but doesn't bundle the premine purchase into the same transaction. The zap wraps the call, computes the premine cost as `marketCap × premineAmount / 100B`, sends the right ETH, and refunds excess. The crucial property: because the premine happens inside the same transaction as `flaunch()`, no MEV bot can frontrun it — there is literally no block in which the pool exists and the premine has not yet been executed.

**`flETH`, `flETHHooks`, `AaveV3Strategy`** are the second-pool half of the system. flETH is a 1:1 wrapped ETH primitive (`0x000000000d564d5be76f7f0d28fe52605afc7cf8`). Whenever ETH enters via the `ETH/flETH` pool, the `flETHHooks` contract intercepts the deposit and routes the ETH through `FlAaveV3WethGateway` into Aave v3 on Base via `AaveV3Strategy`. The aETH yield (~2%) accrues to the Flayer Foundation and funds protocol operations. This is a critical architectural decision: by yielding on idle ETH, Flaunch can avoid taking any per-swap cut while still funding development. The fee switch — currently off — would let FLAY governance redirect up to 10% of swap fees on top of this.

**Treasury Managers** sit one level above the NFT. By default the Memestream NFT's `ownerOf` is the address that receives creator fees. But if the NFT is deposited into a `RevenueManager`, `AddressFeeSplitManager`, `StakingManager`, or a custom contract, the Manager becomes the recipient and applies its own routing logic — protocol fees, multi-recipient splits, stake-weighted distributions, time-locked vesting, etc. Custom managers are how third-party launchpads (Flaunch's API customers) take their own cut of fees while still using Flaunch's hook for the actual AMM mechanics.

If you want to read the most consequential file, it's the 1,187-line `src/contracts/PositionManager.sol` — the `flaunch()` function (creating a coin) and the `beforeSwap`/`afterSwap` pair (every trade) together account for roughly 60% of the protocol's behavior.

## 7. Why this hook is genuinely novel

Three things Flaunch does that no pre-v4 launchpad could do at the AMM layer:

The first is **fee redirection at the pool level**. On v2/v3, swap fees go to LPs, period. The protocol cannot intercept them without putting a router/wrapper in front and convincing every aggregator to use it. With a v4 hook, the protocol *is* the LP rules — fees can be routed to a creator NFT, an Aave deposit, or a buyback queue before they ever accrue to a position. Flaunch uses this to pay creators in ETH instead of the memecoin, which is the single biggest UX win over pump.fun (where creators have to dump their token to realize value).

The second is **enforced launch mechanics**. The 30-minute fixed-price window, the no-sell rule, the per-wallet cap — these are AMM-level invariants that hold for every router, every aggregator, every contract that ever calls `swap()` on the pool. There is no "use the official frontend" disclaimer; the hook itself reverts trades that violate the rules.

The third is **automated single-sided LP management**. The Progressive Bid Wall is essentially a perpetual limit-order book maintained by the protocol, where the order size is funded by trading fees and the price level tracks spot. Building this without hooks would require an off-chain keeper running on a private RPC, paying for transactions, and being trusted not to MEV the wall. With hooks, the rebalance happens inside the same transaction as the swap that triggered it, atomically and trustlessly.

If you want to read the contracts directly, they're on GitHub at `flayerlabs/flaunchgg-contracts` (the `BidWall` and `FairLaunch` contracts in `src/contracts/` are the most interesting), and the canonical mainnet addresses are on Base — `PositionManager` at `0x51Bba15255406Cfe7099a42183302640ba7dAFDC` and `BidWall` at `0x66681f10BA90496241A25e33380004f30Dfd8aa8`. The repo has been audited three times by Enigma Dark plus an initial audit by Omniscia.


