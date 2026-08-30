---
title: "Liquidate: Calculate liquidation and max ratio"
weight: 36
layout: book
hiddenInHomeList: true
---

We are looking at this piece of code:

{{< figure src="/fluid-vault/vault/liquidate-calc-ratios.png" alt="Vault protocol" width="100%" >}}

This part of code calculates 2 things:

- liquidation tick
- max tick

The part for liquidation tick calculation is something we already covered when we were checking user health position [here]({{< ref "../6-borrow-first-time.md#check-collateral-factor-health" >}}).

The only thing left to answer is what is max tick?

Max tick is the upper limit where positions above it get 100% liquidated immediately. This limit is primarily used as the last line of defense against bad debt and has a separate flow.

So Fluid has 2 limits: one for regular liquidations, and one for absorb flow, which we will cover in depth on the next page.

For now, this code just loads max liquidation factor from vault storage to calculate max ratio:

```solidity
/// ...
/// Next 160 bits => 96-255 => Oracle address
/// ...
uint256 internal vaultVariables2;

unchecked {
    temp2_ = (temp_ * ((memoryVars_.vaultVariables2 >> 52) & X10)) / 1000;
}
```

And then, from max ratio, it calculates max tick:

```solidity
(memoryVars_.maxTick, ) = TickMath.getTickAtRatio(temp2_);
```