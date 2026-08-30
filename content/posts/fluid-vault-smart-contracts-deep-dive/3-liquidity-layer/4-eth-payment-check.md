---
title: "Operate: Eth payment check"
weight: 7
layout: book
hiddenInHomeList: true
---


After the skipped DEX callback, we are looking at this piece of code:

{{< figure src="/fluid-vault/liquidity-layer/4-eth-payment-check.png" alt="Liquidity layer" width="100%" >}}

If you remember, `memVar2_` was calculated as `deposit + payback` amount, so its purpose was for this check, where we want to make sure the caller has sent enough native token:

```solidity
memVar2_ > msg.value
```

But also not too much above the excess allowance limit:

```solidity
msg.value > (memVar2_ * (FOUR_DECIMALS + memVar3_)) / FOUR_DECIMALS
```
Where:

```solidity
memVar3_ = 100; // MAX_INPUT_AMOUNT_EXCESS = 1%
FOUR_DECIMALS = 1e4;

// 10100 / 10000 = 1.01

```
So this simply reverts if we sent 1% more native token than the combined deposit plus payback amount passed in the function call, to prevent the caller from accidentally passing more tokens than needed.

Also, one subtle detail needed for the next step: we are resetting the `memVar2_` value to zero.