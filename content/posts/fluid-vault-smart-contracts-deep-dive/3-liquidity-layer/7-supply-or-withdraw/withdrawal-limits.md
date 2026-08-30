---
title: "Withdrawal limits"
weight: 13
layout: book
hiddenInHomeList: true
---

In order to prevent sudden liquidity changes and improve security, Fluid introduces the concept of withdrawal limits.

Withdrawal limits are a mechanism through which a user can't withdraw all liquidity at once if it is above some base limit. Instead, the liquidity is gradually freed up by a certain percentage over a defined time duration.

The base limit controls when withdrawal limits are applied. If the supplied amount is lower than the base limit, then the whole amount of liquidity can be withdrawn at once. This ensures that lower supply amounts can be withdrawn fully because their impact on liquidity is negligible, while higher supply amounts must first go through partial withdrawals.

One important semantic detail across the codebase is that Fluid represents withdrawal limits as **liquidity that must stay in the protocol**, not liquidity that can be withdrawn.

So, the liquidity that a user can withdraw can be calculated by subtracting the withdrawal limit from their total supply:

```text
withdrawableLiquidity = totalSupply - withdrawalLimit
```

Let's look at a simplified example.

## Example

```text
bobSupply = 15
expandDuration = 200 seconds
expandPercentage = 20%
withdrawalLimit = 14
baseLimit = 10.5
```

Bob supplied `15` supply units and now wants to withdraw.

How much is that?

First, because `bobSupply` is greater than `baseLimit`, we know that we want to apply the withdrawal limit.

We know from before that the withdrawal limit must stay in the protocol, so if Bob wants to withdraw right now, he would be able to withdraw only `1` unit:

```text
bobSupply - withdrawalLimit = 15 - 14 = 1
```

But we also said that the amount available for withdrawal is being expanded, so let's say Bob does not withdraw yet, but rather waits `100` seconds.

We know that once `expandDuration` is fully over, Bob will be able to withdraw `expandPercentage` of his liquidity.

In this case, after waiting for `200` seconds, that would be:

```text
15 * 20% = 3
```

So waiting `100` seconds gives Bob additional withdrawal room of:

```text
0.5 * 3 = 1.5
```

This means that the new `withdrawalLimit` would be:

```text
14 - 1.5 = 12.5
```

Meaning Bob would be able to withdraw:

```text
15 - 12.5 = 2.5
```

supply units, and not only `1` like before.

After the withdrawal, the new state becomes:

```text
bobSupply = 12.5
expandDuration = 200 seconds
expandPercentage = 20%
withdrawalLimit = 12.5
baseLimit = 10.5
```

Now, if we do another cycle and wait for `200` more seconds, the new expanded amount will be:

```text
12.5 * 20% = 2.5
```

So if Bob withdraws the maximum amount again, we are left with the following state:

```text
bobSupply = 10
expandDuration = 200 seconds
expandPercentage = 20%
withdrawalLimit = 10
baseLimit = 10.5
```

Now, because `bobSupply` is less than `baseLimit`, the last `10` units can be withdrawn fully without starting a new expand duration.

## Implementation

Now that we have a general idea, let's look deeper into the concrete implementation.

The two main functions for handling withdrawal limits are both part of the `LiquidityCalcs` library:

- `calcWithdrawalLimitBeforeOperate`
- `calcWithdrawalLimitAfterOperate`

### calcWithdrawalLimitBeforeOperate

This is the function we are analyzing:

{{< figure src="/fluid-vault/liquidity-layer/calc-withdrawal-limit-before-operate.png" alt="Liquidity layer" width="100%" >}}

It takes the current user supply data and returns `currentWithdrawalLimit_`, which represents the amount of liquidity that must stay in the protocol and can't be withdrawn.

This function is called before user operation to fetch current withdrawal limit.

It first takes `lastWithdrawalLimit_` from the user bitmap and converts it from BigNumber format. If there is no previous withdrawal limit, it returns `0`, meaning the maximum withdrawal is allowed.

```solidity
// extract last set withdrawal limit
uint256 lastWithdrawalLimit_ = (userSupplyData_ >>
    LiquiditySlotsLink.BITS_USER_SUPPLY_PREVIOUS_WITHDRAWAL_LIMIT) & X64;
lastWithdrawalLimit_ =
    (lastWithdrawalLimit_ >> DEFAULT_EXPONENT_SIZE) <<
    (lastWithdrawalLimit_ & DEFAULT_EXPONENT_MASK);
if (lastWithdrawalLimit_ == 0) {
    // withdrawal limit is not activated. Max withdrawal allowed
    return 0;
}
```

Then it calculates `maxWithdrawableLimit_`. This represents the maximum withdrawable amount once the expand percentage is fully applied.

The expand percentage is read from the user bitmap:

```solidity
maxWithdrawableLimit_ =
    (((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_EXPAND_PERCENT) & X14) * userSupply_) /
    FOUR_DECIMALS;
```

In our example from above, `maxWithdrawableLimit_` would be:

```text
15 * 20% = 3
```

Once we have the maximum withdrawable amount, we can apply the time factor to it depending on the current timestamp and the last stored snapshot:

```solidity
temp_ = block.timestamp - ((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_LAST_UPDATE_TIMESTAMP) & X33);
temp_ =
    (maxWithdrawableLimit_ * temp_) /
    // extract expand duration: After this, decrement won't happen (user can withdraw 100% of withdraw limit)
    ((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_EXPAND_DURATION) & X24); // expand duration can never be 0
```

In our example, this was:

```text
3 * 0.5 = 1.5
```

Then we calculate `currentWithdrawalLimit_` by subtracting the previously calculated time-factored expanded amount from `lastWithdrawalLimit_`:

```solidity
unchecked {
    // underflow explicitly checked & handled
    currentWithdrawalLimit_ = lastWithdrawalLimit_ > temp_ ? lastWithdrawalLimit_ - temp_ : 0;
}
```

In our example, that was:

```text
14 - 1.5 = 12.5
```

Finally, we make sure that the withdrawal limit can't go below the minimum amount that must remain after the expand percentage is fully applied.

For example, if the time difference between the current timestamp and the last snapshot is greater than the expand duration, we would otherwise lower `currentWithdrawalLimit_` beyond the maximum allowed expanded percentage, which is something we want to avoid.

```solidity
temp_ = userSupply_ - maxWithdrawableLimit_;

// if withdrawal limit is decreased below minimum then set minimum
// (e.g. when more than expandDuration time has elapsed)
if (temp_ > currentWithdrawalLimit_) {
    currentWithdrawalLimit_ = temp_;
}
```

In our example, this was not the case, as `12.5` was greater than the fully expanded limit:

```text
15 * (1 - 20%) = 12
```

But if `currentWithdrawalLimit_` was lower than `12`, we would cap it to `12`.

### calcWithdrawalLimitAfterOperate

This is the function we are analyzing:

{{< figure src="/fluid-vault/liquidity-layer/calc-withdrawal-limit-after-operate.png" alt="Liquidity layer" width="100%" >}}

This function differs from the previous one, as it calculates the final withdrawal limit that will be written to storage after the user operation.

It takes the user supply data, the new supply amount after the operation, and the withdrawal limit before the operation that was calculated by `calcWithdrawalLimitBeforeOperate`.

This function first checks if the user is left with an amount lower than the base withdrawal limit:

```solidity
// temp_ => base withdrawal limit. below this, maximum withdrawals are allowed
uint256 temp_ = (userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_BASE_WITHDRAWAL_LIMIT) & X18;
temp_ = (temp_ >> DEFAULT_EXPONENT_SIZE) << (temp_ & DEFAULT_EXPONENT_MASK);

// if user supply is below base limit then max withdrawals are allowed
if (userSupply_ < temp_) {
    return 0;
}
```

If yes, it will write `0` to storage, meaning that on the next withdrawal the user will be able to withdraw the full amount without applying withdrawal limits.

After that, it calculates the fully expanded limit based on the new supply amount:

```solidity
// temp_ => withdrawal limit expandPercent (is in 1e2 decimals)
temp_ = (userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_EXPAND_PERCENT) & X14;
unchecked {
    // temp_ => minimum withdrawal limit: userSupply - max withdrawable limit (userSupply * expandPercent))
    // userSupply_ needs to be at least 1e73 to overflow max limit of ~1e77 in uint256 (no token in existence where this is possible).
    // subtraction can not underflow as maxWithdrawableLimit_ is a percentage amount (<=100%) of userSupply_
    temp_ = userSupply_ - ((userSupply_ * temp_) / FOUR_DECIMALS);
}
```

Finally, it checks whether the withdrawal limit before the operation (`newWithdrawalLimit_`) is lower than this fully expanded limit. If yes, it writes this expanded amount to storage. Otherwise, it keeps the already calculated `newWithdrawalLimit_` from `calcWithdrawalLimitBeforeOperate`.

```solidity
// if new (before operation) withdrawal limit is less than minimum limit then set minimum limit.
// e.g. can happen on new deposits. withdrawal limit is instantly fully expanded in a scenario where
// increased deposit amount outpaces withrawals.
if (temp_ > newWithdrawalLimit_) {
    return temp_;
}
```

-------

Now that we understand what withdrawal limits are and how they are implemented, let's see how these functions are used in the main `_supplyOrWithdraw` function, which we are going to analyze on the next page.