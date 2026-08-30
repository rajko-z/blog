---
title: "Supply or withdraw"
weight: 12
layout: book
hiddenInHomeList: true
---

This is the complete body of the function we are analyzing.

{{< figure src="/fluid-vault/liquidity-layer/supply-or-withdraw-full-function.png" alt="Liquidity layer" width="100%" >}}

We can group it into 7 main parts labeled on the image, which we are going to analyze separately. But before we do that, we are going to introduce the concept of `decay amount`, which is the concept whose implementation represents the majority of the code above.

## What is decay amount and decay duration?

On the previous page, we analyzed withdrawal limits. So let's quickly recall it with one example.

```text
bobSupply = 15
expand = 20%
```

Fully expanded withdrawal is:

```text
15 * 80% = 12
```

Meaning Bob can withdraw the maximum of once expand duration finishes:

```text
15 - 12 = 3
```

Now, let's say Bob comes and deposits `10` more units, making the new withdrawal flow:

```text
25 * 80% = 20
```

Meaning Bob can withdraw the maximum of:

```text
25 - 20 = 5
```

where withdrawal limit jumped `8` units from:

```text
12 -> 20
```

This jump in withdrawal limit is called `decay amount`, and it can help as new temporarily available withdrawal.

### Decay amount

Let's say after this deposit Bob withdraws the maximum of 5 units, making the new state:

```text
bobSupply = 20
withdrawalLimit = 20
fully expanded withdrawal is = 20 * 80% = 16
```

Now if he wants to **withdraw again immediately**, he would need to wait for expansion to pass to push the limit down to `16`.

This is where `decay` becomes helpful. Instead of waiting for a new expand duration, Bob can immediately withdraw more.

When Bob comes and withdraws 5 units, the new `withdrawalLimit` won't be pushed down to `20` (effectively blocking an immediate new withdrawal), but it would rather account for the previously stored decay amount:

```text
temporaryWithdrawalLimit = 20 - 5 = 15
decayAmount = 8 - 5 = 3
```

Now because the expanded withdrawal is minimum of `16`, this `temporaryWithdrawalLimit` will be pushed back to `16`, and one unused unit pushed back to `decayAmount`, making the new final state:

```text
bobSupply = 20
withdrawalLimit = 16
decayAmount = 4
immediately withdrawable = 20 - 16 = 4
```

So after this 5 unit withdrawal, **we can see that immediate withdrawal is 4 now, which wasn't possible in the state without decay amount.**

As an experiment, if Bob immediately withdraws again in cycles, we get the following states.

State before:

```text
bobSupply = 20
withdrawalLimit = 16
decayAmount = 4
immediately withdrawable = 20 - 16 = 4
```

Bob withdraws 4, making:

```text
temporaryWithdrawalLimit = 16 - 4 = 12
decayAmount = 4 - 4 = 0
```

Because max expansion is:

```text
16 * 80% = 12.8
```

the limit and decay amount get pushed back to final values, making the final state:

```text
bobSupply = 16
withdrawalLimit = 12.8
decayAmount = 0.8
immediately withdrawable = 16 - 12.8 = 3.2
```

Now Bob withdraws 3.2 again immediately.

```text
newSupply = 16 - 3.2 = 12.8
```

Only `0.8` decay is left, so the code consumed all of it:

```text
temporaryWithdrawalLimit = 12.8 - 0.8 = 12
decayAmount = 0
```

The fully expanded limit for the new supply would be:

```text
12.8 * 80% = 10.24
```

But because decay amount can only push the existing limit down to `12`, it can't automatically push it down to the new expanded value (simply because we are only left with `0.8` value), so the new limit stays at:

```text
withdrawalLimitAfter = 12
```

Making the final state:

```text
bobSupply = 12.8
withdrawalLimit = 12
decayAmount = 0
immediately withdrawable = 0.8
```

Now, if Bob again withdraws this `0.8` immediately, we don't have `decayAmount` left, so the final state becomes:

```text
bobSupply = 12
withdrawalLimit = 12
decayAmount = 0
immediately withdrawable = 0
```

Now finally, Bob has to wait for expansion for new withdrawable liquidity. **He can't withdraw immediately anymore** as we used all of the decay amount.

If we add all withdrawal values we got:

```text
5 + 4 + 3.2 + 0.8 = 13
```

Which is exactly the previous max withdrawable amount of `3` plus the new deposit of `10`.

So with immediate withdrawables, **Bob was able to withdraw this newly deposited amount without waiting for expand durations.**

### Decay duration

We previously said that once we perform a deposit, this creates temporarily available withdrawal, but it does not mean that we can supply some units and expect that to be available all the time.

Fluid restricts this by introducing decay duration and epochs.

Decay duration is 1 hour long. Meaning, after a deposit is performed, the user has 1 hour where he can use `decayAmount` before it is fully gone.

Depending on how much time has passed, `decayAmount` gets linearly scaled.

`1` hour has `1000` epochs, meaning every epoch is `3.6` seconds long.

Looking at our example from above.

Let's say that Bob waited 2 minutes after he performed a deposit of `10` units. That's approximately `33` epochs.

Remaining decay would be calculated as:

```text
8 × (1000 - 33) / 1000
= 8 × 967 / 1000
= 7.736
```

So after waiting for `2` minutes, Bob's decay amount got lowered by `0.264`, meaning it will be faster until Bob comes to the point where he has to wait for the standard withdrawal expand duration to pass.

--------------

Now that we have a pretty good understanding of the idea behind decay amounts, we can start looking at implementation details.

## 1. Validation

```solidity
uint256 userSupplyData_ = _userSupplyData[msg.sender][token_];

if (userSupplyData_ == 0) {
    revert FluidLiquidityError(ErrorTypes.UserModule__UserNotDefined);
}
if ((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_IS_PAUSED) & 1 == 1) {
    revert FluidLiquidityError(ErrorTypes.UserModule__UserPaused);
}
```

The function begins with 2 simple checks. When we fetch user supply data, we perform:

1. User presence check. We revert if the user is not defined
2. If user supply is paused

Remember that consumers of this liquidity will be other protocols, so this can be a quick way to stop supply from some specific vault for example.

## 2. Fetch data

```solidity
// extract user supply amount
uint256 userSupply_ = (userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_AMOUNT) & X64;
userSupply_ = (userSupply_ >> DEFAULT_EXPONENT_SIZE) << (userSupply_ & DEFAULT_EXPONENT_MASK);

// get current leftover decaying amount
uint256 decayAmount_ = (userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_DECAY_AMOUNT) & X26;
decayAmount_ = (decayAmount_ >> DEFAULT_EXPONENT_SIZE) << (decayAmount_ & DEFAULT_EXPONENT_MASK);

// decay duration is in checkpoints. Also decay duration related constants are in Checkpoints, not in seconds!
uint256 decayDurationCPs_ = (userSupplyData_ >>
    LiquiditySlotsLink.BITS_USER_SUPPLY_DECAY_DURATION_CHECKPOINTS) & X10;
```

This part simply fetches the data from the user bitmap. It fetches:

- current user supply
- currently stored user `decayAmount`
- remaining decay duration checkpoints

## 3. Scale decay amount

```solidity
uint256 internal constant DECAY_CHECKPOINT_DURATION_SCALEDX10 = 36; // 3.6s * 10

if (decayAmount_ > 0) {
    unchecked {
        // calculate decay check points passed by scaling timestamps x 10
        // formula: (block.timestamp * 10 / 36) - (lastUpdateTimestamp * 10 / 36)
        // can not underflow as last timestamp can never be > block.timestamp and divisor can not be 0
        uint256 decayedCPs_ = ((block.timestamp * 10) / DECAY_CHECKPOINT_DURATION_SCALEDX10) -
            ((((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_LAST_UPDATE_TIMESTAMP) & X33) * 10) /
                DECAY_CHECKPOINT_DURATION_SCALEDX10);
        if (decayedCPs_ < decayDurationCPs_) {
            // only partial decay happened, update leftover decay amount
            decayAmount_ = decayAmount_ - (decayAmount_ * decayedCPs_) / decayDurationCPs_; // decayDurationCPs_ can not be 0. can not underflow
            decayDurationCPs_ = decayDurationCPs_ - decayedCPs_; // decayDurationCPs_ => decay duration checkpoints leftover
        } else {
            // full decay happened
            decayAmount_ = 0;
            decayDurationCPs_ = 0;
        }
    }
}
```

This part simply scales the decay amount left depending on how much time has passed since the last snapshot.

If there are more epochs left, the amount is scaled and new epochs left are calculated:

```solidity
decayAmount_ = decayAmount_ - (decayAmount_ * decayedCPs_) / decayDurationCPs_;
decayDurationCPs_ = decayDurationCPs_ - decayedCPs_;
```

Otherwise, decay and leftover epochs are reset and set to 0:

```solidity
decayAmount_ = 0;
decayDurationCPs_ = 0;
```

## 4. Fetch withdrawal limit and calculate new user supply

```solidity
// calculate current, updated (expanded etc.) withdrawal limit
uint256 withdrawLimitBefore_ = LiquidityCalcs.calcWithdrawalLimitBeforeOperate(userSupplyData_, userSupply_);

// calculate updated user supply amount
if (userSupplyData_ & 1 == 1) {
    // mode: with interest
    if (amount_ > 0) {
        // convert amount from normal to raw (divide by exchange price) -> round down for deposit
        newSupplyInterestRaw_ = (amount_ * int256(EXCHANGE_PRICES_PRECISION)) / int256(supplyExchangePrice_);
        userSupply_ = userSupply_ + uint256(newSupplyInterestRaw_);
    } else {
        // convert amount from normal to raw (divide by exchange price) -> round up for withdraw
        newSupplyInterestRaw_ = -int256(
            FixedPointMathLib.mulDivUp(uint256(-amount_), EXCHANGE_PRICES_PRECISION, supplyExchangePrice_)
        );
        // if withdrawal is more than user's supply then solidity will throw here
        userSupply_ = userSupply_ - uint256(-newSupplyInterestRaw_);
    }
} else {
    // mode: without interest
    newSupplyInterestFree_ = amount_;
    if (newSupplyInterestFree_ > 0) {
        userSupply_ = userSupply_ + uint256(newSupplyInterestFree_);
    } else {
        // if withdrawal is more than user's supply then solidity will throw here
        userSupply_ = userSupply_ - uint256(-newSupplyInterestFree_);
    }
}
```

This part simply calculates the withdrawal limit, which was covered on the previous page, and then calculates the new supply amount.

If the amount is with interest, we have to first multiply it by the exchange price. Otherwise, we can work with direct amounts.

Notice that this part works for supply and withdraw as well, where we perform the right scaling and integer conversion where needed.

Nothing special in this part, we can move on.

## 5. Apply decay amount

```solidity
bool checkDecayExpansion_;
if (amount_ < 0) {
    // withdrawal: check withdraw limit, take from decay if available, push down limit
    if (userSupply_ < withdrawLimitBefore_) {
        // if withdraw, then check the user supply after withdrawal is above withdrawal limit.
        // this check is in place also in case where decay is available as protocols do expect max withdrawable amount at once
        // is the fully expanded withdrawal limit (which == withdrawLimitBefore_ in case of decay at last tx)
        revert FluidLiquidityError(ErrorTypes.UserModule__WithdrawalLimitReached);
    }

    if (decayAmount_ > 0) {
        // subtract from decaying amount in case of withdrawal. the resulting withdrawal limit after must be pushed down by
        // the amount of withdraw amount that is covered by available decay amount.
        // withdraw limit after can end up either (see calcWithdrawalLimitAfterOperate()):
        // - 0 if supply below base
        // - withdraw limit before
        // - max expansion if withdraw limit before is < max expansion (can only happen in case of push down from decay or new deposits)
        // so, reducing withdrawLimitBefore_ makes it end up either at pushed down target or at max expansion.
        unchecked {
            uint256 withdrawAmount_ = uint256(-(newSupplyInterestRaw_ + newSupplyInterestFree_)); // only one of either can be set
            if (withdrawAmount_ > decayAmount_) {
                // withdrawal case A -> push down by full available decaying amount
                withdrawLimitBefore_ = withdrawLimitBefore_ > decayAmount_
                    ? withdrawLimitBefore_ - decayAmount_
                    : 0;
                decayAmount_ = 0;
            } else {
                // withdrawal case B -> push down by withdraw amount taken fully from decaying amount
                withdrawLimitBefore_ = withdrawLimitBefore_ > withdrawAmount_
                    ? withdrawLimitBefore_ - withdrawAmount_
                    : 0;
                decayAmount_ = decayAmount_ - withdrawAmount_;
            }
        }

        // Note not full amount taken from decay might be reflected in pushed down limit because of max expansion being hit
        //  -> handled below Ref #43681765878
        checkDecayExpansion_ = true;
    }
}
```

Part 5 gets interesting, as it is the first place where we start combining withdrawal limits with decay amounts.

We first perform the withdrawal check with new user supply to make sure the user is not going below the withdrawal limit.

```solidity
if (userSupply_ < withdrawLimitBefore_) {
    revert FluidLiquidityError(ErrorTypes.UserModule__WithdrawalLimitReached);
}
```

Then, if there is some decay amount stored from a previous interaction, we want to apply it on the newly calculated withdrawal limit.

If the withdrawal amount is greater than the decay amount, we consume the decay amount fully and lower the withdrawal limit.

```solidity
withdrawLimitBefore_ = withdrawLimitBefore_ > decayAmount_
    ? withdrawLimitBefore_ - decayAmount_
    : 0;
decayAmount_ = 0;
```

Same goes for the other direction, where we want to withdraw an amount lower than the decay amount.

This was the case in our example above where `Bob` was withdrawing `4` units on a decay amount of `8`.

```solidity
withdrawLimitBefore_ = withdrawLimitBefore_ > withdrawAmount_
    ? withdrawLimitBefore_ - withdrawAmount_
    : 0;
decayAmount_ = decayAmount_ - withdrawAmount_;
```

And finally, there is this line:

```solidity
//  -> handled below Ref #43681765878
checkDecayExpansion_ = true;
```

This will be used later to check if we went below the fully expanded value, in which case we would need to return units back to decay amount.

Remember that this was the case on the first withdrawal of `5` units after storing `8` as decay amount.

Bob was consuming `5` units, pushing the limit to:

```text
withdrawalLimit = 20 - 5 = 15
decayAmount = 8 - 5 = 3
```

But the fully expanded value was:

```text
20 * 80% = 16
```

Meaning we had to take `1` unit and adjust the final state to:

```text
withdrawalLimit = 16
decayAmount = 4
```

This is what `checkDecayExpansion_` will check later, and because decay amount is present, it simply puts this variable to `true` for a later check.

## 6. Calculate final limits and decay amounts

```solidity
// calculate withdrawal limit to store as previous withdrawal limit in storage
uint256 withdrawLimitAfter_ = LiquidityCalcs.calcWithdrawalLimitAfterOperate(
    userSupplyData_,
    userSupply_,
    withdrawLimitBefore_
);

if (withdrawLimitAfter_ == 0) {
            // if after limit is 0 -> no decay neededar (below base limit anyway full withdrawal possible)
    decayAmount_ = 0;
} else {
    // limit after can only ever become 0, == before or > before. see calcWithdrawalLimitAfterOperate().
    // case 0 -> handled. case == -> nothing to do for deposit, target hit for decay withdrawal.
    if (withdrawLimitBefore_ != withdrawLimitAfter_) {
        if (amount_ > 0) {
            // add new decaying amount in case of excess deposit
            if (withdrawLimitBefore_ == 0) {
                // special case: if before was 0 -> use base withdrawal limit as before reference
                withdrawLimitBefore_ =
                    (userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_BASE_WITHDRAWAL_LIMIT) &
                    X18;
                withdrawLimitBefore_ =
                    (withdrawLimitBefore_ >> DEFAULT_EXPONENT_SIZE) <<
                    (withdrawLimitBefore_ & DEFAULT_EXPONENT_MASK);

                // in this case it is possible that after is < before! when user supply ends up slightly above base then expansion
                // of the limit can reach below base withdrwal limit! Ref #412521521521
            }

            if (withdrawLimitAfter_ > withdrawLimitBefore_) {
                uint256 newDecayAmount_;
                unchecked {
                    newDecayAmount_ = withdrawLimitAfter_ - withdrawLimitBefore_;
                }

                // new decay duration depends on ratio of leftover decay vs new decay, to solve this case:
                // current decay of 10M, 60% passed so 4M decay left. New excess deposit of 100k comes in. decay duration would restart
                // for the whole amount of 4.1M, stretching out the decay. This could compound.
                // With ratio duration instead:
                // 4M : 0.1M, so current leftover decay duration of 40% should have a 40x bigger factor than the 100% for the new amount
                // duration = (40% * 1 hour * 4M + 100% * 1 hour * 0.1M) / (4M + 0.1M) = 1492s = 24.87 minutes

                // decayDurationCPs_ here already is decay duration leftover (in checkpoints)
                decayDurationCPs_ =
                    (decayDurationCPs_ * decayAmount_ + TOTAL_DECAY_CHECKPOINTS * newDecayAmount_) /
                    (decayAmount_ + newDecayAmount_); // new target decay duration. always <= TOTAL_DECAY_CHECKPOINTS. newDecayAmount_ can not be 0

                if (decayDurationCPs_ < MIN_DECAY_DURATION_CHECKPOINTS) {
                    // decay duration after a new deposit is always between at least 4m48s and max 1 hour
                    decayDurationCPs_ = MIN_DECAY_DURATION_CHECKPOINTS;
                }

                decayAmount_ = decayAmount_ + newDecayAmount_;
            } else {
                // edge case because of base limit see above Ref #412521521521. no decay
                decayAmount_ = 0;
                decayDurationCPs_ = 0;
            }
        } else if (checkDecayExpansion_) {
            // Ref #43681765878 case of decay withdrawal: limit did not end up at target pushed down withdrawLimitBefore_.
            uint256 notPushedDownAmount_;
            unchecked {
                notPushedDownAmount_ = withdrawLimitAfter_ > withdrawLimitBefore_
                    ? withdrawLimitAfter_ - withdrawLimitBefore_
                    : 0;
            }
            decayAmount_ = decayAmount_ + notPushedDownAmount_;
        }
    }
}
```

This part is probably the most complex one in this function, so we will analyze it case by case.

So, first, we calculate the final withdrawal limit. Remember from the previous page that this function either returns the already calculated `withdrawalLimitBefore` or it adapts it to the fully expanded value in case the new supply pushes `withdrawalLimitBefore` below it.

So we know that `withdrawalLimitAfter >= withdrawalLimitBefore`.

After that, we immediately set `decayAmount = 0` if the withdrawal limit left is zero, because there is no point in decay amount if there are no limits. Users can withdraw the full amount then.

After this, we come to logic that will trigger only if `withdrawalLimitAfter != withdrawalLimitBefore`. There are 2 cases, depending on whether we are doing supply or withdraw.

### If we are doing withdraw

We want to check the part with `checkDecayExpansion_`:

```solidity
} else if (checkDecayExpansion_) {
    // Ref #43681765878 case of decay withdrawal: limit did not end up at target pushed down withdrawLimitBefore_.
    uint256 notPushedDownAmount_;
    unchecked {
        notPushedDownAmount_ = withdrawLimitAfter_ > withdrawLimitBefore_
            ? withdrawLimitAfter_ - withdrawLimitBefore_
            : 0;
    }
    decayAmount_ = decayAmount_ + notPushedDownAmount_;
```

This is exactly the case already discussed before in our example with Bob, where we initially pushed the limit below the fully expanded value, making:

```text
withdrawLimitBefore_ = 15
```

where the fully expanded value was:

```text
withdrawLimitAfter_ = 16
```

So this `1` unit difference will go back to `decayAmount` as `notPushedDownAmount_`.

### If we are doing deposit

We want to calculate what the new decay amount will be, just like we had the example where Bob's supply of `10` created `8` units of decay.

But there is one case we didn't cover: what if there is already a present decay amount?

Let's say there is current decay of `4M` with `40%` epochs left from the initial amount of `10M`.

And there is a new deposit coming of `100k`.

If this new deposit automatically starts the `4.1M` decay amount, then this system can be easily gamed, as users can simply deposit tiny amounts so they can keep a big decay amount running forever.

To prevent this, Fluid averages this so that the new duration left is calculated like:

```text
duration = (40% * 1 hour * 4M + 100% * 1 hour * 0.1M) / (4M + 0.1M) = 1492s = 24.87 minutes
```

This is exactly what is commented and seen in the code. We first get the current new decay amount as the difference between limits:

```solidity
uint256 newDecayAmount_;
unchecked {
    newDecayAmount_ = withdrawLimitAfter_ - withdrawLimitBefore_;
}
```

Then we perform the described calculation:

```solidity
uint256 internal constant TOTAL_DECAY_CHECKPOINTS = 1e3;
decayDurationCPs_ =
    (decayDurationCPs_ * decayAmount_ + TOTAL_DECAY_CHECKPOINTS * newDecayAmount_) /
    (decayAmount_ + newDecayAmount_); // new target decay duration. always <= TOTAL_DECAY_CHECKPOINTS. newDecayAmount_ can not be 0
```

One small detail is that Fluid decides to cap this minimum decay duration to `80` epochs or `4m48s`, which means that after every deposit, decay duration will be between `80` and `1000` epochs.

```solidity
uint256 internal constant MIN_DECAY_DURATION_CHECKPOINTS = 80; 
if (decayDurationCPs_ < MIN_DECAY_DURATION_CHECKPOINTS) {
    // decay duration after a new deposit is always between at least 4m48s and max 1 hour
    decayDurationCPs_ = MIN_DECAY_DURATION_CHECKPOINTS;
}
```

#### Edge case when withdrawalLimitBefore == 0

There is one edge case when doing deposit where `withdrawalLimitBefore = 0`, meaning the user was either below base or didn't even have a limit set.

In this case, the code caps `withdrawalLimitBefore` to the base value.

```solidity
if (withdrawLimitBefore_ == 0) {
    // special case: if before was 0 -> use base withdrawal limit as before reference
    withdrawLimitBefore_ =
        (userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_BASE_WITHDRAWAL_LIMIT) &
        X18;
    withdrawLimitBefore_ =
        (withdrawLimitBefore_ >> DEFAULT_EXPONENT_SIZE) <<
        (withdrawLimitBefore_ & DEFAULT_EXPONENT_MASK);

    // in this case it is possible that after is < before! when user supply ends up slightly above base then expansion
    // of the limit can reach below base withdrwal limit! Ref #412521521521
}
```

This is done to prevent the scenario where all newly deposited amount is treated as decay amount.

This also creates a scenario where the fully expanded value can get lower than the base value, or in other words:

```text
withdrawalLimitAfter < withdrawalLimitBefore
```

In which case Fluid just clears the decay:

```solidity
else {
    // edge case because of base limit see above Ref #412521521521. no decay
    decayAmount_ = 0;
    decayDurationCPs_ = 0;
}
```

## 7. Cap decay and write values to storage

```solidity
if (decayAmount_ < 10) {
    decayAmount_ = 0;
    decayDurationCPs_ = 0;
} else {
    decayAmount_ = decayAmount_.toBigNumber(
        DECAY_COEFFICIENT_SIZE,
        DEFAULT_EXPONENT_SIZE,
        BigMathMinified.ROUND_DOWN
    );

    if (decayDurationCPs_ > TOTAL_DECAY_CHECKPOINTS) {
        decayDurationCPs_ = TOTAL_DECAY_CHECKPOINTS; // should not be possible but to be extra sure
    } else if (decayDurationCPs_ == 0) {
        decayDurationCPs_ = 1; // decay duration should at least always be minimum possible of 1 if decay amount exists (for checkpoints = 0.1% ~ 3.6 sec)
    }
}

// Converting user's supply into BigNumber
userSupply_ = userSupply_.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);
if (((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_AMOUNT) & X64) == userSupply_) {
    // make sure that operate amount is not so small that it wouldn't affect storage update. if a difference
    // is present then rounding will be in the right direction to avoid any potential manipulation.
    revert FluidLiquidityError(ErrorTypes.UserModule__OperateAmountInsufficient);
}

// Converting withdrawal limit into BigNumber
withdrawLimitAfter_ = withdrawLimitAfter_.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);

_userSupplyData[msg.sender][token_] =
    // mask to update bits 1-161 (supply amount, withdrawal limit, timestamp) and 218-253 (decay amount, decay duration percent)
    (userSupplyData_ & 0xC000000003FFFFFFFFFFFFFC0000000000000000000000000000000000000001) |
    (userSupply_ << LiquiditySlotsLink.BITS_USER_SUPPLY_AMOUNT) | // converted to BigNumber can not overflow
    (withdrawLimitAfter_ << LiquiditySlotsLink.BITS_USER_SUPPLY_PREVIOUS_WITHDRAWAL_LIMIT) | // converted to BigNumber can not overflow
    (block.timestamp << LiquiditySlotsLink.BITS_USER_SUPPLY_LAST_UPDATE_TIMESTAMP) |
    (decayAmount_ << LiquiditySlotsLink.BITS_USER_SUPPLY_DECAY_AMOUNT) | // converted to BigNumber can not overflow
    (decayDurationCPs_ << LiquiditySlotsLink.BITS_USER_SUPPLY_DECAY_DURATION_CHECKPOINTS); // can not overflow as can never be > TOTAL_DECAY_CHECKPOINTS
```

The final part of this function has 2 parts:

- capping the value of decay amount and epochs
- writing it all to storage

### Cap decay amounts

It first performs lower and upper bound checks on decay amounts.

For the lower side, `decayAmount` below `10` is considered dust, so the code explicitly clears it:

```solidity
if (decayAmount_ < 10) {
    decayAmount_ = 0;
    decayDurationCPs_ = 0;
}
```

For the upper side, decay duration can't be more than `1000` epochs or `1 hour`, and it can't be zero, in which case it's capped to at least `1` epoch.

```solidity
if (decayDurationCPs_ > TOTAL_DECAY_CHECKPOINTS) {
    decayDurationCPs_ = TOTAL_DECAY_CHECKPOINTS; // should not be possible but to be extra sure
} else if (decayDurationCPs_ == 0) {
    decayDurationCPs_ = 1; // decay duration should at least always be minimum possible of 1 if decay amount exists (for checkpoints = 0.1% ~ 3.6 sec)
}
```

### Writing to storage

This part first converts user supply to BigNumber format and performs the sanity check to see if the amount changed, to avoid small operate amounts which can round to zero:

```solidity
// Converting user's supply into BigNumber
userSupply_ = userSupply_.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);
if (((userSupplyData_ >> LiquiditySlotsLink.BITS_USER_SUPPLY_AMOUNT) & X64) == userSupply_) {
    // make sure that operate amount is not so small that it wouldn't affect storage update. if a difference
    // is present then rounding will be in the right direction to avoid any potential manipulation.
    revert FluidLiquidityError(ErrorTypes.UserModule__OperateAmountInsufficient);
}
```

After that, `withdrawalLimitAfter` is converted to BigNumber:

```solidity
// Converting withdrawal limit into BigNumber
withdrawLimitAfter_ = withdrawLimitAfter_.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);
```

And finally, storing it all together in `userSupplyData` bitmap storage:

```solidity
_userSupplyData[msg.sender][token_] =
    // mask to update bits 1-161 (supply amount, withdrawal limit, timestamp) and 218-253 (decay amount, decay duration percent)
    (userSupplyData_ & 0xC000000003FFFFFFFFFFFFFC0000000000000000000000000000000000000001) |
    (userSupply_ << LiquiditySlotsLink.BITS_USER_SUPPLY_AMOUNT) | // converted to BigNumber can not overflow
    (withdrawLimitAfter_ << LiquiditySlotsLink.BITS_USER_SUPPLY_PREVIOUS_WITHDRAWAL_LIMIT) | // converted to BigNumber can not overflow
    (block.timestamp << LiquiditySlotsLink.BITS_USER_SUPPLY_LAST_UPDATE_TIMESTAMP) |
    (decayAmount_ << LiquiditySlotsLink.BITS_USER_SUPPLY_DECAY_AMOUNT) | // converted to BigNumber can not overflow
    (decayDurationCPs_ << LiquiditySlotsLink.BITS_USER_SUPPLY_DECAY_DURATION_CHECKPOINTS); // can not overflow as can never be > TOTAL_DECAY_CHECKPOINTS
```

----
----

Congrats! We just finished the flow for supply and withdraw operations, where we covered Fluid's innovation around withdrawal limits and decaying amounts.

On the following pages, we will continue analyzing the main `operate` function, with borrow and payback operations coming next.