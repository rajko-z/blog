---
title: "Borrow limits"
weight: 16
layout: book
hiddenInHomeList: true
---

Just like withdrawal limits, Fluid imposes restrictions on borrow amounts for protocols using the liquidity layer.

Borrow limit is the maximum amount that can be borrowed, and like withdrawals, this amount can expand over some time duration by a predefined percentage.

There is also a base borrow limit, which users can freely borrow up to. Up to the base limit, no limits are applied.

And one addition to this borrow limit is the hard cap value, which is the absolute maximum one user can have as borrow amount. After this value, no expansion is possible. This can be viewed as a hard cap in some other protocols.

## Example

Suppose Bob borrowed 50 units and maxed out his borrow limit at 50.

```
baseBorrowLimit = 20
expandDuration = 200 seconds
expandPercentage = 20%
hardMaxBorrowLimit = 70
bobBorrow = 50
borrowLimit = 50
immediatelyBorrowable = 50 - 50 = 0
```

At this point he can't borrow more, but let's say he waits for an additional 100 seconds. After this, the new borrow limit becomes:

```
50 + 50 * 20% / 2 = 55 
```

Max expansion is 60, but because half of the time duration passed, Bob can borrow up to half of the full expansion size.

The new state becomes:

```
baseBorrowLimit = 20
expandDuration = 200 seconds
expandPercentage = 20%
hardMaxBorrowLimit = 70
bobBorrow = 55
borrowLimit = 55
```

Now let's say Bob waits for 200 seconds, allowing full expansion to pass, leaving `borrowLimit` at:

```
55 + 55 * 20% = 66
```

This leaves Bob an additional 11 units to borrow (`66 - 55`), making the new state:

```
baseBorrowLimit = 20
expandDuration = 200 seconds
expandPercentage = 20%
hardMaxBorrowLimit = 70
bobBorrow = 66
borrowLimit = 66
```

Now, finally, Bob waits again for 200 seconds, which allows full expansion to:

```
66 * 120% = 79.2
```

But note that this value is now greater than the absolute cap of `70`, meaning Bob will be able to borrow only up to the hard cap:

```
70 - 66 = 4
```

Leaving the final state as:

```
baseBorrowLimit = 20
expandDuration = 200 seconds
expandPercentage = 20%
hardMaxBorrowLimit = 70
bobBorrow = 70
borrowLimit = 70
```

If Bob now decides to repay his `60` units, he will be left with `10` units.

If we still look at expansion, this would allow borrow limit up to `12`, or only `2` new units for borrow.

But because Bob is below `baseBorrowLimit`, he can borrow up to `10` more units immediately, pushing the borrow to the size of the base borrow limit.

## Implementation

Implementation is quite similar to the one with withdrawal limits and can also be found in the `LiquidityCalcs` library.

Similarly, we will analyze these 2 functions:

- `calcBorrowLimitBeforeOperate`
- `calcBorrowLimitAfterOperate`

### calcBorrowLimitBeforeOperate

This is the code we are looking at:

{{< figure src="/fluid-vault/liquidity-layer/calculate-borrow-limit-before.png" alt="Liquidity layer" width="100%" >}}

It first calculates the maximum expandable borrow limit:

```solidity
uint256 temp_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_EXPAND_PERCENT) & X14;
uint256 maxExpansionLimit_;
uint256 maxExpandedBorrowLimit_;
unchecked {
    // calculate max expansion limit: Max amount limit can expand to since last interaction
    // userBorrow_ needs to be at least 1e73 to overflow max limit of ~1e77 in uint256 (no token in existence where this is possible).
    maxExpansionLimit_ = ((userBorrow_ * temp_) / FOUR_DECIMALS);

    // calculate max borrow limit: Max point limit can increase to since last interaction
    maxExpandedBorrowLimit_ = userBorrow_ + maxExpansionLimit_;
}
```

Then it scales this maximum depending on how much time has passed since the last snapshot:

```solidity
// currentBorrowLimit_ = expandedBorrowableAmount + extract last set borrow limit
currentBorrowLimit_ =
    // calculate borrow limit expansion since last interaction for `expandPercent` that is elapsed of `expandDuration`.
    // divisor is extract expand duration (after this, full expansion to expandPercentage happened).
    ((maxExpansionLimit_ * temp_) /
        ((userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_EXPAND_DURATION) & X24)) + // expand duration can never be 0
    //  extract last set borrow limit
    BigMathMinified.fromBigNumber(
        (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_PREVIOUS_BORROW_LIMIT) & X64,
        DEFAULT_EXPONENT_SIZE,
        DEFAULT_EXPONENT_MASK
    );
```

After this, this value gets challenged by different caps.

First, we check if this new borrow limit is above the max expanded value. This can happen if more than the full duration time has passed, so we cap it back to the fully expanded value:

```solidity
// if timeElapsed is bigger than expandDuration, new borrow limit would be > max expansion,
// so set to `maxExpandedBorrowLimit_` in that case.
// also covers the case where last process timestamp = 0 (timeElapsed would simply be very big)
if (currentBorrowLimit_ > maxExpandedBorrowLimit_) {
    currentBorrowLimit_ = maxExpandedBorrowLimit_;
}
```

After this, we check if this new borrow limit is below the base limit. If yes, we cap it to base to allow borrow up to the base limit:

```solidity
// temp_ = extract base borrow limit, if current limit is below this, then set base limit as current limit
temp_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_BASE_BORROW_LIMIT) & X18;
temp_ = (temp_ >> DEFAULT_EXPONENT_SIZE) << (temp_ & DEFAULT_EXPONENT_MASK);
if (currentBorrowLimit_ < temp_) {
    return temp_;
}
```

Finally, this value is checked against the maximum hard cap stored for the user. In our case for Bob above, that was `70`. If yes, we cap it to maximum and return the final value for the current borrow limit.

```solidity
// temp_ = extract hard max borrow limit. Above this user can never borrow (not expandable above)
temp_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_MAX_BORROW_LIMIT) & X18;
temp_ = (temp_ >> DEFAULT_EXPONENT_SIZE) << (temp_ & DEFAULT_EXPONENT_MASK);
if (currentBorrowLimit_ > temp_) {
    return temp_;
}
```

### calcBorrowLimitAfterOperate

This is the code we are looking at:

{{< figure src="/fluid-vault/liquidity-layer/calc-borrow-limit-after.png" alt="Liquidity layer" width="100%" >}}

The primary purpose of this function is to decide the actual final borrow limit that will be stored after the borrow operation.

It receives the previously calculated current borrow limit and the new final post-operate borrow amount.

First, it calculates what would be the next borrow limit at full expansion with the post-operate borrow amount:

```solidity
uint256 temp_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_EXPAND_PERCENT) & X14; // (is in 1e2 decimals)

unchecked {
    // borrowLimit_ = calculate maximum borrow limit at full expansion.
    // userBorrow_ needs to be at least 1e73 to overflow max limit of ~1e77 in uint256 (no token in existence where this is possible).
    borrowLimit_ = userBorrow_ + ((userBorrow_ * temp_) / FOUR_DECIMALS);
}
```

Then it checks if this fully expanded limit is below base. If yes, it caps it to base and returns the base limit as the final borrow limit:

```solidity
// temp_ = extract base borrow limit
temp_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_BASE_BORROW_LIMIT) & X18;
temp_ = (temp_ >> DEFAULT_EXPONENT_SIZE) << (temp_ & DEFAULT_EXPONENT_MASK);

if (borrowLimit_ < temp_) {
    // below base limit, borrow limit is always base limit
    return temp_;
}
```

So if Bob from the example repaid all tokens, this wouldn't store 0 as the new borrow limit, essentially blocking new borrow, but it would rather push it back to base, which would allow up to new 20 borrow units of immediate borrow.

After this check, this fully expanded limit is checked against the max hard cap stored for the user, and caps it to max if it is above:

```solidity
// temp_ = extract hard max borrow limit. Above this user can never borrow (not expandable above)
temp_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_MAX_BORROW_LIMIT) & X18;
temp_ = (temp_ >> DEFAULT_EXPONENT_SIZE) << (temp_ & DEFAULT_EXPONENT_MASK);

// make sure fully expanded borrow limit is not above hard max borrow limit
if (borrowLimit_ > temp_) {
    borrowLimit_ = temp_;
}
```

And finally, it takes the lesser value between the newly fully expanded value and previously calculated borrow limit from the `calcBorrowLimitBeforeOperate` function.

```solidity
// if new borrow limit (from before operate) is > max borrow limit, set max borrow limit.
// (e.g. on a repay shrinking instantly to fully expanded borrow limit from new borrow amount. shrinking is instant)
if (newBorrowLimit_ > borrowLimit_) {
    return borrowLimit_;
}
return newBorrowLimit_;
```

But why?

Suppose this is the state:

```
bobBorrow = 50
currentBorrowLimit = 55
```

And Bob performs the borrow of 5 units, making it:

```
new bobBorrow = 55
```

Now the new fully expanded value would be (in this function called `borrowLimit`):

```
55 * 120% = 66
```

But because `55` is less than `66`, we keep `newBorrowLimit_`, which was already calculated before, because we don't want to allow immediate new borrow space as Bob already borrowed max and must wait for the expansion period to end.

Now let's say after this that Bob wants to perform the repay. So we have this state:

```
bobBorrow = 55
currentBorrowLimit = 55
```

And Bob wants to repay `35` units, making the new debt:

```
55 - 35 = 20
```

And new fully expanded value:

```
20 * 120% = 24
```

If we decide to always keep the previously calculated current borrow limit (`newBorrowLimit_`), this would allow Bob to immediately borrow more:

```
55 - 20 = 35
```

That's why we keep the newly expanded value after repay.

-----

This marks our explanation of borrow limits. To summarize, the concept is analogous to withdrawal limits. We also have a borrow limit with a base limit and one addition of max hard cap.

In the next page, we will see how this function gets used in the main `borrowOrPayback` function, which will be far simpler due to not having `decay` amounts like we had before.