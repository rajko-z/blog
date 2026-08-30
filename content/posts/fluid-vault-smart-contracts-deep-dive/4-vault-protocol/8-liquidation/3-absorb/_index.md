---
title: "Liquidate: Absorb"
weight: 37
layout: book
hiddenInHomeList: true
---

## General introduction

On the previous page, we mentioned the concept of `maxTick`, but let's give one example.

Suppose we have the following setup:

```text
Vault = ETH / USDC

CF                    = 80%
Liquidation Threshold = 85%
Liquidation Max Limit = 90%
Liquidation Penalty   = 5%

1 ETH = 2500 USDC
```

{{< figure src="/fluid-vault/vault/liquidation-factors.png" alt="Vault protocol" width="100%" >}}

- In this example, with regular actions on the protocol level, the user can borrow max 80% of value.
- If collateral price starts dropping down, there will be some buffer of 5%, so the user has time to adjust his position before he enters the liquidation zone.
- Liquidation zone will be run in range 85 - 90, where regular incentive with bonus is applied.
- If, however, the user gets above 90%, we are entering the absorb zone where regular liquidation incentive is not applied.

Suppose we have position:

```text
Before liquidation
-------------------
collateral value: 1020$
debt value: 900$
ltv = 88.23%
```

As we are in the liquidation zone with 5% bonus, for `100 $` USDC debt repaid, the liquidator will get `105 $` worth of ETH collateral.

```text
After liquidation
------------------
collateral value: 915$
debt value: 800$
ltv = 87.43%
```

Although still in the liquidation zone, the position is healthier than before (`87.43% < 88.23%`).

Now suppose that starting position is this:

```text
Before liquidation
-------------------
collateral value: 1000$
debt value: 960$
ltv = 96%
```

This position is part of the absorb zone, but let's naively try to apply the bonus factor:

```text
After liquidation
------------------
collateral value: 895$
debt value: 860$
ltv = 96.08%
```

**96.08% > 96%, we made the position even more risky!!!**

This is the exact reason why we can't use normal liquidation rules for highly risky positions, hence why Fluid introduces the absorb zone.

## Implementation introduction

Let's go back to the main liquidation logic. This is the snippet of code we are looking at:

{{< figure src="/fluid-vault/vault/absorb-snipet.png" alt="Vault protocol" width="100%" >}}

We are first loading current `topTick` from vault:

```solidity
// extracting top tick as top tick will be the current tick
unchecked {
    currentData_.tick = (vaultVariables_ & 4) == 4
        ? int256((vaultVariables_ >> 3) & X19)
        : -int256((vaultVariables_ >> 3) & X19);
}
```

Then we compare it against `maxTick` to check if we need to run absorb flow:

```solidity
if (currentData_.tick > memoryVars_.maxTick) ...
```

As we are analyzing absorb flow, we will assume that `topTick` is indeed in the absorb zone above `maxTick`. So upon entering this function, we see the following call:

```solidity
vaultVariables_ = (
    abi.decode(
        _spell(
            SECONDARY_IMPLEMENTATION,
            abi.encodeWithSignature("absorb(uint256,int256)", vaultVariables_, memoryVars_.maxTick)
        ),
        (uint256)
    )
);
```

We will go back to this soon, but what I want you to see here is that we are not passing standard liquidation function params like `debt` or `colPerUnitDebt`, meaning this `absorb` flow will be some generalized logic to clean up positions.

We also see that we expect this function to give us new `vaultVariables` with new `topTick`, which we can update in the current function:

```solidity
// updating current tick to new topTick after absorb
unchecked {
    currentData_.tick = (vaultVariables_ & 4) == 4
        ? int256((vaultVariables_ >> 3) & X19)
        : -int256((vaultVariables_ >> 3) & X19);
}
```

After this, we check if the liquidator passed `0` as `debtAmt_`, which will mean that he only wanted to run absorb flow without real liquidations later. In this case, we just update vault variables in storage and return:

```solidity
if (debtAmt_ == 0) {
    // updating vault variables on storage as the transaction was for only absorb
    vaultVariables = vaultVariables_;
    return (0, 0);
}
```

----

Now going back to the main `absorb` function.

We [already mentioned]({{< ref "../../1-vaults.md#vault-methods" >}}) at the start of the vault protocol section that there will be 2 main methods on vault that we already touched (`operate`, `liquidate`) and 1 other (`absorb`). This other method is part of secondary implementation, that's being delegatecalled to with the `spell` method:

```solidity
function _spell(address target_, bytes memory data_) private returns (bytes memory response_) {
    assembly {
        let succeeded := delegatecall(gas(), target_, add(data_, 0x20), mload(data_), 0, 0)
        let size := returndatasize()
        
        response_ := mload(0x40)
        mstore(0x40, add(response_, and(add(add(size, 0x20), 0x1f), not(0x1f))))
        mstore(response_, size)
        returndatacopy(add(response_, 0x20), 0, size)

        switch iszero(succeeded)
        case 1 {
            // throw if delegatecall failed
            returndatacopy(0x00, 0x00, size)
            revert(0x00, size)
        }
    }
}
```

This only exists to reduce bytecode size of the vault implementation, which is not an uncommon thing to do in DeFi protocols (Compound V3 does the same thing with CometExt: [https://github.com/compound-finance/comet/blob/main/contracts/CometExt.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometExt.sol)).

Both implementation contracts will follow the same storage layout (extending the `Variables` contract), so for us there is no difference in analysis, and we can jump straight to implementation.