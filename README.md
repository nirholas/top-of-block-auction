# TopOfBlockAuction

**Sells the right to trade first in a block, and gives the proceeds to the liquidity providers who are being traded against.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://top-of-block-auction.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/TopOfBlockAuctionHook.sol`](src/hooks/TopOfBlockAuctionHook.sol)
- **Licence:** Apache-2.0

## How it works

The first swap in a block against a pool whose price has moved overnight is worth money, and today that money goes to whoever wins the gas auction or pays the builder. The liquidity providers, whose stale quote is the entire source of the value, receive the ordinary swap fee and nothing else. This is the largest single transfer away from passive liquidity in the system, and it happens in a market the pool cannot see and does not participate in.

The idea of auctioning that right and paying the pool instead is not new: it is the am-AMM proposal, and MEV-Share and MEV-Boost redistribute adjacent value off-chain. What has been missing is a version the pool runs itself, per block, with no auctioneer, no off-chain infrastructure and nobody to trust with the proceeds. This is that.

A searcher bids native currency for a named future block. The highest bid at the time the block arrives wins the exclusive right to the first swap in it, and the bid is paid to the pool's liquidity providers. Everybody else can still trade in that block, just not first.

Once the winner has taken their slot, or if nobody bid, the pool is ordinary. Bidding on a future block rather than the current one is what makes this work without an auctioneer. A bid for the block being built cannot be evaluated inside that block without knowing bids that have not arrived, so the auction would have to be settled by somebody.

Bidding one block ahead means the winner is already known when the block opens, and the contract simply checks who it is. A losing bid is never taken. Bids are held and the loser withdraws in full, so bidding costs nothing but gas, and a searcher who is outbid is not out of pocket.

## Prior art

The mechanism is the am-AMM auction (Adams, Moallemi, Reynolds, Robinson), which auctions pool management rights over a longer horizon, and the broader family of order-flow auctions run off-chain by builders and relays. Running the auction per block, inside the pool, with the winner known before the block opens so no auctioneer is needed, is the contribution here.

## Where it does not help

The pool cannot see whether the winner actually used their slot well, only that they took it, so a searcher who wins and does nothing simply wastes their bid and blocks nobody. More importantly, a builder can still reorder the block: this sells the first slot in the pool's own accounting, not the first position in the block, so a searcher who controls ordering can place their own transaction before the winner's and the winner gets a slot that is no longer first. It raises the cost of that rather than preventing it.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    TopOfBlockAuctionHook.Config({
        leadBlocks: /* uint32 */ 0,
        minBid: /* uint128 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `leadBlocks` | `uint32` |  |
| `minBid` | `uint128` |  |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `BidTooLow(uint256)` | The bid does not beat the standing one. |
| `BlockTooSoon(uint256)` | The bid targets a block that is not far enough ahead, or has already passed. |
| `InvalidLead()` | `leadBlocks` of zero would ask the pool to settle an auction inside the block it is bidding on. |
| `NothingToWithdraw()` | There is nothing to withdraw. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SlotBelongsTo(address)` | Somebody else holds the first slot in this block. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # TopOfBlockAuction
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, auction, order-flow, lp-economics, no-admin
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/top-of-block-auction
cd top-of-block-auction
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.
