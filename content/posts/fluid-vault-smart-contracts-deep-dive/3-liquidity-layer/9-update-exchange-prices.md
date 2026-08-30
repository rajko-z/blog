---
title: "Operate: Update exchange prices / utilization / ratios"
weight: 17
layout: book
hiddenInHomeList: true
---

Before we analyze, let's quickly revisit some of the things we covered earlier [here]({{< ref "./6-calculate-exchange-prices/liquiditycalcs-calc-exchange-prices.md" >}}).

- We previously brought exchange prices forward from the previous checkpoint to the current timestamp. During this action, we introduced the concept of `supplyRatio` and `borrowRatio`.

Remember that `supplyRatio` and `borrowRatio` always stay below 100% and represent the ratio between interest and interest-free supplying or borrowing.

```text
if supplyInterestFree > supplyWithInterest
    supplyRatio = supplyWithInterest / supplyInterestFree
else
    supplyRatio = supplyInterestFree / supplyWithInterest
```

or

```text
if borrowInterestFree > borrowWithInterest
    borrowRatio = borrowWithInterest / borrowInterestFree
else
    borrowRatio = borrowInterestFree / borrowWithInterest
```

All of this calculation was done on previously stored values for total supply/borrow, borrow rate, and utilization, before the new operate amount.

- After that, we updated total amounts with the current operate amount (supply/withdraw/borrow or payback), where we also saw how withdrawal and borrow limits work.

-------

Now that we have post-operate total amounts, we need to recalculate supply/borrow ratio, utilization, and store previously calculated supply and borrow exchange prices in storage.

One note, which will be seen immediately as the first comment in the next code snippet, is that Fluid does not automatically update all the values all the time.

There is a threshold update value, meaning for all values to be updated, either:

- 1 day passed from the previous checkpoint
- utilization is above the update threshold
- supply ratio is above the update threshold
- borrow ratio is above the update threshold

Knowing this, we can start analyzing the following code:

{{< figure src="/fluid-vault/liquidity-layer/utilization-update.png" alt="Liquidity layer" width="100%" >}}

The code can be divided into 6 main parts:

1. calculation of supply ratio
2. calculation of borrow ratio
3. calculation of utilization
4. 1 day difference check
5. check for storage update (update threshold)
6. storage update

Let's start analyzing the first part.

## 1. Calculation of supply ratio

First, we store the new supply with interest in the temp `memVar3_` variable and perform the already seen max token cap if we are doing supply. Note that `supplyRawInterest` is already updated with the post-operate amount.

```solidity
memVar3_ = ((o_.supplyRawInterest * o_.supplyExchangePrice) / EXCHANGE_PRICES_PRECISION);
if (memVar3_ > MAX_TOKEN_AMOUNT_CAP && supplyAmount_ > 0) {
    // only withdrawals allowed if total supply raw reaches MAX_TOKEN_AMOUNT_CAP
    revert FluidLiquidityError(ErrorTypes.UserModule__ValueOverflow__TOTAL_SUPPLY);
}
```

After this, we store total supply in `memVar_`

```solidity
// memVar_ => total supply. set here so supplyWithInterest (memVar3_) is only calculated once. For utilization
memVar_ = o_.supplyInterestFree + memVar3_;
```

Then we calculate new supplyRatio depending on which interest is bigger:

```solidity
if (memVar3_ > o_.supplyInterestFree) {
    // memVar3_ is ratio with 1 bit as 0 as supply interest raw is bigger
    memVar3_ = ((o_.supplyInterestFree * FOUR_DECIMALS) / memVar3_) << 1;
    // because of checking to divide by bigger amount, ratio can never be > 100%
} else if (memVar3_ < o_.supplyInterestFree) {
    // memVar3_ is ratio with 1 bit as 1 as supply interest free is bigger
    memVar3_ = (((memVar3_ * FOUR_DECIMALS) / o_.supplyInterestFree) << 1) | 1;
    // because of checking to divide by bigger amount, ratio can never be > 100%
} else if (memVar_ > 0) {
    // supplies match exactly (memVar3_  == o_.supplyInterestFree) and total supplies are not 0
    // -> set ratio to 1 (with first bit set to 0, doesn't matter)
    memVar3_ = FOUR_DECIMALS << 1;
} // else if total supply = 0, memVar3_ (supplyRatio) is already 0.
```

There are 4 cases:
- 1. `supplyWithInterest > supplyInterestFree` -> Store `supplyInterestFree/supplyWithInterest` and leave first bit to `0`.

Example:
```text
supplyWithInterest = 80
supplyInterestFree = 20
ratio = 20 / 80 = 25% = 2500
encoded value = 2500 << 1 = 5000
```

- 2. `supplyInterestFree > supplyWithInterest` -> Store  `supplyWithInterest/supplyInterestFree` and set first bit to `1`.
  
Example:
```text
supplyWithInterest = 20
supplyInterestFree = 80
ratio = 20 / 80 = 25% = 2500
encoded value = (2500 << 1) | 1 = 5001

```
- 3. `supplyInterestFree == supplyWithInterest` -> Store `100%` or more precisely: `10000 << 1`

- 4. `totalSupply == 0` -> Leave `memVar3_ = 0`

## 2. Calculation of borrow ratio

Completely equivalent with the supply side, we just use `borrowWithInterest` and `borrowInterestFree` variables.

Final borrow rate will be stored in the `memVar4_` variable and total borrow in `memVar2_`.

## 3. Calculation of utilization

Utilization is calculated as: `totalBorrow / totalSupply`.

If there is no supply, utilization stays at `0`.

In the calculations from above, we know that `memVar_` is total supply and `memVar2_` is total borrow, so this code calculates it:

```solidity
uint256 utilization_;
if (memVar_ > 0) {
    utilization_ = ((memVar2_ * FOUR_DECIMALS) / memVar_);
```

If we are doing a new borrow, we have to check max utilization as well:

```solidity
// for borrow operations, ensure max utilization is not reached
if (borrowAmount_ > 0) {
    if (utilization_ > memVar_) {
        revert FluidLiquidityError(ErrorTypes.UserModule__MaxUtilizationReached);
    }
}
```

Typically, this `memVar_` will be 100%, meaning max utilization will be capped to 100%, but there is an option to load optional config for a specific token, in which case an alternative max value will be used:

```solidity
memVar_ = (o_.exchangePricesAndConfig >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_USES_CONFIGS2) &
        1 ==
        1
        ? (_configs2[token_] & X14) // read configured max utilization
        : FOUR_DECIMALS; // default max utilization = 100%
```

## 4. Day difference check

```solidity
uint256 internal constant FORCE_STORAGE_WRITE_AFTER_TIME = 1 days;

if (
    block.timestamp >
    // extract last update timestamp + 1 day
    (((o_.exchangePricesAndConfig >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_LAST_TIMESTAMP) & X33) +
        FORCE_STORAGE_WRITE_AFTER_TIME)
) {
    memVar_ = 1; // set write to storage flag
} else {
    memVar_ = 0;
}
```

Code above simply checks what was the last time exchange prices were updated. If the time difference is above the constant of 1 day, then it forces a storage update by writing `1` to `memVar_`. Otherwise, it sets it to `0`.

## 5. Check for storage update

If storage was updated less than 1 day ago, we start checking the update threshold.

First, we check utilization:

```solidity
// memVar_ => extract last utilization
memVar_ = (o_.exchangePricesAndConfig >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_UTILIZATION) & X14;
// memVar2_ => storage update threshold in percent
memVar2_ =
    (o_.exchangePricesAndConfig >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_UPDATE_THRESHOLD) &
    X14;
unchecked {
    // set memVar_ to 1 if current utilization to previous utilization difference is > update storage threshold
    memVar_ = (utilization_ > memVar_ ? utilization_ - memVar_ : memVar_ - utilization_) > memVar2_
        ? 1
        : 0;
```

We read last utilization and update threshold values, and simply compare to see if the difference is bigger than the threshold.

If yes, we finish further checking and go straight to the storage update section. If not, we continue checking with supply ratio next.

```solidity
memVar_ =
    (o_.exchangePricesAndConfig >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_SUPPLY_RATIO) &
    X15;
// set memVar_ to 1 if current supplyRatio to previous supplyRatio difference is > update storage threshold
if ((memVar_ & 1) == (memVar3_ & 1)) {
    memVar_ = memVar_ >> 1;
    memVar_ = (
        (memVar3_ >> 1) > memVar_ ? (memVar3_ >> 1) - memVar_ : memVar_ - (memVar3_ >> 1)
    ) > memVar2_
        ? 1
        : 0; // memVar3_ = supplyRatio, memVar_ = previous supplyRatio, memVar2_ = update storage threshold
} else {
    // if inverse bit is changing then always update on storage
    memVar_ = 1;
}
```

We read the last stored supply ratio and compare it with the newly calculated one stored in `memVar3_`.

If both previous and new ratio have the same sign, meaning both are `supplyInterestFree / supplyWithInterest` or `supplyWithInterest / supplyInterestFree`, then we decode full values and compare them with the threshold.

Note that the same update threshold value is used for utilization as well.

If they are not the same sign, meaning one is `supplyInterestFree / supplyWithInterest` and the other is `supplyWithInterest / supplyInterestFree`, then Fluid considers this an important enough change to force update.

If `supplyRatio` is again not big enough in change, the last thing we check is borrow ratio the same way as supply ratio:

```solidity
if (memVar_ == 0) {
    // utilization, time, and supplyRatio difference is not big enough -> check borrowRatio difference
    // memVar_ => extract last borrowRatio
    memVar_ =
        (o_.exchangePricesAndConfig >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_BORROW_RATIO) &
        X15;
    // set memVar_ to 1 if current borrowRatio to previous borrowRatio difference is > update storage threshold
    if ((memVar_ & 1) == (memVar4_ & 1)) {
        memVar_ = memVar_ >> 1;
        memVar_ = (
            (memVar4_ >> 1) > memVar_ ? (memVar4_ >> 1) - memVar_ : memVar_ - (memVar4_ >> 1)
        ) > memVar2_
            ? 1
            : 0; // memVar4_ = borrowRatio, memVar_ = previous borrowRatio, memVar2_ = update storage threshold
    } else {
        // if inverse bit is changing then always update on storage
        memVar_ = 1;
    }
}
```

At the end of this block, `memVar_` will determine if we need to update storage or not.

## 6. Storage update

This block of code will update values in storage if `memVar_` is `1`, which we covered previously.

First, before the update, it calculates the new borrow rate from new utilization.

```solidity
// memVar_ => calculate new borrow rate for utilization. includes value overflow check.
memVar_ = LiquidityCalcs.calcBorrowRateFromUtilization(_rateData[token_], utilization_);
```

Fluid uses a classical utilization-based kink rate model. It supports 2 versions, so depending on the configured version, the curve contains either one or two kinks.

The code won't be analyzed here, simply to save on space and time, as it is not something unseen in DeFi. So this is left to readers unfamiliar with interest rate curves as an exercise.

Once we have borrow rate, before writing to storage we perform sanity checks on max possible values for exchange prices and utilization:

```solidity
// ensure values written to storage do not exceed the dedicated bit space in packed uint256 slots
if (o_.supplyExchangePrice > X64 || o_.borrowExchangePrice > X64) {
    revert FluidLiquidityError(ErrorTypes.UserModule__ValueOverflow__EXCHANGE_PRICES);
}
if (utilization_ > X14) {
    revert FluidLiquidityError(ErrorTypes.UserModule__ValueOverflow__UTILIZATION);
}
```

After that, we write to storage new values for:

```text
- borrow rate
- utilization
- update time
- supply exchange price
- borrow exchange price
- supplyRatio
- borrowRatio
```

by masking the right bits in the bitmap:

```solidity
o_.exchangePricesAndConfig =
    (o_.exchangePricesAndConfig &
        // mask to update bits: 0-15 (borrow rate), 30-43 (utilization), 58-248 (timestamp, exchange prices, ratios)
        0xfe000000000000000000000000000000000000000000000003fff0003fff0000) |
    memVar_ | // calcBorrowRateFromUtilization already includes an overflow check
    (utilization_ << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_UTILIZATION) |
    (block.timestamp << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_LAST_TIMESTAMP) |
    (o_.supplyExchangePrice << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_SUPPLY_EXCHANGE_PRICE) |
    (o_.borrowExchangePrice << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_BORROW_EXCHANGE_PRICE) |
    // ratios can never be > 100%, no overflow check needed
    (memVar3_ << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_SUPPLY_RATIO) | // supplyRatio (memVar3_ holds that value)
    (memVar4_ << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_BORROW_RATIO); // borrowRatio (memVar4_ holds that value)
// Updating on storage
_exchangePricesAndConfig[token_] = o_.exchangePricesAndConfig;
```

The only thing left is to answer what happens when `memVar_` stays at `0`.

Well, we don't update values in storage, but we still update exchange prices in the temp memory struct, which will later be used to emit an event.

```solidity
// do not update in storage but update o_.exchangePricesAndConfig for updated exchange prices at
// event emit of LogOperate
o_.exchangePricesAndConfig =
    (o_.exchangePricesAndConfig &
        // mask to update bits: 91-218 (exchange prices)
        0xfffffffffc00000000000000000000000000000007ffffffffffffffffffffff) |
    (o_.supplyExchangePrice << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_SUPPLY_EXCHANGE_PRICE) |
    (o_.borrowExchangePrice << LiquiditySlotsLink.BITS_EXCHANGE_PRICES_BORROW_EXCHANGE_PRICE);
```

So although storage update only happens if 1 day passes or value differences are significant enough, events triggered from the `operate` function will always have updated supply and exchange prices.