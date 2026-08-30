---
title: "Liquidate: Setup for liquidation loop"
weight: 39
layout: book
hiddenInHomeList: true
---

We are looking at this code:

{{< figure src="/fluid-vault/vault/liq-setup.png" alt="Vault protocol" width="100%" >}}

This code is just setup code with no new logic, so we will quickly run over it.

We first validate that the input amount for debt to liquidate is not too low or too high:

```solidity
if (debtAmt_ < 10000 || debtAmt_ > X128) {
    revert FluidVaultError(ErrorTypes.Vault__InvalidLiquidationAmt);
}
```

Then we set up memory variables that will be used later:

```solidity
// setting up status if top tick is liquidated or not
currentData_.tickStatus = vaultVariables_ & 2 == 0 ? 1 : 2;
// Tick info is mainly used as a place holder to store temporary tick related data
// (it can be current or ref using same memory variable)
TickData memory tickInfo_;
tickInfo_.tick = currentData_.tick;
```

After this, we load the current branch data to memory:

```solidity
 branch_.id = (vaultVariables_ >> 22) & X30;
branch_.data = branchData[branch_.id];
branch_.debtFactor = (branch_.data >> 116) & X50;
```

Then, we initialize debt factor to 1, if it is not initialized. If you are unfamiliar with debt factors, recall it from [here]({{< ref "../7-borrow-second-time/2-fetch-liquidated-position/1-liquidations-and-factors.md" >}}):

```solidity
if (branch_.debtFactor == 0) {
    // Initializing branch debt factor. 35 | 15 bit number. Where full 35 bits and 15th bit is occupied.
    // Making the total number as (2**35 - 1) << 2**14.
    // note: initial debt factor can be any number.
    branch_.debtFactor = ((X35 << 15) | (1 << 14));
}
```

Then we read base branch minima if it exists, or initialize it to `type(int).min` if not present:

```solidity
// fetching base branch's minima tick. if 0 that means it's a master branch
temp_ = (branch_.data >> 196) & X20;
if (temp_ > 0) {
    unchecked {
        branch_.minimaTick = (temp_ & 1) == 1 ? int256((temp_ >> 1) & X19) : -int256((temp_ >> 1) & X19);
    }
} else {
    branch_.minimaTick = type(int).min;
}
```

The final part is performing a sanity check that debt to liquidate is not less than 1e9 of total debt. As stated in the comments, and as we saw before, instead of thinking of all edge cases when working with low amount values, we just eliminate that case internally:

```solidity
// debtAmt_ should be less than 2**128 & EXCHANGE_PRICES_PRECISION is 1e12
unchecked {
    currentData_.debtRemaining = (debtAmt_ * EXCHANGE_PRICES_PRECISION) / memoryVars_.borrowExPrice;
}

// extracting total debt
temp2_ = (vaultVariables_ >> 146) & X64;
temp2_ = ((temp2_ >> 8) << (temp2_ & X8));

if ((temp2_ / 1e9) > currentData_.debtRemaining) {
    // if liquidation amount is less than 1e9 of total debt then revert
    // so if total debt is $1B then minimum liquidation limit = $1
    // so if total debt is $1T then minimum liquidation limit = $1000
    // partials precision is slightlty above 1e9 so this will make sure that on every liquidation atleast 1 partial gets liquidated
    // not sure if it can result in any issue but restricting amount further more to remove very low amount scenarios totally
    revert FluidVaultError(ErrorTypes.Vault__InvalidLiquidationAmt);
}
```

## Absorb setup

Before running into the main liquidation loop, we also have this part left from absorb flow that we trigger now.

{{< figure src="/fluid-vault/vault/absorb-setup.png" alt="Vault protocol" width="80%" >}}

First, we load `absorbedLiquidity` from storage, which was previously updated during absorb flow.

```solidity
temp_ = absorbedLiquidity;
// temp2_ -> absorbed col
temp2_ = (temp_ >> 128) & X128;
// temp_ -> absorbed debt
temp_ = temp_ & X128;
```

This variable will contain packed debt and collateral amounts:

```solidity
/// note: stores absorbed liquidity
/// First 128 bits raw debt amount
/// last 128 bits raw col amount
uint256 internal absorbedLiquidity;
```

Then we ask the question: `If we want to clear this absorbed liquidity from liquidator debt, will we clear all of it or will something be left?`

**If we clear all:**

- We reset absorbed liquidity to 0
- Remove absorbed debt from liquidator amount
- Increase collateral and debt liquidated so we can later transfer amount to liquidator

```solidity
// updating on storage
absorbedLiquidity = 0;
unchecked {
    currentData_.debtRemaining -= temp_;
}
currentData_.totalDebtLiq = temp_;
currentData_.totalColLiq = temp2_;
```

> [!NOTE]
> For absorbed debt we account absorbed collateral without any incentives.
>
> But here is one important observation to make. This absorb flow is optional, and yes, there is no liquidation incentive, however liquidator can still make profit from absorb flow!
>
> If we think about it, absorb will run on high LTV value, but there will still be buffer from max limit to 100%, so overcollateralized collateral becomes incentive itself.
>
> So depending on the aggregate risk of all absorbed positions, absorbing debt can still be incentivized to call. Of course, if the limit comes close to 100% or even above, in that case we have bad debt in the system. The only way to clear that bad debt is to realize loss, because collateral absorbed will be worth less than debt repaid.

**If we don't clear all:**

- Clear debt and collateral proportionally
- Make liquidator amount 0, as we used it fully
- Increase collateral and debt liquidated so we can later transfer amount to liquidator
- Update absorbed liquidity

```solidity
// Removing collateral in equal proportion as debt
currentData_.totalColLiq = ((temp2_ * currentData_.debtRemaining) / temp_);
temp2_ -= currentData_.totalColLiq;
// Removing debt
currentData_.totalDebtLiq = currentData_.debtRemaining;
unchecked {
    temp_ -= currentData_.debtRemaining;
}
currentData_.debtRemaining = 0;

// updating on storage
absorbedLiquidity = temp_ | (temp2_ << 128);
```