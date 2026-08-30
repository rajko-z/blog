---
title: "Borrow for the first time"
weight: 27
layout: book
hiddenInHomeList: true
---

We are doing borrow for the first time. We still only have a supply position, and our tick has the initial value of `type(int).min`.

We are only going to scan relevant code that we deliberately skipped during the first supply scan of the main `operate` function: [Supply/Withdraw for the first]({{< ref "./2-first-supply-withdraw-no-debt.md" >}})

------------------

## Calculate debt raw

```solidity
(o_.liquidityExPrice, , o_.supplyExPrice, o_.borrowExPrice) = updateExchangePrices(o_.vaultVariables2);

{
    // supply or withdraw
    ...
}

// borrow or payback
if (newDebt_ > 0) {
    // borrow new debt, rounding up adding extra wei in debt
    temp_ = ((uint(newDebt_) * EXCHANGE_PRICES_PRECISION) / o_.borrowExPrice) + 1;
    // if borrow fee is 0 then it'll become temp_ + 0.
    // Only adding fee in o_.debtRaw and not in newDebt_ as newDebt_ is debt that needs to be borrowed from Liquidity
    // as we have added fee in debtRaw hence it will get added in user's position & vault's total borrow.
    // It can be collected with rebalance function.
    o_.debtRaw += temp_ + (temp_ * ((o_.vaultVariables2 >> 82) & X10)) / 10000;
    // final user's debt should not be above 2**128 bits
    if (o_.debtRaw > X128) {
        revert FluidVaultError(ErrorTypes.Vault__UserCollateralDebtExceed);
    }
}
```

We calculated exchange prices and want to calculate the debt raw amount.

Just like with the supply/withdraw case, we divide with borrow exchange price and round up in favor of the protocol:

```solidity
temp_ = ((uint(newDebt_) * EXCHANGE_PRICES_PRECISION) / o_.borrowExPrice) + 1;
```

Now, here is the interesting thing: there is an optional borrow fee that will be taken on the borrow amount. This percentage is stored in `vaultVariables2`.

```solidity
/// Next 10 bits => 82-91 => borrow fee. 100 = 0.01 = 1%. (max precision of 0.01%) (max borrow fee can be 10.23%). Fees on borrow.
```

As stated in the comment, this `debtRaw` can be bigger than the borrowed amount from the liquidity layer. So we see a possible difference between vault and liquidity accounting, but we also mentioned before that this difference can be collected with the `rebalance` function when we were talking about exchange prices.

------------------

## Assign new tick

We already said that positions are placed in ticks, so let's assign one for our newly generated debt.

{{< figure src="/fluid-vault/vault/assign-new-tick.png" alt="Vault protocol" width="100%" >}}

This time `if(o_.debtRaw > 0)` will get triggered, as we are not working with supply-only positions anymore.

This `if` block can be divided into 2 main parts:

- assigning a new tick
- checking for new top tick

Let's focus on the first part for now and the function `_addDebtToTickWrite`.

This is the body of the function:

{{< figure src="/fluid-vault/vault/add-debt-to-tick-write.png" alt="Vault protocol" width="100%" >}}

We can divide this function into 5 units:

1. Sanity check if debt is too low
2. Calculation of new tick and debt
3. Update of tick data
   - Case when tick is already initialized and not liquidated
   - Case when tick is liquidated
   - Increase tickID and initialize tick if needed
4. Sanity check if tick debt is too low
5. Update of `tickData` mapping

### 1. Sanity check if debt is too low

```solidity
if (netDebtRaw_ < 10000) {
    // thrown if user's debt is too low
    revert FluidVaultError(ErrorTypes.Vault__UserDebtTooLow);
}
```

We just do not want to support dust debt creation. This also makes sure that `newDebtRaw` will be at least `10000` units, which can prevent rounding and weird cases in future calculations.

### 2. Calculation of new tick and debt

```solidity
// tick_ & ratio_ returned from library is round down. Hence increasing it by 1 and increasing ratio by 1 tick.
uint ratio_ = (netDebtRaw_ * TickMath.ZERO_TICK_SCALED_RATIO) / totalColRaw_;
(tick_, ratio_) = TickMath.getTickAtRatio(ratio_);
unchecked {
    ++tick_;
    ratio_ = (ratio_ * 10015) / 10000;
}
userRawDebt_ = (ratio_ * totalColRaw_) >> 96;
rawDust_ = userRawDebt_ - netDebtRaw_;
```

We calculate the ratio by looking at current `netDebtRaw_` and `totalColRaw_`.

If we remember our example from before, we were supplying 1 ETH and borrowing 1000 USDC, and we were getting to ratio:

```
ratio = (1000 * 1e6) / (1 * 1e18) = 0.000000001
tick = math.log(1e-9, 1.0015) = -13825.869602414956
```

There is one important implementation detail: **the tick library is always rounding down!**

This means that the tick we get will be `-13826` and not `-13825`.

Of course, this is not something we want, as tick `-13826` is less risky than tick `-13825`, meaning we are undercutting user debt.

That's why once we fetch the round-down tick and ratio at that tick, we intentionally put the user position in the tick above, effectively performing the round up.

```solidity
unchecked {
    ++tick_;
    ratio_ = (ratio_ * 10015) / 10000;
}
```

Once we have the new exact ratio, we can calculate `userRawDebt_` that will be stored:

```solidity
userRawDebt_ = (ratio_ * totalColRaw_) >> 96;
```

Doing `>> 96` is because ratio is stored in `Q96` format as mentioned on the ticks [page]({{< ref "./4-ticks.md" >}}).

Because this new debt is higher than the actual borrow amount the user requested, we need to store dust debt so we don't lose information on actual debt:

```solidity
rawDust_ = userRawDebt_ - netDebtRaw_;
```


### 3. Update of tick data

Now that we calculated the user's tick and raw debt, we can start updating tick structures.

We first want to load the latest tick epoch and data. Remember that `tickData` mapping only stores the latest epoch, and that `tickId` stores liquidation history for other epochs.

```solidity
// Current state of tick
uint256 tickData_ = tickData[tick_];
tickId_ = (tickData_ >> 1) & X24;
```

### 3.1 Case when tick is already initialized and not liquidated

We first cover the case where tick is already initialized and not liquidated:

```solidity
uint tickNewDebt_;
if (tickId_ > 0 && tickData_ & 1 == 0) {
    // Current debt in the tick
    uint256 tickExistingRawDebt_ = (tickData_ >> 25) & X64;
    tickExistingRawDebt_ = (tickExistingRawDebt_ >> 8) << (tickExistingRawDebt_ & X8);

    // Tick's already initialized and not liquidated. Hence simply add the debt
    tickNewDebt_ = tickExistingRawDebt_ + userRawDebt_;
    if (tickExistingRawDebt_ == 0) {
        // Adding tick into tickHasDebt
        _updateTickHasDebt(tick_, true);
    }
}
```

`tickId > 0` tells us that the latest epoch was already initialized, otherwise it would be zero.

`tickData_ & 1 == 0` tells us that the latest epoch was not liquidated, as the first bit is still zero.

This means that there exists some epoch greater than zero and the first bit is not set to `1` in the latest epoch.

For this case, we just read total active debt in the latest epoch and convert it from BigNumber format:

```solidity
uint256 tickExistingRawDebt_ = (tickData_ >> 25) & X64;
tickExistingRawDebt_ = (tickExistingRawDebt_ >> 8) << (tickExistingRawDebt_ & X8);
```

Then we add the newly calculated debt, as this user will be part of the latest active epoch:

```solidity
tickNewDebt_ = tickExistingRawDebt_ + userRawDebt_;
```

And then finally, if the tick was empty, we update the `tickHasDebt` mapping, which will tell us info about all ticks containing some debt at the moment.

The curious reader is left to analyze the implementation detail of `_updateTickHasDebt`, but it is essentially just bitmap manipulation, so we are going to skip it.

### 3.2 Case when tick is liquidated

The second case tells us that there exists some initialized epoch, but that epoch is liquidated:

```solidity
// Liquidation happened or tick getting initialized for the very first time.
if (tickId_ > 0) {
    // Meaning a liquidation happened. Hence move the data to tickID
    unchecked {
        uint tickMap_ = (tickId_ + 2) / 3;
        // Adding 2 in ID so we can get right mapping ID. For example for ID 1, 2 & 3 mapping should be 1 and so on..
        // For example shift for id 1 should be 0, for id 2 should be 85, for id 3 it should be 170 and so on..
        tickId[tick_][tickMap_] =
            tickId[tick_][tickMap_] |
            ((tickData_ >> 25) << (((tickId_ + 2) % 3) * 85));
    }
}
```

Let's quickly recall one example from the page about [ticks data structures]({{< ref "./5-ticks-data-structures.md" >}}):

1. a few users were supplying to the `ETH/USDC` vault, and all positions were placed into tick `-13825` -> **epoch 0**
2. price went down suddenly, and all those users in tick `-13825` got liquidated -> **epoch 2**
3. price went up and new users got placed into tick `-13825` -> **epoch 3**

What's happening in the code above is that we are coming to the latest epoch that is liquidated, in this case **epoch 2**.

Now we don't want to put the new user with liquidated users from the latest epoch, that's why we need to snapshot the state of the previous liquidated one and create a new fresh epoch.

The code above does the first part. It snapshots the latest liquidated epoch state to the `tickId` mapping by copying the fields from the `tickData` mapping.

**This is the only place where the `tickId` mapping will be updated**. So liquidations will update the latest `tickData` mapping, and once new users get assigned to a tick with the latest liquidated epoch, this epoch will be snapshotted to history.

### 3.3 Increase tickID and initialize tick if needed

This part is an extension to the previous one. It triggers for both cases:

1. if the latest epoch is liquidated
2. if the latest epoch is not initialized

```solidity
// Increasing total ID by one
unchecked {
    ++tickId_;
}
tickNewDebt_ = userRawDebt_;

// Adding tick into tickHasDebt
_updateTickHasDebt(tick_, true);
```

Regardless if the latest epoch got liquidated or the epoch was not initialized, we need the new fresh epoch.

So we increase the `tickId` (number of epochs), give it the user debt, and put it in the `tickHasDebt` bitmap containing info about active tick debts.


### 4. Sanity check if tick debt is too low

```solidity
if (tickNewDebt_ < 10000) {
    // thrown if tick's debt/liquidity is too low
    revert FluidVaultError(ErrorTypes.Vault__TickDebtTooLow);
}
```

Same sanity check we had at the beginning of the function, but now on the updated `tickNewDebt`. Again, this is to remove edge cases when working with low liquidity ticks.

### 5. Update of tickData mapping

```solidity
tickData[tick_] = (tickId_ << 1) | (tickNewDebt_.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 25);
```

Finally, we update the `tickData` with latest epoch info and new debt converted to BigNumber format.

Now if we recall, `tickData` will also store info for liquidations of the latest epoch:

```solidity
/// If liquidated
/// The below 3 things are of last ID. This is to be updated when user creates a new position
/// Next 1 bit => 25 => Is 100% liquidated? If this is 1 meaning it was above max tick when it got liquidated (100% liquidated)
/// Next 30 bits => 26-55 => branch ID where this tick got liquidated
/// Next 50 bits => 56-105 => debt factor 50 bits (35 bits coefficient | 15 bits expansion)
```

In this update, this will be left empty, which makes sense as we created a fresh new epoch for the tick, which is not yet liquidated.

**But here is the question for the reader. In step 3.2 we copied the liquidation data from the latest epoch to the `tickId` history mapping, and now we clear all fields before that, including any existing debt for the previous latest epoch.**

**What happened to debt? Did we just lose the info?**

*Answer: We mentioned before, but liquidated debt will be part of branch data, which is a whole other topic we will cover soon.*

------------------

## After values sanity checks

After we assigned the new tick, we want to perform basic sanity checks:

1. If we are doing payback, new net debt must be less than old net debt.

```solidity
if (newDebt_ < 0) {
    // anyone can payback debt of any position
    // hence, explicitly checking the debt should decrease
    if ((o_.debtRaw - o_.dustDebtRaw) > o_.oldNetDebtRaw) {
        revert FluidVaultError(ErrorTypes.Vault__InvalidPaybackOrDeposit);
    }
}
```

2. If we are doing only a supply operation, the new ratio should be lowered as the position is less risky:

```solidity
if ((newCol_ > 0) && (newDebt_ == 0)) {
    // anyone can deposit collateral in any position
    // Hence, explicitly checking that new ratio should be less than old ratio
    if (
        (((o_.debtRaw - o_.dustDebtRaw) * TickMath.ZERO_TICK_SCALED_RATIO) / o_.colRaw) >
        ((o_.oldNetDebtRaw * TickMath.ZERO_TICK_SCALED_RATIO) / o_.oldColRaw)
    ) {
        revert FluidVaultError(ErrorTypes.Vault__InvalidPaybackOrDeposit);
    }
}
```

------------------

## Check for new topTick

We finished assigning the tick, but we are left with the second important check for `topTick`.

```solidity
if (o_.tick >= o_.topTick) {
    // Updating topTick in storage
    // temp_ => tick to insert in vault variables
    unchecked {
        temp_ = o_.tick < 0 ? uint(-o_.tick) << 1 : (uint(o_.tick) << 1) | 1;
    }
    if (vaultVariables_ & 2 == 0) {
        // Current branch not liquidated. Hence, just update top tick
        vaultVariables_ =
            (vaultVariables_ & 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc00000) |
            (temp_ << 2);
    } else {
        // Current branch liquidated
        // Initialize a new branch
        // temp2_ => totalBranchId_
        unchecked {
            temp2_ = ((vaultVariables_ >> 52) & X30) + 1; // would take 34 years to overflow if a new branch is created every second
        }
        // Connecting new active branch with current active branch which is now base branch
        // Current top tick is now base branch's minima tick
        branchData[temp2_] =
            (((vaultVariables_ >> 22) & X30) << 166) | // current branch id set as base branch id
            (((vaultVariables_ >> 2) & X20) << 196); // current top tick set as base branch minima tick
        // Updating new vault variables in memory with new branch
        vaultVariables_ =
            (vaultVariables_ & 0xfffffffffffffffffffffffffffffffffffffffffffc00000000000000000000) |
            (temp_ << 2) | // new top tick
            (temp2_ << 22) | // new branch id
            (temp2_ << 52); // total branch ids
    }
}
```

**What is topTick?** -> It's the most risky current tick that will be the first one when liquidations start to happen.

**What is branch?** -> Branch is just a way to track liquidations and accumulate debt. We can view it similarly to tick epochs. Every vault that has been through liquidations will have a different set of branches.

So at every moment, the vault needs to track its most risky tick so it knows the starting point for liquidation. In this case, we want to see if the newly assigned tick to the user is riskier than the current `topTick`.

If that's the case, there are a few on-chain structures that we need to update.

There are 2 main cases in the code above:

1. Current branch is not liquidated
2. Current branch is liquidated

### 1. Current branch is not liquidated

Suppose this is the current state for some ETH / USDC vault.

{{< figure src="/fluid-vault/vault/active-branch.png" alt="Vault protocol" width="60%" >}}

We have some active branch `N` which is not liquidated, but connected to some previous liquidated branch `N - 1`.

For this case, we can ignore this previous connection and only focus on the current active branch having its top tick `-15000`.

Now let's take the example from before: user supplied 1 ETH and borrowed 1000 USDC, putting him in tick `-13825`.

Because this tick is bigger than current `topTick`, we need to update it for the current active branch `N`.

{{< figure src="/fluid-vault/vault/update-top-tick.png" alt="Vault protocol" width="60%" >}}

If we look at the code, it comes down to one update in `vaultVariables`:

```solidity
temp_ = o_.tick < 0 ? uint(-o_.tick) << 1 : (uint(o_.tick) << 1) | 1;

vaultVariables_ =
    (vaultVariables_ & 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc00000) |
    (temp_ << 2);
```

### 2. Current branch is liquidated

In this case, let's say the latest branch `N` went through liquidations from some previous `topTick` and finished at tick `-15000`. As mentioned before, this tick where liquidation finishes will be called `minimaTick`.

{{< figure src="/fluid-vault/vault/current-branch-liquidated.png" alt="Vault protocol" width="60%" >}}

In this case, we know that this `minimaTick` will also be `topTick`, because if there is any higher `topTick`, it would have to be part of a newer branch, but we are covering the case where the latest branch is liquidated, so there can't exist such newer branch.

Now, when a new `topTick` comes, we don't want to put the user in the latest liquidated branch, so we create a new fresh active branch and connect it with the previous branch:

{{< figure src="/fluid-vault/vault/current-branch-liquidated-new-top-tick.png" alt="Vault protocol" width="60%" >}}

For this `N + 1` branch, the branch `N` it got generated from will be called **`base branch`**.

The reason we need this connection will make sense later, but for now, it's enough to know that this connection allows us to merge branches back into their base branches, and allows us to track how positions moved through liquidations.

We can validate this with code:

```solidity
// Current branch liquidated
// Initialize a new branch
// temp2_ => totalBranchId_
unchecked {
    temp2_ = ((vaultVariables_ >> 52) & X30) + 1; // would take 34 years to overflow if a new branch is created every second
}
// Connecting new active branch with current active branch which is now base branch
// Current top tick is now base branch's minima tick
branchData[temp2_] =
    (((vaultVariables_ >> 22) & X30) << 166) | // current branch id set as base branch id
    (((vaultVariables_ >> 2) & X20) << 196); // current top tick set as base branch minima tick
// Updating new vault variables in memory with new branch
vaultVariables_ =
    (vaultVariables_ & 0xfffffffffffffffffffffffffffffffffffffffffffc00000000000000000000) |
    (temp_ << 2) | // new top tick
    (temp2_ << 22) | // new branch id
    (temp2_ << 52); // total branch ids
```

We first calculate the new branch id:

```solidity
unchecked {
    temp2_ = ((vaultVariables_ >> 52) & X30) + 1; // would take 34 years to overflow if a new branch is created every second
}
```

Then for this new branch, we snapshot its base branch and base branch minima tick.

For the example above, for branch `N + 1`, branch `N` would be the base branch and tick `-15000` would be the base branch minima tick.

```solidity
/// ...
/// Next 30 bits => 166-195 => Branch's ID with which this branch is connected. If 0 then that means this is the master branch
/// Next 1 bit => 196 => sign of minima tick of branch this is connected to. 0 if master branch.
/// Next 19 bits => 197-215 => minima tick of branch this is connected to. 0 if master branch.
mapping(uint256 => uint256) internal branchData;

// Connecting new active branch with current active branch which is now base branch
// Current top tick is now base branch's minima tick
branchData[temp2_] =
    (((vaultVariables_ >> 22) & X30) << 166) | // current branch id set as base branch id
    (((vaultVariables_ >> 2) & X20) << 196); // current top tick set as base branch minima tick
```

At the end, we store the new `topTick`, new latest branch id, and increased total number of branches in `vaultVariables`.

```solidity
// Updating new vault variables in memory with new branch
vaultVariables_ =
    (vaultVariables_ & 0xfffffffffffffffffffffffffffffffffffffffffffc00000000000000000000) |
    (temp_ << 2) | // new top tick
    (temp2_ << 22) | // new branch id
    (temp2_ << 52); // total branch ids
```

-----

Great, we covered the main part of this first-time borrow flow. We assigned the user the new tick and compared that tick to latest `topTick`. Now there are a few things left to cover which we couldn't have covered before for supply-only positions.

## Check Collateral Factor (Health)

This is the part of code we are analyzing:

```solidity
if (
    o_.debtRaw > 0 &&
    (o_.oldTick <= o_.tick ||
        (o_.debtRaw - o_.dustDebtRaw) > (((o_.oldNetDebtRaw * 1000000001) / 1000000000) + 1))
) {
    // Oracle returns price at 100% ratio.
    // converting oracle 160 bits into oracle address
    // temp_ => debt price w.r.t to col in 1e27
    temp_ = IFluidOracle(address(uint160(o_.vaultVariables2 >> 96))).getExchangeRateOperate();
    // Note if price would come back as 0 `getTickAtRatio` will fail

    // reverting if oracle price is too high or lower than 1e9 to avoid precision issues
    if (temp_ > 1e54 || temp_ < 1e9) {
        revert FluidVaultError(ErrorTypes.Vault__InvalidOraclePrice);
    }

    // Converting price in terms of raw amounts
    temp_ = (temp_ * o_.supplyExPrice) / o_.borrowExPrice;

    // capping oracle pricing to 1e45 (#487RGF783GF: id reference for other similar cases in codebase)
    // This means we are restricting collateral price to never go above 1e45
    // Above 1e45 precisions gets too low for calculations
    // This can will never happen for all good token pairs (for example, WBTC/DAI pair when WBTC price is $1M, oracle price will come as 1e43)
    // Restricting oracle price doesn't pose any risk to protocol as we are capping collateral price, meaning if price is above 1e45
    // user is simply not able to borrow more
    if (temp_ > 1e45) {
        temp_ = 1e45;
    }

    // temp2_ => ratio at CF. CF is in 3 decimals. 900 = 90%
    temp2_ = ((temp_ * ((o_.vaultVariables2 >> 32) & X10)) / 1000);

    // Price from oracle is in 1e27 decimals. Converting it into (1 << 96) decimals
    temp2_ = ((temp2_ * TickMath.ZERO_TICK_SCALED_RATIO) / 1e27);

    // temp3_ => tickAtCF_
    (temp3_, ) = TickMath.getTickAtRatio(temp2_);
    if (o_.tick > temp3_) {
        // Above CF, user should only be allowed to reduce ratio either by paying debt or by depositing more collateral
        // Not comparing collateral as user can potentially use safe/deleverage to reduce tick & debt.
        // On use of safe/deleverage, collateral will decrease but debt will decrease as well making the overall position safer.
        revert FluidVaultError(ErrorTypes.Vault__PositionAboveCF);
    }
}
```

We are checking the user collateral factor only if his position is getting riskier. That's either when he goes to a riskier tick or his new debt becomes higher than the previous one:

```solidity
(o_.oldTick <= o_.tick || (o_.debtRaw - o_.dustDebtRaw) > (((o_.oldNetDebtRaw * 1000000001) / 1000000000) + 1))
```

The first part fetches the oracle price:

```solidity
// Oracle returns price at 100% ratio.
// converting oracle 160 bits into oracle address
// temp_ => debt price w.r.t to col in 1e27
temp_ = IFluidOracle(address(uint160(o_.vaultVariables2 >> 96))).getExchangeRateOperate();
```

Note that oracle price is returning how much collateral is worth in debt units.

The price is scaled in `1e27 + debtTokenDecimals - collTokenDecimals`.

If we have an `ETH / USDC` vault, and price `1 ETH = 2500 USDC`, we can expect oracle to return:

```
2500 * 10**(27 + 6 - 18) = 2500 * 1e15
```

Now, here is the interesting part. Because this oracle price is basically returning price at 100% collateral ratio, we want somehow to find the tick at exactly vault collateral ratio.

For example, if collateral ratio inside the ETH/USDC vault is 90%, this means that the user can borrow a maximum of 90% of his collateral value. So including the oracle price, we want to find what would be the riskiest possible tick if the user decides to use full borrow power. Because if we find that, we can simply compare the current user tick with the maximum allowed tick.

This is exactly what Fluid is doing.

We know from before:

```
actual collateral = collateralRaw * supplyExPrice / 1e12
actual debt = debtRaw * borrowExPrice / 1e12
ratio = debtRaw / collateralRaw
```

Because oracle is returning in actual token amounts, we need to convert that to raw value.

```
actual debt = actual collateral * PRICE / 1e27
debtRaw * borrowExPrice / 1e12 = (collateralRaw * supplyExPrice / 1e12) * PRICE / 1e27
debtRaw * borrowExPrice = collateralRaw * supplyExPrice * PRICE / 1e27

// divide both sides with `collateralRaw * borrowExPrice`

debtRaw * borrowExPrice / (collateralRaw * borrowExPrice) = collateralRaw * supplyExPrice * PRICE / 1e27 / (collateralRaw * borrowExPrice)

debtRaw / collateralRaw = PRICE / 1e27 * supplyExPrice / borrowExPrice

ratio = PRICE / 1e27 * supplyExPrice / borrowExPrice
```

So this is exactly what this line is calculating (`/ 1e27` will be performed later):

```solidity
temp_ = (temp_ * o_.supplyExPrice) / o_.borrowExPrice;
```

Now that we have ratio at 100%, all we need to do is to multiply that ratio with `cf` that we read from vault variables:

```solidity
// temp2_ => ratio at CF. CF is in 3 decimals. 900 = 90%
temp2_ = ((temp_ * ((o_.vaultVariables2 >> 32) & X10)) / 1000);
```

And then adapt to `Q96` format so we can find the exact maximum allowed tick:

```solidity
// Price from oracle is in 1e27 decimals. Converting it into (1 << 96) decimals
temp2_ = ((temp2_ * TickMath.ZERO_TICK_SCALED_RATIO) / 1e27);

// temp3_ => tickAtCF_
(temp3_, ) = TickMath.getTickAtRatio(temp2_);
```

Once we have the tick, the health factor check comes down to a single line check:

```solidity
if (o_.tick > temp3_) revert FluidVaultError(ErrorTypes.Vault__PositionAboveCF);
```

-----

## Update user position in storage

The only difference from the supply-only case is that now the position has some debt and tick, and is not marked as supply-only, which means that the first bit of position data will be `0`:

```solidity
positionData[nftId_] =
    ((temp_ == 0) ? 1 : 0) | // setting if supply only position (1) or not (first bit)
    (temp_ << 1) |
    (o_.tickId << 21) |
    (o_.colRaw.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 45) |
    // dust debt is rounded down because user debt = debt - dustDebt. rounding up would mean we reduce user debt
    (o_.dustDebtRaw.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 109);
```

## Send debt

Finally, we can send debt tokens to the user from the liquidity layer:

```solidity
if (newDebt_ > 0) {
    // borrow
    LIQUIDITY.operate(BORROW_TOKEN, 0, newDebt_, address(0), to_, new bytes(0));
}
```

-----

We explored a lot on this page. So take your time to grasp the concepts. On the next page, we will see what happens when we change debt of an existing debt position.