---
title: "Liquidate: Liquidation loop"
weight: 40
layout: book
hiddenInHomeList: true
---

Finally, we are arriving at the core liquidation loop that will be the heart of this vault `liquidate` function.

{{< figure src="/fluid-vault/vault/main-liquidation-loop.png" alt="Vault protocol" width="100%" >}}

As seen on the image above, we can divide this function into different logical parts which we will analyze separately:

- Setup of local vars
- Calculation of current tick ratio
- Addition of new debt and collateral for liquidation
- Update of `tickHasDebt` and next active tick search
- Finding the reference tick
- Finding the reference ratio
- Calculation of debt and collateral to liquidate
- Finish of liquidation loop
- Updating amounts and factors for next iteration
- Merging branches
- Updating tick and ratio for next iteration

## Setup of local vars

```solidity
uint debtLiquidated_;
uint colLiquidated_;
uint debtFactor_ = BigMathVault.TWO_POWER_64;

TickHasDebt memory tickHasDebt_;
unchecked {
    tickHasDebt_.mapId = (currentData_.tick < 0)
        ? (((currentData_.tick + 1) / 256) - 1)
        : (currentData_.tick / 256);
}
```

We start slowly by setting a few local variables that will be used later. Then we find the `mapId` of the `tickHasDebt` mapping for current `topTick`.

## Calculation of current tick ratio

{{< figure src="/fluid-vault/vault/liq-calculation-of-current-tick-ratio.png" alt="Vault protocol" width="100%" >}}

Current `topTick` from which we are starting the liquidation can either be liquidated or not.

If it is not liquidated, then we know it is a perfect tick (no partial) with some debt, so we can read the ratio directly. We also set `nextTick` to current tick, which will be used later.

If it is liquidated, then `topTick` is also `minimaTick`, and it will have some partial which we need to use in order to calculate the exact ratio.

Now there is one interesting case left, labeled at point 3 on the image above. In order to understand this check, we will cheat a bit, and jump to future code where we end liquidation at liquidation threshold.

```solidity
// End in liquidation threshold.
// finalRatio_ = currentData_.refRatio;
// Increasing liquidation threshold tick by 1 partial. With 1 partial it'll reach to the next tick.
// Ratio change will be negligible. Doing this as liquidation threshold tick can also be a perfect non-liquidated tick.
unchecked {
    tickInfo_.tick = currentData_.refTick + 1;
}
// Making partial as 1 so it doesn't stay perfect tick
tickInfo_.partials = 1;
// length is not needed as only partials are written to storage
```

When Fluid ends liquidation exactly at liquidation threshold, we don't want that liquidation tick to be a perfect tick.

Why?

Because there could also exist the same valid perfect tick with debt, so we would have a harder time picking the next tick in the liquidation loop.

So to avoid that, if we landed at perfect tick `X` as liquidation tick, we will deliberately set it to `X+1` and give it a partial of 1. This way we know that liquidation tick will never match a perfect tick.

That's good, but then we also need to handle that case later.

That's what we are doing with this check:

```solidity
if ((memoryVars_.liquidationTick + 1) == tickInfo_.tick && (tickInfo_.partials == 1)) revert ...
```

So when we are calculating the tick where we want to start liquidation, we don't want that tick to be `liquidationTick`, as that would mean we don't have any meaningful debt to liquidate in between.

## Addition of new debt and collateral for liquidation

{{< figure src="/fluid-vault/vault/liq-addition-of-new-debt.png" alt="Vault protocol" width="80%" >}}

We are entering the liquidation loop now!

The first thing we do is ask whether current tick is liquidated or not.

### Tick is not liquidated

If the tick is not liquidated, we know that debt will live in the latest generation in the `tickData` mapping.

So we extract the debt and mark the tick as liquidated. If you remember from before, we actually already touched this exact part of code when we were running liquidation loops [manually]({{< ref "../7-borrow-second-time/2-fetch-liquidated-position/1-liquidations-and-factors.md">}}).

As soon as we touch the latest generation of `tickData` in the liquidation loop, we want to mark it as liquidated and store the current branch and debt factor.

```solidity
tickData[currentData_.tick] =
    1 | // set tick as liquidated
    (temp2_ & 0x1fffffe) | // set same total tick ids
    (branch_.id << 26) | // branch id where this tick got liquidated
    (branch_.debtFactor << 56);
```

### Tick is liquidated

If the tick is liquidated, debt will live in the branch itself, so we just read the debt from the branch:

```solidity
temp_ = (branch_.data >> 52) & X64;
// Converting big number into normal number
temp_ = (temp_ >> 8) << (temp_ & X8);
```

### Adding debt

`currentData_.debt` will be the variable to store all debt in consideration to be liquidated. So at any moment, we can't liquidate more than this variable.

In this case, we are adding new debt for the first time in the first liquidation loop:

```solidity
// Adding new debt into active debt for liquidation
currentData_.debt += temp_;
```

### Adding collateral

The same way, `currentData_.col` variable will hold active collateral for liquidation.

Here, we update it for the first time by calculating it from debt and ratio:

```solidity
// Adding new col into active col for liquidation
// Ratio is in 2**96 decimals hence multiplying debt with 2**96 to get proper collateral
currentData_.col += (temp_ * TickMath.ZERO_TICK_SCALED_RATIO) / currentData_.ratio;
```

## Update of tickHasDebt and next active tick search

{{< figure src="/fluid-vault/vault/liq-update-tick-has-debt.png" alt="Vault protocol" width="80%" >}}

The primary purpose of this block is to update the `tickHasDebt` mapping for current tick and to give us the next active tick.

It runs in 2 cases:

- `tickHasDebt` is empty, meaning we need to fetch the next non-zero 256-bit index
- current tick is active tick, which means we need to remove it from the `tickHasDebt` mapping as it is getting liquidated

If empty, we start by fetching `tickHasDebtIndex`:

```solidity
if (tickHasDebt_.tickHasDebt == 0) {
    tickHasDebt_.tickHasDebt = tickHasDebt[tickHasDebt_.mapId];
}
```

Then, if current tick is not already liquidated, we remove it from the `tickHasDebt` mapping by locating the exact bit to clear in the bitmap index:

```solidity
if (currentData_.tickStatus == 1) {
    unchecked {
        tickHasDebt_.bitsToRemove = uint(-currentData_.tick + (tickHasDebt_.mapId * 256 + 256));
    }
    // Removing current top tick from tickHasDebt
    tickHasDebt_.tickHasDebt =
        (tickHasDebt_.tickHasDebt << tickHasDebt_.bitsToRemove) >>
        tickHasDebt_.bitsToRemove;
    // Updating in storage if tickHasDebt becomes 0.
    if (tickHasDebt_.tickHasDebt == 0) {
        tickHasDebt[tickHasDebt_.mapId] = 0;
    }
}
```

In both cases, the next step will be to find the next active tick. The reason we need this is because every liquidation loop will try to find the next highest tick as liquidation point. That highest liquidation point can be either some active tick in `tickHasDebt`, minima, or liquidation tick. So when comparing options we are about to see in a moment, we need to know what is the next highest active tick from the `tickHasDebt` mapping.

The while loop just iterates over bitmap and it either breaks when we find the next active tick:

```solidity
if (tickHasDebt_.tickHasDebt > 0) {
    unchecked {
        tickHasDebt_.nextTick =
            tickHasDebt_.mapId *
            256 +
            int(tickHasDebt_.tickHasDebt.mostSignificantBit()) -
            1;
    }
    break;
}
```

or when the next active tick is below liquidation tick, in which case we can assign it to minimum value as we know this tick won't be liquidated in this loop nor it will be a valid next liquidation point:

```solidity
if ((tickHasDebt_.mapId * 256) < memoryVars_.liquidationTick) {
    tickHasDebt_.nextTick = type(int).min;
    break;
}
```

## Finding the reference tick

{{< figure src="/fluid-vault/vault/reference-tick.png" alt="Vault protocol" width="80%" >}}

If you recall from [liquidations and factors page]({{< ref "../7-borrow-second-time/2-fetch-liquidated-position/1-liquidations-and-factors.md">}}), every liquidation iteration needs to know how much debt should be liquidated in the current iteration so that after debt / collateral composition moves to the next liquidation point.

That next liquidation point is called reference tick and can be 1 of 3 values:

1. Next active tick from `tickHasDebt` mapping
2. Base branch minima tick
3. Liquidation tick

The code above finds the max of these 3 values and sets the `refTickStatus` so we can differentiate between these 3 cases.

Note that comparisons are strict greater-than, meaning if we end at liquidation tick equal to active tick or minima, we will use liquidation tick and finish the liquidation loop.

So at a high level, at every iteration we have these 6 possible scenarios (without equal cases) for how these 3 points can be arranged:

{{< figure src="/fluid-vault/vault/find-reference-tick.png" alt="Vault protocol" width="80%" >}}

## Finding the reference ratio

{{< figure src="/fluid-vault/vault/find-reference-ratio.png" alt="Vault protocol" width="80%" >}}

Once we have reference tick, we need to calculate reference ratio.

This code above just reads the ratio from `TickMath.getRatioAtTick` and applies the partial if reference tick is minima.

## Calculation of debt and collateral to liquidate

{{< figure src="/fluid-vault/vault/calculate-debt-and-coll-to-liquidate.png" alt="Vault protocol" width="100%" >}}

We are starting from some tick with some debt and we want to liquidate just enough debt so that final ratio ends at that reference ratio.

This is the formula, as stated in comments:

```text
(debt - x) / (coll - x * collPerDebt) = refRatio
```

Multiply by `(coll - x * collPerDebt)` on both sides:

```text
(debt - x) = refRatio * (coll - x * collPerDebt)
(debt - x) = refRatio * coll - refRatio * x * collPerDebt
```

Move `x` to one side:

```text
refRatio * x * collPerDebt - x = refRatio * coll - debt
```

Factor out:

```text
x * (refRatio * collPerDebt - 1) = refRatio * coll - debt
```

Get `x`:

```text
     refRatio * coll - debt
x = ----------------------------
     refRatio * collPerDebt - 1
```

Both numerator and denominator are negative, which is quite well explained in comments:

> [!NOTE]
> "Calculation results of numerator & denominator is always negative which will cancel out to give positive output in the end so we can safely cast to uint.
> 
> **for nominator**:  
> ratioStart can only be >= ratioEnd, so first part can only be reducing `currentData_.debt`, leading to `currentData_.debt reduced - currentData_.debt original * 1e27` -> can only be a negative number.
>
> **for denominator:**  
> `currentData_.colPerDebt` and `currentData_.refRatio` are inversely proportional to each other.  
> The maximum value they can ever be is ~9.97e26, which is 0.3% away from 100%, because liquidation threshold + liquidation penalty can never be > 99.7%.  
> This can also be verified by going back from min / max ratio values further up where we fetch oracle price etc.
>
> As optimization, we can inverse nominator and denominator subtraction to directly get a positive number."

So we can multiply by `-1` and get:

```text
       debt - refRatio * coll
x = ---------------------------
     1 - refRatio * collPerDebt
```

We know that `coll` is starting collateral that can be substituted with: `debt / ratioStart`:

```text
       debt - (refRatio * debt / ratioStart)
x = -----------------------------------------
            1 - refRatio * collPerDebt
```

We can then multiply both sides with `1e27`:

```text
       1e27 * (debt - (refRatio * debt / ratioStart))
x = ---------------------------------------------------
            1e27 * (1 - refRatio * collPerDebt)
```

Now if we just expand denominator:

```text
1e27 * (1 - refRatio * collPerDebt) = 1e27 - 1e27 * refRatio * collPerDebt
```

We know from before that `collPerDebt` is stored in `1e27` decimals and `refRatio` is stored in `Q96` representation, so we get:

```text
1e27 - 1e27 * refRatio / Q96 * collPerDebt / 1e27
=
1e27 - collPerDebt * refRatio / Q96
```

Substituting this to final `x`:

```text
       (debt - (refRatio * debt / ratioStart))  * 1e27
x = ---------------------------------------------------
            1e27 - collPerDebt * refRatio / Q96
```

which matches exactly Solidity code above.

Note that once we know debt to liquidate, getting collateral to liquidate is straightforward:

```solidity
colLiquidated_ = (debtLiquidated_ * currentData_.colPerDebt) / 1e27;
```

----

Note that there is one more subtle implementation detail left:

```solidity
if (currentData_.debt == debtLiquidated_) {
    debtLiquidated_ -= 1;
}
```

This just prevents rounding issues and ensures that all future calculations do not push debt to zero.

A few of such calculations that we will see later:

- Final ratio calc:

```solidity
 temp_ =
    ((currentData_.debt - debtLiquidated_) * TickMath.ZERO_TICK_SCALED_RATIO) /
    (currentData_.col - colLiquidated_);
```

- Debt factor calc:

```solidity
debtFactor_ = (debtFactor_ * (currentData_.debt - debtLiquidated_)) / currentData_.debt;
```

## Updating amounts and factors for next iteration

Let's say we don't want to finish liquidation just yet, so after we calculate debt to liquidate, we need to update data for the next iteration.

{{< figure src="/fluid-vault/vault/update-amounts-and-factors.png" alt="Vault protocol" width="100%" >}}

We first start by subtracting debt liquidated from liquidator remaining debt to liquidate.

```solidity
currentData_.debtRemaining -= debtLiquidated_;
```

Then we update debt factor and branch debt factor which we already covered [here]({{< ref "../7-borrow-second-time/2-fetch-liquidated-position/1-liquidations-and-factors.md">}}):

```solidity
debtFactor_ = (debtFactor_ * (currentData_.debt - debtLiquidated_)) / currentData_.debt;
branch_.debtFactor = branch_.debtFactor.mulDivBigNumber(debtFactor_);
```

> [!Reminder]
> Every branch will start with debt factor of 1, and every iteration will:
>
> - lower that debt factor depending on the debt liquidated
> - store the new branch debt factor for next iteration
>
> This way, whenever we touch some tick, we immediately store the debt factor of current branch, which would represent the point when the tick got introduced in the liquidation loop.
>
> After liquidation finishes, the branch will have final end debt factor.
>
> Retrieving the user positions later will use starting debt factor from `tickData` and final branch debt factor.

Besides calculating factors, we update internal accounting variables that will tell us how much is liquidated so far and with how much debt and collateral we enter into the next liquidation loop:

```solidity
currentData_.totalDebtLiq += debtLiquidated_;
currentData_.debt -= debtLiquidated_;
currentData_.totalColLiq += colLiquidated_;
currentData_.col -= colLiquidated_;
```

## Merging branches

If our next reference tick happens to be base branch minima tick, we hit merge event.

{{< figure src="/fluid-vault/vault/merge-branches-code.png" alt="Vault protocol" width="80%" >}}

1. We first read base branch data along with base branch debt factor.
2. After that we calculate connection factor as:

```text
connectionFactor = baseBranchDebtFactor / currentBranchDebtFactor
```

3. Then we update current branch data in storage.

```solidity
branchData[branch_.id] =
    ((branch_.data >> 166) << 166) | // deleting debt / partials / minima tick
    2 | // setting as merged
    (connectionFactor_ << 116); // set new connectionFactor
```

Notice that we mark the branch as merged, and we delete the previous debt/partials and minima.

The reason we need to delete the debt is because we previously added it to active debt for liquidation, so it's no longer part of this branch. Also, the minima with partials loses its purpose once the branch gets merged, because the only minima important from that moment is the minima from the base branch this current branch gets merged into.

The most important thing to store will be `connectionFactor`, as it will allow us to safely move all positions from current branch to base branch and not lose info of who got liquidated and how much on the previous branch.

4. The fourth step would be to update the branch pointer for next iteration:

- we set the base branch as new active branch for next liquidation. Let's call this new branch `B`
- we update the new minima to be the minima of base branch of `B`.

{{< figure src="/fluid-vault/vault/new-ref-minima.png" alt="Vault protocol" width="80%" >}}

## Updating tick and ratio for next iteration

```solidity
// Making refTick as currentTick
currentData_.tick = currentData_.refTick;
currentData_.tickStatus = currentData_.refTickStatus;
currentData_.ratio = currentData_.refRatio;
```

For the next iteration, all that's left to do is mark the `refTick` as the current new tick. This means that in the next iteration, new `refTick` will be calculated from previous `refTick` and that's how loop iterates.

## Finish of liquidation loop

The only thing left to analyze in this liquidation loop is what happens when we finish it.

{{< figure src="/fluid-vault/vault/liq-final-loop-code.png" alt="Vault protocol" width="100%" >}}

The only way we can enter this final loop state is if:

- either we are left with no debt from liquidator
- or we reached the liquidation tick

which can be validated by the outer if:

```solidity
if (debtLiquidated_ >= currentData_.debtRemaining || currentData_.refTickStatus == 3)  ...
```

So let's analyze those 2 cases.

### No debt remaining for liquidation

We wanted to reach the next reference tick, but we don't have enough debt left from liquidator from previous liquidation iterations.

In that case, we calculate exact debt and collateral remaining and the exact ratio we ended up at:

```solidity
debtLiquidated_ = currentData_.debtRemaining;
colLiquidated_ = (debtLiquidated_ * currentData_.colPerDebt) / 1e27;

// Liquidating to debt. temp_ => final ratio after liquidation
// liquidatable debt - debtLiquidated / liquidatable col - colLiquidated
temp_ =
    ((currentData_.debt - debtLiquidated_) * TickMath.ZERO_TICK_SCALED_RATIO) /
    (currentData_.col - colLiquidated_);
```

Once we have ratio, we can calculate the tick:

```solidity
(tickInfo_.tick, tickInfo_.ratioOneLess) = TickMath.getTickAtRatio(temp_);
```

> [!NOTE]
>
> Remember that `TickMath.getTickAtRatio` rounds down!

Now we enter this first weird check:

```solidity
if ((tickInfo_.tick < currentData_.refTick) && (tickInfo_.partials == X30)) ...
```

So what does this do?

We know when we set [reference ratio and partials](#finding-the-reference-ratio) that partials of `X30` were used if `refTick` was active perfect or liquidation tick.

So this check is really just a sanity check that should not trigger in normal circumstances, as newly ended tick will be above `refTick` in this case. But if this ever triggers, we want to move final tick just slightly above `refTick`:

```solidity
// this situation might never happen
// if this happens then there might be some very edge case precision of few weis which is returning 1 tick less
// if the above were to ever happen then tickInfo_.tick only be currentData_.refTick - 1
// in this case the partial will be very very near to full (X30)
// increasing tick by 2 and making partial as 1 which is basically very very near to currentData_.refTick
unchecked {
    tickInfo_.tick += 2;
}
tickInfo_.partials = 1;
```

We can't increase the tick by `1` and set partial close to `~X30`, as that would not cross `refTick`. It would mean we increased the tick effectively by `0.9999...99`.

But instead, if we increase it by `2` and set partial to `1`, that would effectively mean we increased it by `1.0000...01`, which would be just enough to cross `refTick`.

Good, now let's see what happens when we don't hit this rare edge case.

First of all, as `TickMath.getTickAtRatio` rounds down, we want to increase the calculated tick by 1:

```solidity
// Increasing tick by 1 as final ratio will probably be a partial
++tickInfo_.tick;
```

Then, if reference tick is minima tick, we snapshot its partial value:

```solidity
temp2_ = (currentData_.refTickStatus == 2 && tickInfo_.tick == currentData_.refTick)
    ? tickInfo_.partials
    : 0;
```

You will notice one thing here. We don't just look if `refTick` is minima, but we also compare it against calculated tick.

Why?

Well, the main idea here is that both minima and calculated tick will have some partial. If they end in the same tick, we want to compare partials later to ensure that calculated tick is always strictly above `refTick`.

The next step is to calculate the partial for tick:

```solidity
tickInfo_.ratio = (tickInfo_.ratioOneLess * 10015) / 10000;
tickInfo_.length = tickInfo_.ratio - tickInfo_.ratioOneLess;
tickInfo_.partials = ((temp_ - tickInfo_.ratioOneLess) * X30) / tickInfo_.length;
```

Then, we ensure that calculated partial won't end up at perfect tick. We already talked about this [before](#calculation-of-current-tick-ratio), but we don't want final liquidation to end at perfect tick. Rather, every liquidation should end up at minima that will have some partial:

```solidity
// Taking edge cases where partial comes as 0 or X30 meaning perfect tick.
// Hence, increasing or reducing it by 1 as liquidation tick cannot be perfect tick.
tickInfo_.partials = tickInfo_.partials == 0
    ? 1
    : tickInfo_.partials >= X30
        ? X30 - 1
        : tickInfo_.partials;
```

Finally, we use previously stored `refTick` partial (if it was minima tick) to compare it against current partial:

```solidity
if (temp2_ > 0 && temp2_ >= tickInfo_.partials) {
    // if refTick is liquidated tick and hence contains partials then checking that
    // current liquidation tick's partial should not be less than last liquidation refTick
    // not sure if this is even possible to happen but adding checks to avoid it fully
    // if it reverts here then next liquidation on next block should go through fine
    revert FluidVaultError(ErrorTypes.Vault__LiquidationReverts);
}
```

So with this check, we want to guarantee that tick will be above `refTick`. They may have the same value, but partial of tick should be greater than `refTick` partial.

### End in liquidation tick

The other case is straightforward. We know that we ended up at `refTick`, which is liquidation tick, and the only thing left to do is to ensure that this `refTick` won't be perfect tick. Again, we do this by increasing it by 1 and setting the partial to 1, which was [already discussed](#calculation-of-current-tick-ratio).

```solidity
tickInfo_.tick = currentData_.refTick + 1;

// Making partial as 1 so it doesn't stay perfect tick
tickInfo_.partials = 1;
```

### Update factors and amounts

Once we have final tick, we can repeat the steps [we did during liquidation loops](#updating-amounts-and-factors-for-next-iteration) and update the factors with collateral and debt amounts. We won't repeat the steps as logic is the same.

### Update branch and vault data in storage

The final step would be to update branch and vault data.

We set current branch as liquidated, and we store previously calculated values:

- minima tick
- branch partials (which with `minimaTick` we ensured was above `refTick`)
- leftover debt
- final debt factor

```solidity
branchData[branch_.id] =
    ((branch_.data >> 166) << 166) |
    1 | // set as liquidated
    (temp2_ << 2) | // minima tick of branch
    (tickInfo_.partials << 22) |
    (currentData_.debt.toBigNumber(56, 8, BigMathMinified.ROUND_UP) << 52) | // branch debt
    (branch_.debtFactor << 116);
```

And for vault, we set the new `topTick`, label it as liquidated and set the current latest branch where liquidation ended.

```solidity
vaultVariables_ =
    ((vaultVariables_ >> 52) << 52) |
    2 | // set as liquidated
    (temp2_ << 2) | // top tick
    (branch_.id << 22);
break;
```

----------

Whoa, we finally finished the main liquidation loop, which marks our job 99% done! 🎉