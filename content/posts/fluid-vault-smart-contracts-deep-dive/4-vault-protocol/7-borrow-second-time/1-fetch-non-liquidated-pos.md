---
title: "Fetch non liquidated position"
weight: 29
layout: book
hiddenInHomeList: true
---

We are fetching a user position which is not liquidated. Here is the code to look at:

{{< figure src="/fluid-vault/vault/fetch-non-liquidated-position.png" alt="Vault protocol" width="100%" >}}

----

## 1. Fetch tick debt

First, we load total raw debt for the current user's tick:

```solidity
// User didn't got liquidated
// Removing user's debt from tick data
// temp2_ => debt in tick
temp2_ = (temp_ >> 25) & X64;
```

Note that `temp_` is loaded tick data:

```solidity
/// ...
/// Next 64 bits => 25-88 => raw debt
/// ...
mapping(int256 => uint256) internal tickData;
```

----

## 2. Zero tick liquidity sanity check

Then there is one sanity check to revert if tick debt is `0`:

```solidity
// below require can fail when a user liquidity is extremely low (talking about way less than even $1)
// adding require meaning this vault user won't be able to interact unless someone makes the liquidity in tick as non 0.
// reason of adding is the tick has already removed from everywhere. Can removing it again break something? Better to simply remove that case entirely
if (temp2_ == 0) {
    revert FluidVaultError(ErrorTypes.Vault__TickIsEmpty);
}
```

As stated in comments, and as we saw earlier checks across the codebase, Fluid wants to minimize working with edge cases. So instead of supporting cases such as this one, it simply removes it so it does not need to think about nor write special code to handle it.

----

## 3. Remove user debt from tick

After that, we remove the current user debt from tick data.

```solidity
unchecked {
    temp2_ = o_.debtRaw < temp2_ ? temp2_ - o_.debtRaw : 0;
}
```

Why?

We already mentioned on the previous page, we want to close the current user debt position and reopen it in a new tick later, so that's why we need to clear the previous debt.

----

## 5. Update tick data and raw debt

Let's skip step 4 for now, and look at the 2 more lines left:

```solidity
tickData[o_.tick] = (temp_ & X25) | (temp2_.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 25);

// Converted positionRawDebt_ in net position debt
o_.debtRaw -= o_.dustDebtRaw;
```

We simply write new total lowered debt to the current tick and we calculate actual new net debt, from which we will later assign a new position and tick. We want net debt, so that the final outcome is the same as someone doing borrow for the first time, as the triggered code will be the same.

----

## 4. Handle low liquidity tick

Now we go back to step 4 and let's see what happens to tick when it's left with low liquidity.

Think about it, the user was part of some tick, we removed the user's debt from that tick, and now that tick is basically empty with no debt. What should we do with such a tick?

The obvious thing to do is to just remove it from the `tickHasDebt` mapping, which Fluid does in this line:

```solidity
// Removing from tickHasDebt
_updateTickHasDebt(o_.tick, false);
```

But there is one catch: what if this tick is also the current `topTick`? In that case, we would want to find its replacement, because `topTick` should represent the current riskiest tick with actual debt, so liquidations know where the frontier line is.

## Setting the new topTick

```solidity
if (o_.tick == o_.topTick) {
    // if tick is top tick then current top tick is perfect tick -> fetch & set new top tick

    // Updating new top tick in vaultVariables_ and topTick_
    (vaultVariables_, o_.topTick) = _setNewTopTick(o_.topTick, vaultVariables_);
}
```

If the current tick is `topTick`, we want to find a new tick by calling the internal `_setNewTopTick` function.

This is the code we are analyzing:

{{< figure src="/fluid-vault/vault/set-new-top-tick.png" alt="Vault protocol" width="100%" >}}

Before we start analyzing code, let's visualize different cases here:

1. Next top tick is on current active branch
2. Next top tick is on current active branch and matches base branch minima tick
3. Next top tick is base branch minima tick

### 1) Next top tick is on current active branch

{{< figure src="/fluid-vault/vault/non-liquidated-first-case.png" alt="Vault protocol" width="100%" >}}

As we can see on the image, there are 2 subcases.

1.1) If current branch has base branch, new `topTick` that has debt will be higher than base branch minima tick, so we will just update `topTick` in the current branch.

1.2) If there is no base branch, we just update `topTick` in the current branch with the next biggest tick with debt.

### 2) Next top tick is on current active branch and matches base branch minima tick

Now imagine that from first case `1.1)`, the next tick we found is exactly at base branch minima tick.

{{< figure src="/fluid-vault/vault/non-liquidate-second-case.png" alt="Vault protocol" width="60%" >}}

In this case, Fluid would use `topTick` from the current active branch, meaning it won't close the current branch and set base as current, although minima and current active tick with debt are the same.

This also makes sense if we recall the first borrow scenario, where we were assigning user tick on [this page]({{< ref "../6-borrow-first-time.md#assign-new-tick" >}}).

There was a case when we checked if the user tick is `>=` of current `topTick`:

```solidity
if (o_.tick >= o_.topTick) {
    ...
    if (vaultVariables_ & 2 == 0) {
        // Current branch not liquidated. Hence, just update top tick
    } else {
        ...
@=====>// Create new branch
    }
}
```

So if `tick == topTick`, we would create a new branch on top of the liquidated branch, which would have the same `topTick` as base branch minima tick.

So this is the exact state that we will end up in step `2)`, where we have base branch and current active branch in the same line.

### 3) Next top tick is base branch minima tick

In the final case, next top active tick will be below current branch minima tick.

In that case, we want to remove the current active branch, set the base branch as the current one, and set base branch minima tick as current `topTick`:

{{< figure src="/fluid-vault/vault/non-liquidated-third-case.png" alt="Vault protocol" width="100%" >}}

### Implementation

Now that we understand and visualized what's going on, let's quickly scan the implementation, which should be quite straightforward.

We first load base branch minima tick:

```solidity
uint branchId_ = (vaultVariables_ >> 22) & X30; // branch id of current branch

uint256 branchData_ = branchData[branchId_];
int256 baseBranchMinimaTick_;
if ((branchData_ >> 196) & 1 == 1) {
    baseBranchMinimaTick_ = int((branchData_ >> 197) & X19);
} else {
    unchecked {
        baseBranchMinimaTick_ = -int((branchData_ >> 197) & X19);
    }
    if (baseBranchMinimaTick_ == 0) {
        // meaning the current branch is the master branch
        baseBranchMinimaTick_ = type(int).min;
    }
}
```

Then, we fetch next top active tick:

```solidity
/// @dev gets next perfect top tick (tick which is not liquidated)
/// @param topTick_ current top tick which will no longer be top tick
/// @return nextTick_ next top tick which will become the new top tick
function _fetchNextTopTick(int topTick_) internal view returns (int nextTick_) {
    int mapId_;
    uint tickHasDebt_;

    unchecked {
        mapId_ = topTick_ < 0 ? ((topTick_ + 1) / 256) - 1 : topTick_ / 256;
        uint bitsToRemove_ = uint(-topTick_ + (mapId_ * 256 + 256));
        // Removing current top tick from tickHasDebt
        tickHasDebt_ = (tickHasDebt[mapId_] << bitsToRemove_) >> bitsToRemove_;

        // For last user remaining in vault there could be a lot of iterations in the while loop.
        // Chances of this to happen is extremely low (like ~0%)
        while (true) {
            if (tickHasDebt_ > 0) {
                nextTick_ = mapId_ * 256 + int(tickHasDebt_.mostSignificantBit()) - 1;
                break;
            }

            // Reducing mapId_ by 1 in every loop; if it reaches to -129 then no filled tick exist, meaning it's the last tick
            if (--mapId_ == -129) {
                nextTick_ = type(int).min;
                break;
            }

            tickHasDebt_ = tickHasDebt[mapId_];
        }
    }
}
```

Because the `tickHasDebt` structure is already structured by ticks, we can start searching from the riskiest tick and end the search with the first tick that has active debt.

Once we have the next top active tick, we need to compare it with base branch minima tick.

```solidity
newTopTick_ = baseBranchMinimaTick_ > nextTopTickNotLiquidated_
    ? baseBranchMinimaTick_
    : nextTopTickNotLiquidated_;
```

Now, if both `baseBranchMinimaTick_` and `nextTopTickNotLiquidated_` are `type(int).min`, that means we are on the master branch and no other users are present on any other active tick, meaning we removed debt from the last user in the vault. In that case, we mark the branch as not liquidated and set `topTick` as empty:

```solidity
if (newTopTick_ == type(int).min) {
    // if this happens that means this was the last user of the vault :(
    vaultVariables_ = vaultVariables_ & 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc00001;
```

Then, we move on to the final logic of setting `topTick`. For cases `1)` and `2)` that we visually covered, we set `topTick` as the current active `topTick` of the current branch:

```solidity
else if (newTopTick_ == nextTopTickNotLiquidated_) {
    // New top tick exist in current non liquidated branch
    if (newTopTick_ < 0) {
        unchecked {
            vaultVariables_ =
                (vaultVariables_ & 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc00001) |
                (uint(-newTopTick_) << 3);
        }
    } else {
        vaultVariables_ =
            (vaultVariables_ & 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc00001) |
            4 | // setting top tick as positive
            (uint(newTopTick_) << 3);
    }
```

And for case `3)`, where the new active tick is below base branch minima tick, we clear the current branch data and update the vault variables:

- set minima as `topTick`
- set base branch id as current latest id
- remove total branches for vault

```solidity
// if this happens that means base branch exists & is the next top tick
// Remove current non liquidated branch as active.
// Not deleting here as it's going to get initialize again whenever a new top tick comes
branchData[branchId_] = 0;
// Inserting liquidated branch's minima tick
unchecked {
    vaultVariables_ =
        (vaultVariables_ & 0xfffffffffffffffffffffffffffffffffffffffffffc00000000000000000001) |
        2 | // Setting top tick as liquidated
        (((branchData_ >> 196) & X20) << 2) | // new current top tick = base branch minima tick
        (((branchData_ >> 166) & X30) << 22) | // new current branch id = base branch id
        ((branchId_ - 1) << 52); // reduce total branch id by 1
```

------------

Great! We covered how fetching of a non liquidated position looks. Let's see what happens with liquidated positions on the next page.