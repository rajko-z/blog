---
title: "Operate: Setup memory variables"
weight: 5
layout: book
hiddenInHomeList: true
---

After we performed sanity checks, we want to set up memory variables which will help us later to perform various calculations during the function.

We are looking at this piece of code:

{{< figure src="/fluid-vault/liquidity-layer/2-setup-memory-vars.png" alt="Liquidity layer" width="100%" >}}

First, we initialize `OperateMemoryVars`. This is just a common pattern to group local variables inside one struct to prevent stack too deep errors.

Then we are introduced to variables `memVar_` and `memVar2_`. This is a pattern heavily used across the Fluid codebase, where `memVar_`, `memVar1_` ... `memVarN_` are temporary variables that are reused across calculations and assignments to avoid creating new ones, with the goal of reducing gas cost.

It does sacrifice readability to some degree, but we will have to get used to it.

`memVar2_` calculates `operateAmountIn` of total assets passed to this function as `deposit + payback` amount, which will be used later. This is the reason why we capped amounts to 128 bits when we performed sanity checks on the previous page.

`memVar3_` is just temporarily assigned to the percentage of excess value which will be used later.

```solidity
/// @dev limit for triggering a revert if sent along excess input amount diff is bigger than this percentage (in 1e2)
uint256 internal constant MAX_INPUT_AMOUNT_EXCESS = 100; // 1%
```

Note that `memVar3_` is already assigned previously, as it represents one of two return values from the operate function. Of course, this value will later get its real semantic meaning; right now it is only used for temp storage.

```solidity
    function operate(
       ...
    ) external payable reentrancy returns (uint256 memVar3_, uint256 memVar4_)
//                                                     ^
//                                                     |
//                                                     |
```