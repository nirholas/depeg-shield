# DepegShield

**Makes leaving a peg expensive and returning to it cheap, in proportion to how far the pool has already strayed.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://depeg-shield.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/DepegShieldHook.sol`](src/hooks/DepegShieldHook.sol)
- **Licence:** Apache-2.0

## How it works

A pegged pool fails in a specific way. Something spooks the market, the first sellers cross, the price slips, and the slip is itself the signal that brings the next sellers. Liquidity providers are filled the whole way down at a fee that was set for a pool sitting at par.

By the time anyone reacts, the pool is one-sided and the providers own the asset that broke. This hook makes the fee a function of two things: how far the pool is from the peg, and which way the swap pushes it. A swap that widens the gap pays `baseFee` plus a surcharge that grows with the existing deviation.

A swap that closes the gap pays less than `baseFee`, down to a floor, with the discount growing the same way. The result is a spread that opens as the pool strays and pays anyone willing to push it back. widening: fee = baseFee + maxSurcharge * deviation / (deviation + halfDeviation) restoring: fee = baseFee - (baseFee - minFee) * deviation / (deviation + halfDeviation) Both are LP fees, so the surcharge is paid to liquidity and the discount is given up by liquidity.

That is the right trade for a provider in a pegged pool: paying for the flow that repairs the pool is cheaper than being filled on the way out. The peg is a tick, not an oracle. `pegTick = 0` is a one-to-one pool; a pair whose par is not one-to-one sets the tick that corresponds to par.

Since it is fixed at initialization, there is nothing to manipulate and no feed to go stale, and a pool whose peg genuinely re-bases has to be re-created, which for a pegged pair is the honest outcome. Deviation is measured in ticks. One tick is one basis point to within rounding, so `halfDeviationTicks = 50` means half the surcharge applies once the pool is fifty basis points off par.

Prior art: stable-swap curves flatten the price impact near par, and dynamic-fee hooks keyed on volatility exist. Neither is directional. A curve treats a swap toward the peg and a swap away from it identically, and a volatility fee charges the repairing flow exactly as much as the flow that broke the pool.

Charging asymmetrically by direction of travel is what is new here.

## Prior art

Stable-swap curves flatten price impact near par, and dynamic-fee hooks keyed on volatility exist. Neither is directional: a curve prices a swap toward the peg and one away from it identically, and a volatility fee charges the repairing flow exactly as much as the flow that broke the pool. Charging asymmetrically by direction of travel is what is new.

## Where it does not help

The peg is fixed at initialization, so a pair whose par genuinely re-bases has to be re-created. For a pegged pair that is the honest outcome, but it does mean this is the wrong hook for a drifting reference such as a yield-bearing wrapper.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    DepegShieldHook.Config({
        pegTick: /* int24 */ 0,
        baseFee: /* uint24 */ 0,
        minFee: /* uint24 */ 0,
        maxSurcharge: /* uint24 */ 0,
        halfDeviationTicks: /* uint24 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `pegTick` | `int24` | ticks |
| `baseFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `minFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `maxSurcharge` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `halfDeviationTicks` | `uint24` | ticks |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidConfig()` | `halfDeviationTicks` was zero, or `minFee` exceeded `baseFee`. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SurchargeTooLarge()` | `baseFee + maxSurcharge` must leave room under the 100% protocol maximum. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 2 of the fourteen:

- `afterInitialize`
- `beforeSwap`

Mask: `0x1080`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # DepegShield
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # risk, stablecoin, dynamic-fee, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/depeg-shield
cd depeg-shield
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
