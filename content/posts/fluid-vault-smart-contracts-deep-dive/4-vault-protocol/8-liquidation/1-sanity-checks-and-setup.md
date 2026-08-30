---
title: "Liquidate: Sanity check and setup"
weight: 35
layout: book
hiddenInHomeList: true
---

We are entering the `liquidate` function with the code below:

{{< figure src="/fluid-vault/vault/liquidate-sanity-checks.png" alt="Vault protocol" width="100%" >}}

Liquidator is sending:

- `debtAmt_` -> amount of debt to liquidate
- `colPerUnitDebt_` -> min amount of collateral per one unit of debt that liquidator is expecting to get in return
- `to_` -> where to send collateral tokens
- `absorb_` -> whether liquidation should first consume absorbed debt (or better known as bad debt)

Upon entering the function, we first perform the standard reentrancy check:

```solidity
// ############# turning re-entrancy bit on #############
if (vaultVariables_ & 1 == 0) {
    // Updating on storage
    vaultVariables = vaultVariables_ | 1;
} else {
    revert FluidVaultError(ErrorTypes.Vault__AlreadyEntered);
}
```

Then we check for `msg.value`. If the borrow token is native token (e.g. ETH for Ethereum network and L2s), then we ensure that value sent must match the `debtAmount` param. If not, then no `msg.value` should be present:

```solidity
if (BORROW_TOKEN == NATIVE_TOKEN) {
    if ((msg.value != debtAmt_) && (to_ != 0x000000000000000000000000000000000000dEaD)) {
        revert FluidVaultError(ErrorTypes.Vault__InvalidMsgValueLiquidate);
    }
} else if (msg.value > 0) {
    revert FluidVaultError(ErrorTypes.Vault__InvalidMsgValueLiquidate);
}
```

## Dead-address simulation

We are also seeing this hardcoded constant of `0x00..dEaD` address, which is a common pattern to deliberately use the main logic function to return internal calculations instead of writing a new separate getter function. In this case, it will later return calculated liquidation amounts:

```solidity
if (to_ == 0x000000000000000000000000000000000000dEaD) {
    // revert with liquidated amounts if to_ address is the dead address.
    // this can be used in a resolver to find the max liquidatable amounts.
    revert FluidLiquidateResult(actualColAmt_, actualDebtAmt_);
}
```

Once we check for `msg.value`, we are checking for the existence of `topTick`. Without `topTick`, we can't run liquidation as we don't know what the starting point is. This case will rarely ever be triggered, because if there are some users, there will be a top tick:

```solidity
memoryVars_.vaultVariables2 = vaultVariables2;

if (((vaultVariables_ >> 2) & X20) == 0) {
    revert FluidVaultError(ErrorTypes.Vault__TopTickDoesNotExist);
}
```

Then we update exchange prices that were already covered [here]({{< ref "../2-first-supply-withdraw-no-debt.md#4-update-of-exchange-prices" >}}):

```solidity
// Below are exchange prices of vaults
(, , memoryVars_.supplyExPrice, memoryVars_.borrowExPrice) = updateExchangePrices(memoryVars_.vaultVariables2);
```

After this, we initialize a bunch of temp memory structs that will be used during liquidation:

```solidity
struct LiquidateMemoryVars {
    uint256 vaultVariables2;
    int liquidationTick;
    int maxTick;
    uint256 supplyExPrice;
    uint256 borrowExPrice;
}

// note: All the below token amounts are in raw form.
struct CurrentLiquidity {
    uint256 debtRemaining; // Debt remaining to liquidate
    uint256 debt; // Current liquidatable debt before reaching next check point
    uint256 col; // Calculate using debt & ratioCurrent
    uint256 colPerDebt; // How much collateral to liquidate per unit of Debt
    uint256 totalDebtLiq; // Total debt liquidated till now
    uint256 totalColLiq; // Total collateral liquidated till now
    int tick; // Current tick to liquidate
    uint ratio; // Current ratio to liquidate
    uint tickStatus; // if 1 then it's a perfect tick, if 2 that means it's a liquidated tick
    int refTick; // ref tick to liquidate
    uint refRatio; // ratio at ref tick
    uint refTickStatus; // if 1 then it's a perfect tick, if 2 that means it's a liquidated tick, if 3 that means it's a liquidation threshold
}

struct BranchData {
    uint id;
    uint data;
    uint ratio;
    uint debtFactor;
    int minimaTick;
    uint baseBranchData;
}
```