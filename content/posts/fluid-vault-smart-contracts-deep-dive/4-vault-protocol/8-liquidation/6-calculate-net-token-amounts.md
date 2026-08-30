---
title: "Liquidate: Calculate net token amounts"
weight: 41
layout: book
hiddenInHomeList: true
---

We are mostly done with this function. We finished the liquidation loop, and now we need to calculate the exact amounts that need to be transferred to the liquidator.

{{< figure src="/fluid-vault/vault/calc-exact-amount.png" alt="Vault protocol" width="80%" >}}

First, we convert amounts to net amounts by multiplying with exchange prices:

```solidity
actualDebtAmt_ = (currentData_.totalDebtLiq * memoryVars_.borrowExPrice) / EXCHANGE_PRICES_PRECISION;
actualColAmt_ = (currentData_.totalColLiq * memoryVars_.supplyExPrice) / EXCHANGE_PRICES_PRECISION;
```

The reason for doing this is that we previously converted from net to raw before we started the liquidation loop, as ratio calculation works with raw amounts:

```solidity
// debtAmt_ should be less than 2**128 & EXCHANGE_PRICES_PRECISION is 1e12
unchecked {
    currentData_.debtRemaining = (debtAmt_ * EXCHANGE_PRICES_PRECISION) / memoryVars_.borrowExPrice;
}
```

Then we perform a few sanity checks, with this one being the most important:

```solidity
if (((actualColAmt_ * 1e18) / actualDebtAmt_) < colPerUnitDebt_) {
    revert FluidVaultError(ErrorTypes.Vault__ExcessSlippageLiquidation);
}
```

We want to ensure that the liquidator receives enough collateral for liquidated debt by using his passed `colPerUnitDebt_` value.

After this, we use the [dead-address simulation trick]({{< ref "./1-sanity-checks-and-setup.md" >}}#dead-address-simulation) mentioned earlier to revert with the calculated liquidation amounts:

```solidity
if (to_ == 0x000000000000000000000000000000000000dEaD) {
    // revert with liquidated amounts if to_ address is the dead address.
    // this can be used in a resolver to find the max liquidatable amounts.
    revert FluidLiquidateResult(actualColAmt_, actualDebtAmt_);
}
```