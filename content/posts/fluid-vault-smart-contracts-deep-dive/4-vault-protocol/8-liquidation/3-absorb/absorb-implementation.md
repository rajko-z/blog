---
title: "Absorb implementation"
weight: 38
layout: book
hiddenInHomeList: true
---

This is the function we are analyzing:

{{< figure src="/fluid-vault/vault/absorb.png" alt="Vault protocol" width="100%" >}}

The primary purpose of this function is to liquidate all active ticks and branches above `maxTick` and set a new state from where regular liquidation can continue later. 3 main parts are labeled on the image:

1. Liquidation of ticks above max ratio
2. Liquidation of branches above max ratio
3. Updating state with new `topTick` and `vaultVariables`

## 1. Liquidation of ticks above max ratio

This part is straightforward, although bitmap manipulation is a bit tricky. All it does is iterate over active ticks above `maxTick` and absorb debt and collateral.

Let's do a quick reminder where we are.

Remember from [here]({{< ref "../../5-ticks-data-structures.md#tickhasdebt" >}}) that information about active ticks with debt will be stored in one bitmap that can be indexed. Conceptually, it can be visualized like this:

```
...
mapId =  2  -> ticks    512 ... 767
mapId =  1  -> ticks    256 ... 511
mapId =  0  -> ticks      0 ... 255
mapId = -1  -> ticks   -256 ... -1
mapId = -2  -> ticks   -512 ... -257
...
```

If we take, for example, `mapId 2` with active ticks 522, 592 and 712, their offset and place in bitmap will be:

```
522 - 512 = 10
592 - 512 = 80
712 - 512 = 200
```

```solidity
000...001...000...001...000...00010000000000
        ^           ^            ^
       200          80           10
```

This also means that we don't need to iterate over the full ticks range [`[-32767, 32767]`]({{< ref "../../4-ticks.md#3-what-is-the-minimal-and-maximal-value-for-tick" >}}) one by one, which would be gas intensive, but rather jump to most significant bits in the bitmap and skip all empty ticks in between.

Also, the latest tick epoch lives in [`tickData`]({{< ref "../../5-ticks-data-structures.md#tickdata" >}}), so we are only touching active users that are part of the latest generation, not users which have their debt liquidated and stored in branch.

Now going back to code.

We first fetch `topTick` and bitmap index. The starting tick is increased by one because the later loop will find ticks below the starting tick, so we want `topTick` to be part of the loop:

```solidity
// temp_ -> top tick
temp_ = ((vaultVariables_ >> 2) & X20);
// increasing startingTick_ by 1 so the current tick comes into looping equation
a_.startingTick = (temp_ & 1) == 1 ? (int(temp_ >> 1) + 1) : (-int(temp_ >> 1) + 1);

tickHasDebt_.mapId = a_.startingTick < 0 ? ((a_.startingTick + 1) / 256) - 1 : a_.startingTick / 256;

tickHasDebt_.tickHasDebt = tickHasDebt[tickHasDebt_.mapId];
```

After that, we start searching.

We will have 2 while loops. The inner one will search ticks in one bitmap index, and the outer one will move between tick indexes. The inner one will contain most of the logic, but let's quickly go over the outer while loop first.

```solidity
// For last user remaining in vault there could be a lot of while loop.
// Chances of this to happen is extremely low (like ~0%)
tickHasDebt_.nextTick = TickMath.MAX_TICK;
while (true) {
    if (tickHasDebt_.tickHasDebt > 0) {
        ...
        // inner while loop
    }

    // tickHasDebt_.tickHasDebt == 0 from here.
    if (tickHasDebt_.nextTick <= maxTick_) {
        break;
    }

    if (tickHasDebt_.mapId < -129) {
        tickHasDebt_.nextTick = type(int).min;
        break;
    }

    // Fetching next tickHasDebt by decreasing tickHasDebt_.mapId first
    tickHasDebt_.tickHasDebt = tickHasDebt[--tickHasDebt_.mapId];
}
```

We just iterate over the `tickHasDebt` structure and we break:

1. if we found a valid active tick below or equal to `maxTick`
2. if we finished the search for all indexes

If there is some active tick in the current bitmap index, we will trigger the:

```solidity
if (tickHasDebt_.tickHasDebt > 0)
```

and from there on, we expand to the main while loop:

```solidity
a_.mostSigBit = tickHasDebt_.tickHasDebt.mostSignificantBit();
tickHasDebt_.nextTick = tickHasDebt_.mapId * 256 + int(a_.mostSigBit) - 1;

while (tickHasDebt_.nextTick > maxTick_) {
    // storing tickData into temp_
    temp_ = tickData[tickHasDebt_.nextTick];
    // temp2_ -> tick's debt
    temp2_ = (temp_ >> 25) & X64;
    // converting big number into normal number
    temp2_ = (temp2_ >> 8) << (temp2_ & X8);
    // Absorbing tick's debt & collateral
    a_.debtAbsorbed += temp2_;
    // calculating collateral from debt & ratio and adding to a_.colAbsorbed
    a_.colAbsorbed += ((temp2_ * TickMath.ZERO_TICK_SCALED_RATIO) /
        TickMath.getRatioAtTick(int24(tickHasDebt_.nextTick)));
    // Update tick data on storage. Making tick as 100% liquidated
    tickData[tickHasDebt_.nextTick] = 1 | (temp_ & 0x1fffffe) | (1 << 25); // set as 100% liquidated

    // temp_ = bits to remove
    temp_ = 257 - a_.mostSigBit;
    tickHasDebt_.tickHasDebt = (tickHasDebt_.tickHasDebt << temp_) >> temp_;
    if (tickHasDebt_.tickHasDebt == 0) break;

    a_.mostSigBit = tickHasDebt_.tickHasDebt.mostSignificantBit();
    tickHasDebt_.nextTick = tickHasDebt_.mapId * 256 + int(a_.mostSigBit) - 1;
}
// updating tickHasDebt on storage
tickHasDebt[tickHasDebt_.mapId] = tickHasDebt_.tickHasDebt;
```

We first locate the highest next active tick by reading the most significant bit:

```solidity
a_.mostSigBit = tickHasDebt_.tickHasDebt.mostSignificantBit();
tickHasDebt_.nextTick = tickHasDebt_.mapId * 256 + int(a_.mostSigBit) - 1;
```

Then, as long as we are reading the tick above `maxTick`, we will:

- load total debt from the tick's latest generation:

```solidity
// storing tickData into temp_
temp_ = tickData[tickHasDebt_.nextTick];
// temp2_ -> tick's debt
temp2_ = (temp_ >> 25) & X64;
// converting big number into normal number
temp2_ = (temp2_ >> 8) << (temp2_ & X8);
```

- absorb all of the debt and collateral:

```solidity
// Absorbing tick's debt & collateral
a_.debtAbsorbed += temp2_;
// calculating collateral from debt & ratio and adding to a_.colAbsorbed
a_.colAbsorbed += ((temp2_ * TickMath.ZERO_TICK_SCALED_RATIO) /
    TickMath.getRatioAtTick(int24(tickHasDebt_.nextTick)));
```

- mark the `tickData` as fully liquidated:

```solidity
// Update tick data on storage. Making tick as 100% liquidated
tickData[tickHasDebt_.nextTick] = 1 | (temp_ & 0x1fffffe) | (1 << 25); // set as 100% liquidated
```

We already mentioned this before, but with absorb we want to clear all debt aggressively, so there is no logic of how much debt needs to be cleared, as the liquidator is not doing repayment here. We rather want to snapshot this in internal `absorbed` variables that will later be written to storage.

The rest of the code removes this tick from bitmap and loads the next tick at the most significant bit, as that will be the highest next active tick:

```solidity
// temp_ = bits to remove
temp_ = 257 - a_.mostSigBit;
tickHasDebt_.tickHasDebt = (tickHasDebt_.tickHasDebt << temp_) >> temp_;
if (tickHasDebt_.tickHasDebt == 0) break;

a_.mostSigBit = tickHasDebt_.tickHasDebt.mostSignificantBit();
tickHasDebt_.nextTick = tickHasDebt_.mapId * 256 + int(a_.mostSigBit) - 1;
```

This finishes the first part of the `absorb` function where we clear all positions above max tick and find the next active tick that's `<= maxTick`.

This can be represented visually:

{{< figure src="/fluid-vault/vault/absorb-first-part.png" alt="Vault protocol" width="80%" >}}

## 2. Liquidation of branches above max ratio

So we liquidated active ticks, but we didn't touch the branches whose `minimaTick` is also above max tick, because that's also the debt we need to absorb.

Let's understand which cases we have here:

1. current active branch is not liquidated and not connected to either branch
2. current active branch is not liquidated but connected to some other liquidated base branch
3. current active branch is liquidated

{{< figure src="/fluid-vault/vault/absorb-branch-scenarios.png" alt="Vault protocol" width="80%" >}}

### 2.1 Current active branch is not liquidated and not connected to either branch

For case 1, we simply don't have any branch to liquidate and we can skip this part entirely. This can also be validated in code. We first check if the branch is liquidated, and try to load the base branch minima. Because there is no base branch, minima will be set to `type(int).min`, so later the main while loop where we liquidate branches won't trigger:

{{< figure src="/fluid-vault/vault/absorb-2-1.png" alt="Vault protocol" width="100%" >}}

### 2.2 Current active branch is not liquidated but connected to some other liquidated base branch

{{< figure src="/fluid-vault/vault/absorb-2-2.png" alt="Vault protocol" width="100%" >}}

In this case, although the latest active branch is not liquidated, we will have a base branch minima tick which we want to compare with `maxTick`.

### 2.3 Current active branch is liquidated

{{< figure src="/fluid-vault/vault/absorb-2-3.png" alt="Vault protocol" width="100%" >}}

In this case, the latest active branch is liquidated, so `minimaTick` is `topTick` as well, which we can compare with `maxTick`.

### Main liquidate branches loop

For cases `2.2` and `2.3`, we have some `minimaTick` to compare against `maxTick`, so we run into the main liquidate branches loop:

{{< figure src="/fluid-vault/vault/absorb-liquidate-branches-loop.png" alt="Vault protocol" width="100%" >}}

The main purpose of this loop is to iterate over connected liquidated branches and clear debt as long as `minimaTick` is bigger than `maxTick`.

We can follow the steps:

1. First, we calculate exact `minimaRatio`. Of course, liquidation does not have to end on a perfect tick as we saw before, so we calculate [partials]({{< ref "../../7-borrow-second-time/2-fetch-liquidated-position/2-fetch-latest-position.md#53-calculate-collateral" >}}) as well
2. Second, we absorb all branch debt
3. Third, we absorb all collateral
4. Then, we close the branch. This case is interesting as closing the branch automatically marks all positions as [100% liquidated]({{< ref "../../7-borrow-second-time/2-fetch-liquidated-position/2-fetch-latest-position.md#4-branch-is-closed" >}}).
5. And finally, the last step is to set the next branch for a new iteration if there exists one. Otherwise, we end the loop

----

So what we really did here was absorb and close all connected branches above `maxTick`, which is the yellow branch graph on the image above. Besides that, in the first step we also liquidated all latest tick generations that were sitting on top of `maxTick`.

{{< figure src="/fluid-vault/vault/absorb-branches.png" alt="Vault protocol" width="60%" >}}

Now the only last thing to do is to see where we actually landed, and what the structure should be after all of these absorptions, which is exactly what the 3rd main part of the `absorb` function does.

## 3. Updating state with new `topTick` and `vaultVariables`

We want to find the next `topTick`.

That will be the highest of:

1. Either next active tick we found during absorption of latest generation of ticks above `maxTick`
2. Or minimaTick from the branch we ended up after closing all branches above `maxTick`

## 3.1. Next active tick is greater or equal to minimaTick

We can get to this part from all 3 cases we mentioned [before](#2-liquidation-of-branches-above-max-ratio), so let's analyze them separately.

### 3.1.1 We started from non-liquidated branch with no base branch

We are looking at the following code:

{{< figure src="/fluid-vault/vault/next-tick-bigger-than-minima-1.png" alt="Vault protocol" width="60%" >}}

We start by reading `nextTick`. If it is `type(int).min`, we will assign it to zero, because later once read again from storage, `0` will mean `type(int).min` in memory.

After that, because we started from a non-liquidated branch with no base branch, we know that `newBranchId` will be `0`.

Why?

Because new branches are only ever created once a new user comes with `topTick` greater than current minima. Recall this flow from [here]({{< ref "../../6-borrow-first-time.md#2-current-branch-is-liquidated" >}}).

In this case, we initialize a new branch and update vault variables:

- we set new `topTick` that matches next active tick
- increase the branch count from 0 to 1
- set the latest current branch to 1

Then, because we don't have any minima, we explicitly set that this branch `1` will not have any base branch:

```solidity
// new base branch does not have any connected branch
branchData[newBranchId_] = 0;
```

-------

*Now I have to mention one interesting edge case here. If we were about to stop this absorb flow right now, we would finish with branchId of 1. And let's say we run again new absorb from where we left, on the same parameters. In that case, branchId would be non zero, and we would actually hit this part of updating only `topTick`:*

`vaultVariables_ = ((vaultVariables_ >> 22) << 22) | (temp2_ << 2);`

*So economically, we didn't even need to increase the branch to 1 in the first case, as working with branch id 0 would still be valid. In any case, the final effect for users will be the same, regardless if branch stayed at 0 or 1.*

-------

### 3.1.2 We started from non-liquidated branch that has base branch

{{< figure src="/fluid-vault/vault/next-tick-bigger-than-minima-2.png" alt="Vault protocol" width="60%" >}}

In this case, we started with a non zero branch id that was connected to some base branch.

We read the `topTick`, but this time, just update it directly in `vaultVariables` without initializing a new branch.

```solidity
// using already initialized non liquidated branch
vaultVariables_ = ((vaultVariables_ >> 22) << 22) | (temp2_ << 2);
```

At the end, we update the base branch connection. If we closed all branches during [step 2](#2-liquidation-of-branches-above-max-ratio), we would be left with no base branch minima, in which case we remove the connection:

```solidity
branchData[newBranchId_] = 0;
```

Otherwise, we connect the existing active branch with the new base branch found during the close loop:

```solidity
temp2_ = branch_.minimaTick < 0
    ? (uint(-branch_.minimaTick) << 1)
    : ((uint(branch_.minimaTick) << 1) | 1);
// set base branch id and minima tick
branchData[newBranchId_] = (branch_.id << 166) | (temp2_ << 196);
```

These 2 cases can be represented on the image below:

{{< figure src="/fluid-vault/vault/absorb-312.png" alt="Vault protocol" width="80%" >}}

### 3.1.3 We started from liquidated branch

{{< figure src="/fluid-vault/vault/next-tick-bigger-than-minima-3.png" alt="Vault protocol" width="60%" >}}

In this case, the only difference from before is that we started from a liquidated branch. That means that `newBranchId == 0` and that the new top tick will have to be part of a new branch. As in the previous step, this branch can have (or not have) a base branch depending on where the close while loop stopped.

## 3.2. Next active tick is lower than minimaTick

Now we are left with the other case, when `minimaTick` we got after closing the branches is bigger than next active tick.

In this case, we know that to be able to have some minima, the initial branch was either liquidated or had an existing base branch with minima, so we are going to analyze those 2 cases.

### 3.2.1 We started from non-liquidated branch with base branch

{{< figure src="/fluid-vault/vault/absorb-321.png" alt="Vault protocol" width="80%" >}}

In this case, we were starting from some active branch `N` that was having branch `N-1` as base branch. After absorbing debt, the end branch minima was bigger than next active tick. This means that we need to remove previous branch `N`, and only use some liquidated branch `X` (branch where absorption ended) with its `minimaTick` as `newTopTick`.

This is exactly what the code above does.

We first read `minimaTick`. After that, because `newBranchId != 0`, we update the vault variables with:

- minimaTick as new `topTick`
- end branch after close loop as current active branch
- total branches decreased by 1

Finally, we remove the connection from the previous active branch:

```solidity
branchData[newBranchId_] = 0;
```

This case can also be visualized on the image below:

{{< figure src="/fluid-vault/vault/absorb-321-image.png" alt="Vault protocol" width="50%" >}}

### 3.2.2 We started from liquidated branch

{{< figure src="/fluid-vault/vault/absorb-322.png" alt="Vault protocol" width="80%" >}}

There is one more case to cover. We landed at `minimaTick` greater than next active tick and we started from an already liquidated branch.

In that case, the only difference from the previous example is that we don't need to uninitialize the branch, as the starting branch was already liquidated. We update the new `topTick` pointing to end branch minima and we set that branch as the current latest branch for the vault.

```solidity
vaultVariables_ = ((vaultVariables_ >> 52) << 52) | 2 | (temp2_ << 2) | (branch_.id << 22);
```

-----

## End of absorb implementation

We finished `absorb` and the only last thing to do is to return from the function:

```solidity
// updating absorbed liquidity on storage
absorbedLiquidity = absorbedLiquidity + a_.debtAbsorbed + (a_.colAbsorbed << 128);

emit LogAbsorb(a_.colAbsorbed, a_.debtAbsorbed);

// returning updated vault variables
return vaultVariables_;
```

We update the `absorbedLiquidity` storage variable and return updated `vaultVariables` that will contain new `topTick` and new latest branch.

How this `absorbedLiquidity` is used will be clear on the next page.

----

We covered a lot. So take some time to process information and when you are ready, jump back to the main liquidation function we started analyzing.