---
title: "fn: fetchLatestPosition"
weight: 32
layout: book
hiddenInHomeList: true
---

We have background now, so let's revisit the main function for fetching user positions after liquidations:

{{< figure src="/fluid-vault/vault/fetch-latest-position-2.png" alt="Vault protocol" width="100%" >}}

## 1. Fetch tick generation data

We know that the user got liquidated, partially or fully. Liquidation happened on some branch, that might be, or might not be, merged into another branch (which can also be merged with its base, etc.).

We also know that the user is part of some epoch, which we call tick ID, which can be the latest current epoch living in the `tickData` mapping, or an older epoch that lives in history, in the `tickId` mapping.

So, let's check this in code.

We start with this block where we check if the user is part of the latest tick epoch by checking if total tick IDs match the user tick ID. This means no new epoch was generated for that tick and we can fetch liquidation info directly from `tickData`:

```solidity
// Checking if tick's total ID = user's tick ID
if (((tickData_ >> 1) & X24) == positionTickId_) {
    // fetching from tick data itself
    isFullyLiquidated_ = ((tickData_ >> 25) & 1) == 1;
    branchId_ = (tickData_ >> 26) & X30;
    connectionFactor_ = (tickData_ >> 56) & X50;
}
```

This was the case for the majority of ticks and user positions from the previous page example, but remember that we also had a case for tick `C` getting a new user on an already liquidated branch. In that case, we need to fetch liquidation info from history (`tickId` mapping):

```solidity
uint256 tickLiquidationData_;
unchecked {
    // Fetching tick's liquidation data. One variable contains data of 3 IDs. Tick Id mapping is starting from 1.
    tickLiquidationData_ =
        tickId[positionTick_][(positionTickId_ + 2) / 3] >>
        (((positionTickId_ + 2) % 3) * 85);
}

isFullyLiquidated_ = (tickLiquidationData_ & 1) == 1;
branchId_ = (tickLiquidationData_ >> 1) & X30;
connectionFactor_ = (tickLiquidationData_ >> 31) & X50;
```

Note that in both cases here, `connectionFactor_` is a bit misleading, as it actually represents the tick debt factor when the user was entering the liquidation loop for the first time. However, it's named like this because it will be reused later in the function to calculate the actual `connectionFactor` for merged branches.

## 2. User got fully liquidated

We didn't cover this case yet, but the user can get fully liquidated, which we will see later. If that's the case, we just return early with empty debt and default tick value:

```solidity
if (isFullyLiquidated_) {
    positionTick_ = type(int).min;
    positionRawDebt_ = 0;
}
```

## 3. Calculate connection factor

The following block is calculating the connection factor as long as we have some branch that's merged.

The snippet from branch data:

```solidity
/// First 2 bits => 0-1 => if 0 then not liquidated, if 1 then liquidated, if 2 then merged, if 3 then closed
```

In our example on the previous page, we only had one merge event, but we can also chain these branches as long as there exists a base branch from which the current branch was created.

```solidity
while ((branchData_ & 3) == 2) {
    // If true then the branch is merged

    // userTickDebtFactor * connectionDebtFactor *... connectionDebtFactor aka adjustmentDebtFactor
    connectionFactor_ = connectionFactor_.mulBigNumber(((branchData_ >> 116) & X50));
    if (connectionFactor_ == BigMathVault.MAX_MASK_DEBT_FACTOR) break; // user ~100% liquidated
    // Note we don't need updated branch data in case of 100% liquidated so saving gas for fetching it

    // Fetching new branch data
    branchId_ = (branchData_ >> 166) & X30; // Link to base branch of current branch
    branchData_ = branchData[branchId_];
}
```

This branch factor fetching will depend on branch state:

```solidity
(branchData_ >> 116) & X50
```

In this case, because the branch is `merged`, we know that in this place `connectionFactor` will be stored and not debt factor like on a regular unmerged branch:

```solidity
/// If not merged
/// Next 50 bits => 116-165 => Debt factor or of this branch. (35 bits coefficient | 15 bits expansion)
/// If merged
/// Next 50 bits => 116-165 => Connection/adjustment debt factor of this branch with the next branch.
/// If closed
/// Next 50 bits => 116-165 => Debt factor as 0. As all the user's positions are now fully gone
```

At the end of this while loop, `connectionFactor` will match the divider we saw in the previous page example:

{{< figure src="/fluid-vault/vault/connection-factor.png" alt="Vault protocol" width="60%" >}}

This while loop will only run for Felix and Grace, which can be seen with this `1.2` multiplier, and all other positions will have the default debt factor fetched from tick data in step 1, because those positions' branch `N` was never merged.

---

There is one more thing left in the code above. If you look carefully, there is one small implementation detail which we missed.

If the connection factor reaches a big enough sanity check value (having too many merge events for the youngest branch), we will consider the position fully liquidated.

```solidity
uint private constant COEFFICIENT_SIZE_DEBT_FACTOR = 35;
uint private constant EXPONENT_SIZE_DEBT_FACTOR = 15;
uint internal constant MAX_MASK_DEBT_FACTOR = (1 << (COEFFICIENT_SIZE_DEBT_FACTOR + EXPONENT_SIZE_DEBT_FACTOR)) - 1;
```

So this just ensures that the connection factor can be represented in 35-bit coefficient and 15-bit exponent format. If it can't, that means that user debt is effectively zero, so we view it as fully liquidated.

## 4. Branch is closed

We didn't talk about closed branches yet, but sometimes it is possible for a branch to be closed, or fully liquidated, in which case the user is left with no debt and we put his tick in the default uninitialized state of `type(int).min`, just like in step 2. We will go over this later, when we analyze the main liquidation function, but for now, we just return early from this function as the user is fully liquidated.

```solidity
if (((branchData_ & 3) == 3) || (connectionFactor_ == BigMathVault.MAX_MASK_DEBT_FACTOR)) {
    // Branch got closed (or user liquidated ~100%). Hence make the user's position 0
    // Rare cases to get into this situation
    // Branch can get close often but once closed it's tricky that some user might come iterating through there
    // If a user comes then that user will be very mini user like some cents probably
    positionTick_ = type(int).min;
    positionRawDebt_ = 0;
}
```

## 5. Calculate new debt and collateral

Once we have the connection factor, we want to calculate final debt and collateral.

### 5.1 Calculate debt

First, let's calculate debt:

```solidity
// position debt = debt * base branch minimaDebtFactor / connectionFactor
positionRawDebt_ = positionRawDebt_.mulDivNormal(
    (branchData_ >> 116) & X50, // minimaDebtFactor
    connectionFactor_
);
```

In our example on the previous page, the minima debt factor was `0.384` on the final non-merged branch.

### 5.2 Dust branch debt edge case

Then we come to this code block:

```solidity
// Reducing user's liquidity by 0.01% if user got liquidated.
// As this will make sure that the branch always have some debt even if all liquidated user left
// This saves a lot more logics & consideration on Operate function
// if we don't do this then we have to add logics related to closing the branch and factor connections accordingly.
if (positionRawDebt_ > (initialPositionRawDebt_ / 100)) {
    positionRawDebt_ = (positionRawDebt_ * 9999) / 10000;
} else {
    // if user debt reduced by more than 99% in liquidation then making user as fully liquidated
    positionRawDebt_ = 0;
}
```

This part is interesting because we are deliberately lowering the debt position by 0.01%.

Well first of all, we are not forgiving only 0.01% debt, because we are also calculating collateral from this debt later, meaning this shrinks the user position on both sides.

Fluid wants to differentiate between 2 cases:

1. During a valid liquidation loop, we closed the branch as fully liquidated.
2. Last user is leaving a liquidated branch, is branch debt now zero?

This code handles the second case.

Imagine that in our example on the previous page, only Alice's position is left in liquidated branch `N`, having total debt of `5.760k`.

Now, if Alice calls the `operate` function and gets assigned to a new tick that is active and part of a new branch on top of `N`, the debt of branch `N` would be:

```text
5.760k - 5.760k = 0
```

Now we have a state of a liquidated branch that is the base branch of branch `N+1`, but also has debt of `0`. This means that the debt factor of this branch `N` would be `0`. If the debt factor is zero, then the connection factor from branch `N+1` loses its purpose and it will also be zero:

```text
connectionFactor = baseBranchFactor / childBranchFactor
connectionFactor = 0 / childBranchFactor
connectionFactor = 0
```

So we lose a way to track positions, meaning this liquidated branch with zero debt should be an invalid state to be at.

There are 2 cases how this can be solved:

1. Mark the branch as closed and somehow recalculate connection factors so that retrieving positions from `N+1` gets possible
2. Simply leave some dust residual left, so that the last user leaving the branch can't leave the branch with zero debt, so connection factors are preserved

As stated in comments, Fluid chooses this second option as implementation is much simpler. We just leave 0.01% position left.

This also matches what we will see in future liquidation logic, where we explicitly require for the branch to have some dust debt left:

```solidity
if (currentData_.debt < 100) {
    // this can happen when someone tries to create a dust tick
    revert FluidVaultError(ErrorTypes.Vault__BranchDebtTooLow);
}
```

If we take `1 USDC` debt (6 token decimals), leaving `0.01%` of liquidity will produce:

```text
1000000 * 0.01% = 100
```

which will match exactly this dust debt left case for the branch. In practice, the branch will take every user leaving 0.01% position, so it will have more than `100` units of dust debt left.

### 5.3 Calculate collateral

Okay, we finished debt calculation, now we are left with the final part of calculating collateral:

```solidity
// positionTick_ -> read minima tick of branch
unchecked {
    positionTick_ = branchData_ & 4 == 4
        ? int((branchData_ >> 3) & X19)
        : -int((branchData_ >> 3) & X19);
}
// Calculating user's collateral
uint256 ratioAtTick_ = TickMath.getRatioAtTick(int24(positionTick_));
uint256 ratioOneLess_;
unchecked {
    ratioOneLess_ = (ratioAtTick_ * 10000) / 10015;
}
// formula below for better readability:
// length = ratioAtTick_ - ratioOneLess_
// ratio = ratioOneLess_ + (length * positionPartials_) / X30
// positionRawCol_ = (positionRawDebt_ * (1 << 96)) / ratio_
positionRawCol_ =
    (positionRawDebt_ * TickMath.ZERO_TICK_SCALED_RATIO) /
    (ratioOneLess_ + ((ratioAtTick_ - ratioOneLess_) * ((branchData_ >> 22) & X30)) / X30);
```

On the previous page, we explained that most of the time, we won't finish liquidation on a perfect rounded tick, but it will rather be somewhere between 2 ticks. In that case, minima will also have some partial.

{{< figure src="/fluid-vault/vault/partial.png" alt="Vault protocol" width="60%" >}}

By knowing upper tick (`positionTick`) and partial, we can locate the exact point of X.

In code, we first load position tick (on image point A):

```solidity
unchecked {
    positionTick_ = branchData_ & 4 == 4
        ? int((branchData_ >> 3) & X19)
        : -int((branchData_ >> 3) & X19);
}
```

Then, we calculate ratio at position tick (upper bound):

```solidity
uint256 ratioAtTick_ = TickMath.getRatioAtTick(int24(positionTick_));
```

After that, we calculate the lower bound (on our image position B), which would be the ratio at one perfect tick below:

```solidity
uint256 ratioOneLess_;
unchecked {
    ratioOneLess_ = (ratioAtTick_ * 10000) / 10015;
}
```

Finally, we just apply the formula and locate the exact point for minima tick by applying partial, which will be stored on the branch level:

```solidity
positionRawCol_ =
    (positionRawDebt_ * TickMath.ZERO_TICK_SCALED_RATIO) /
    (ratioOneLess_ + ((ratioAtTick_ - ratioOneLess_) * ((branchData_ >> 22) & X30)) / X30);
```

So partial will dictate how close we are to each side of perfect ticks.

----------------

This finishes this `fetchLatestPosition` function. At the end, we return data where we found liquidated position:

```solidity
return (positionTick_, positionRawDebt_, positionRawCol_, branchId_, branchData_);
```

Position tick will be either the minima tick of the final non-merged branch where the user landed, or it will be `type(int).min` if the user got fully liquidated.

Position raw debt and collateral will be recalculated amounts for the user position, and branch ID with `branchData` will be final branch info where the user landed.