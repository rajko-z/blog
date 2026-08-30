---
title: "LiquidityCalcs: calcExchangePrices"
weight: 10
layout: book
hiddenInHomeList: true
---

This is the function we will be analyzing:

{{< figure src="/fluid-vault/liquidity-layer/calc-exchange-prices.png" alt="Liquidity layer" width="100%" >}}

This function is part of the `LiquidityCalcs` library, which can be found in the libraries folder:

{{< figure src="/fluid-vault/liquidity-layer/lib.png" alt="Liquidity layer" width="40%" >}}

Every DeFi lending protocol has to track interest paid by borrowers and yield gained by suppliers. Usually, these are handled by separate indexes which are multiplied by principal, and that is what this function does with exchange prices.

But besides that, Fluid introduces one important add-on, which is logic for explicitly tracking who pays interest and who does not.

This means that not everyone earns supply yield and not everyone pays borrow interest. We can divide that into 2 categories:

```text
Supply Side
- supplyWithInterest
- supplyInterestFree
```

```text
Borrow Side
- borrowWithInterest
- borrowInterestFree
```

Knowing this, we can start analyzing the function.

We can see that the function can be divided into a few logical pieces:

```
1. Extraction of exchange prices
2. Calculation of borrow exchange price
3. Calculation of supply exchange price
    3.1. Case when supply with interest is 0
    3.2. Calculation of `utilization * supplyRatio`
        3.2.1. Case when `supplyInterestFree > supplyWithInterest`
        3.2.2. Case when `supplyWithInterest > supplyInterestFree`
    3.3. Calculation of `borrowRatio`
        3.3.1. Case when `borrowInterestFree > borrowWithInterest`
        3.3.2. Case when `borrowWithInterest > borrowInterestFree`
    3.4. Final calculation of supply exchange price
```

So we will start from extraction.

-----

## 1. Extract exchange prices and borrow rate from bitmap

First, we extract current exchange prices from storage and perform a sanity check to revert on zero prices:

```solidity
// Extracting exchange prices
supplyExchangePrice_ =
    (exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_SUPPLY_EXCHANGE_PRICE) &
    X64;
borrowExchangePrice_ =
    (exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_BORROW_EXCHANGE_PRICE) &
    X64;

if (supplyExchangePrice_ == 0 || borrowExchangePrice_ == 0) {
    revert FluidLiquidityCalcsError(ErrorTypes.LiquidityCalcs__ExchangePriceZero);
}
```

To extract them, we perform common bitwise operations. In this case, we need the correct position in the `exchangePricesAndConfig` bitmap:

```solidity
uint256 internal constant BITS_EXCHANGE_PRICES_SUPPLY_EXCHANGE_PRICE = 91;
uint256 internal constant BITS_EXCHANGE_PRICES_BORROW_EXCHANGE_PRICE = 155;
```

This matches the comments on top of the `exchangePricesAndConfig` bitmap:

```solidity
/// Next  64 bits =>  91-154 => supply exchange price (1e12 -> max value 18_446_744,073709551615)
/// Next  64 bits => 155-218 => borrow exchange price (1e12 -> max value 18_446_744,073709551615)
```

And after the right shift, we simply perform a bitwise `and` with X64 (`0xffffffffffffffff`), which just isolates the first 64 bits, which is the value we are looking for.

After that, the same way, we extract the borrow rate:

```solidity
uint256 temp_ = exchangePricesAndConfig_ & X16; // temp_ = borrowRate
```

Note that here we don't need to shift, as the borrow rate is stored in the first 16 bits:

```solidity
/// First 16 bits =>   0- 15 => borrow rate (in 1e2: 100% = 10_000; 1% = 100 -> max value 65535)
```
------

## 2. Calculate borrow exchange price

This part of calculating the borrow exchange price is standard. We want to calculate how much time has passed, and linearly apply it to `borrowExchangePrice`.

So if the borrow rate is `10%` annually, after half a year has passed, we want to increase `borrowExchangePrice` by `5%`.

So we first calculate the time passed from the last snapshot, by using the same trick for extracting data from the bitmap:

```solidity
/// What we know from bitmap
/// 33 bits =>  58- 90 => last update timestamp (enough until 16 March 2242 -> max value 8589934591)

uint256 internal constant BITS_EXCHANGE_PRICES_LAST_TIMESTAMP = 58;
uint256 internal constant X33 = 0x1ffffffff;

uint256 secondsSinceLastUpdate_ = block.timestamp -
    ((exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_LAST_TIMESTAMP) & X33);
```

Then we extract `borrowRatio` as well:

```solidity
/// What we know from bitmap
/// 1 bit  => 234-234 => if 0 then ratio is borrowInterestFree / borrowWithInterest else ratio is borrowWithInterest / borrowInterestFree
/// 14 bits => 235-248 => borrowRatio: borrowInterestFree / borrowWithInterest (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383)

uint256 internal constant BITS_EXCHANGE_PRICES_BORROW_RATIO = 234;
uint256 internal constant X15 = 0x7fff;

uint256 borrowRatio_ = (exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_BORROW_RATIO) &
    X15;
```
    
**Wait, what is borrowRatio?**

Remember that we want to support both interest-free borrows and regular ones, so this just tells us the ratio between these two.

The first bit determines how we should look at this value.

For `1`, it is `borrowWithInterest / borrowInterestFree`.

For `0`, it is `borrowInterestFree / borrowWithInterest`.

Fluid makes sure that this denominator is always less than 100%, meaning:

For `1`, `borrowInterestFree` is greater than `borrowWithInterest`.

For `0`, `borrowWithInterest` is greater than `borrowInterestFree`.

Same will apply later to supply side.

Why we need this now can be explained by the next check:

```solidity
if (secondsSinceLastUpdate_ == 0 || temp_ == 0 || borrowRatio_ == 1) {
    // if no time passed, borrow rate is 0, or no raw borrowings: no exchange price update needed
    // (if borrowRatio_ == 1 means there is only borrowInterestFree, as first bit is 1 and rest is 0)
    return (supplyExchangePrice_, borrowExchangePrice_);
}
```

We want to return early with unchanged exchange prices if one of these is true:

1. No time has passed since the last update (e.g. if prices are already accumulated in the same block): `secondsSinceLastUpdate_ == 0`
2. Borrow rate is 0: `temp_ == 0`
3. There are no borrowers paying interest at the moment. This can be checked by the borrow ratio, as `1` represents `[00000000000000][1]`, which tells us that `borrowWithInterest / borrowInterestFree` is `0`. Meaning `borrowWithInterest` is `0`, and no borrower is paying interest.

If nothing above is true, we simply calculate the linear interest gain for the current `borrowExchangePrice_`:

```solidity
// calculate new borrow exchange price.
// formula borrowExchangePriceIncrease: previous price * borrow rate * secondsSinceLastUpdate_.
// nominator is max uint112 (uint64 * uint16 * uint32). Divisor can not be 0.
borrowExchangePrice_ +=
    (borrowExchangePrice_ * temp_ * secondsSinceLastUpdate_) /
    (SECONDS_PER_YEAR * FOUR_DECIMALS);
```

Because the borrow rate is represented in 1e2 (`100% = 1e4`), we divide by `1e4`.


---------------

## 3. Calculate supply exchange price

This part is harder, as we can't simply do something you would see in other protocols:

```solidity
supplyRate ≈ borrowRate × utilization × (1 - reserveFactor)
```

This would usually work, as you calculate how much supplied assets are actually borrowed, you assume all borrowers pay interest, you take the protocol fee cut, and that's it.

Here, that does not apply, as again:

1. We can't simply use `borrowRate`, as not all borrowers pay interest on utilized assets
2. Not all borrow interest goes to all suppliers, only to the `supplyWithInterest` group

So how can we calculate this? Let's take a concrete example from Fluid codebase comments and expand it further.

----

#### Example

`supplyWithInterest = 80`  
`supplyInterestFree = 20`  
`totalSupply = 100`  
`borrowWithInterest = 50`  
`borrowInterestFree = 10`  
`totalBorrow = 60`  
`borrow rate = 40%`  
`fee = 10%`  
`borrowRatio = 10 / 50 = 20%` (because `borrowWithInterest` > `borrowInterestFree`)  
`supplyRatio = 20 / 80 = 25%` (because `supplyWithInterest` > `supplyInterestFree`)  

Half a year passes.

For the **borrow side**:

- Only `borrowWithInterest` pays interest -> `50 * 40% * 0.5 = 10`. This means that `borrowWithInterest` grows from `50 -> 60`
- Borrow exchange price goes from `1 -> 1.2`
- `borrowInterestFree` stays the same at `10 -> 10`
- And `totalBorrow` grows from `60 -> 70`

For the **supply side**:

We know that the total yield paid in this half a year is `10`.

Protocol takes a cut of `10%`, meaning `9` is left for suppliers.

But not all suppliers, only the `supplyWithInterest` group. This means that the supply return rate is: `9 / 80 = 11.25%`.

So from this we know that:

- `supplyWithInterest` grows from `80 -> 89`
- Supply exchange price goes from `1 -> 1.1125`
- `supplyInterestFree` stays the same at `20 -> 20`
- `totalSupply` goes from `100 -> 109`

Getting the **formula**:

To get to the previously manually calculated `11.25%` supply rate from `borrowRate`, Fluid introduced `ratioSupplyYield`.

This can be seen in comments:

```solidity
// FOR SUPPLY EXCHANGE PRICE:
// all yield paid by borrowers (in mode with interest) goes to suppliers in mode with interest.
// formula: previous price * supply rate * secondsSinceLastUpdate_.
// where supply rate = (borrow rate  - revenueFee%) * ratioSupplyYield. And
// ratioSupplyYield = utilization * fractionOfSuppliersThatEarnsInterest * fractionOfBorrowThatPaysInterest
```

Let's prove it on the example above.

How much supply is actually utilized?

```solidity
utilization = totalBorrow / totalSupply = 60 / 100 = 60%
```

What is the fraction of borrowed amount that actually pays interest?

```solidity
fractionOfBorrowThatPaysInterest = borrowWithInterest / totalBorrow = 50 / 60 = 83.3333%
```

What is the fraction of suppliers that get all the yield?

```solidity
fractionOfSuppliersThatEarnsInterest = totalSupply / supplyWithInterest = 100 / 80 = 125%
```

Why inverse? Because fewer suppliers get all the yield. The fewer suppliers with interest, the more yield per token they get, because all the yield goes to a smaller group. It is not distributed equally.

Putting these numbers in the final formula:

```solidity
ratioSupplyYield = utilization * fractionOfSuppliersThatEarnsInterest * fractionOfBorrowThatPaysInterest
                 = 60% * 83.3333% * 125% = 62.5%

supplyRate       = (borrow rate - revenueFee%) * ratioSupplyYield
                 = (40% - 10% * 40%) * 62.5% = 36% * 62.5% = 22.5%
```

And finally, we divide this by 2, as we calculated half-year yield before:

```solidity
finalSupplyRate  = supplyRate * timePassed / year
                 = 22.5% * 0.5 = 11.25%
```

We can see that this is exactly the value we calculated manually before.

----

Now that we got an idea with the example above, let's quickly finish up the implementation details from the `calcExchangePrices` function.

### 3.1 Case when supply with interest is zero

We first handle the case when there is no `supplyWithInterest`. In that case, we simply return the current unchanged `supplyExchangePrice`, as there is no additional yield gained.

Similarly to the borrow ratio, we extract `supplyRatio` and check if it is `1`.

`1` means that `supplyWithInterest / supplyInterestFree = 0`, or more precisely, `supplyWithInterest = 0`, meaning there are no supplied tokens earning yield.

```solidity
// temp_ => supplyRatio (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383)
// if first bit 0 then ratio is supplyInterestFree / supplyWithInterest (supplyWithInterest is bigger)
// else ratio is supplyWithInterest / supplyInterestFree (supplyInterestFree is bigger)
temp_ = (exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_SUPPLY_RATIO) & X15;

if (temp_ == 1) {
    // if no raw supply: no exchange price update needed
    // (if supplyRatio_ == 1 means there is only supplyInterestFree, as first bit is 1 and rest is 0)
    return (supplyExchangePrice_, borrowExchangePrice_);
}
```

If there is some `supplyWithInterest` we start calculating `ratioSupplyYield`.  
Fluid does this in 2 steps:
- calculate `utilization * supplyRatio`
- then calculate `borrowRatio`

----

### 3.2 Calculate utilization * supplyRatio

This part has branching, depending if there is more `supplyInterestFree` or `supplyWithInterest`.

#### 3.2.1 First branch (supplyInterestFree > supplyWithInterest):

```solidity
// ratioSupplyYield precision is 1e27 as 100% for increased precision when supplyInterestFree > supplyWithInterest
if (temp_ & 1 == 1) {
    // ratio is supplyWithInterest / supplyInterestFree (supplyInterestFree is bigger)
    temp_ = temp_ >> 1;
    temp_ = (1e27 * FOUR_DECIMALS) / temp_;
    // utilization * (100% + 100% / supplyRatio)
    temp_ =
        (((exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_UTILIZATION) & X14) *
            (1e27 + temp_)) / (FOUR_DECIMALS);
}
```

The part of getting utilization is straightforward. We are just extracting it from the stored value in the bitmap, nothing new there.

The rest can look confusing at first, but it just represents the formulas we already described before.

We know from before that:

```text
fractionOfSuppliersThatEarnsInterest = totalSupply / supplyWithInterest
```

This can also be written as:

```text
(supplyInterestFree + supplyWithInterest) / supplyWithInterest
```

or:

```text
1 + supplyInterestFree / supplyWithInterest
```

If we expand calculation in Fluid code:

```text
utilization * (100% + 100% / supplyRatio)
```

```text
= utilization * (100% + 100% / (supplyWithInterest / supplyInterestFree))
```

```text
= utilization * (100% + 100% * supplyInterestFree / supplyWithInterest)
```

```text
= 100% * utilization * (1 + supplyInterestFree / supplyWithInterest)
```

We see this is the same scaled value we got before.

#### 3.2.2 Second branch (supplyWithInterest > supplyInterestFree):

```solidity
} else {
    // ratio is supplyInterestFree / supplyWithInterest (supplyWithInterest is bigger)
    temp_ = temp_ >> 1;
    temp_ =
        // 1e27 * utilization * (100% + supplyRatio) / 100%
        (1e27 *
            ((exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_UTILIZATION) & X14) *
            (FOUR_DECIMALS + temp_)) /
        (FOUR_DECIMALS * FOUR_DECIMALS);
}
```

This part is completely analogous to the previous branch. The only difference is that we don't need to reverse the stored `supplyRatio`, as it already comes in the form:

```text
supplyInterestFree / supplyWithInterest
```

-------

### 3.3 Calculate borrow ratio

We know from before that `fractionOfBorrowThatPaysInterest` can be calculated as:

```text
borrowWithInterest / totalBorrow
```

Which can be written as:

```text
borrowWithInterest / (borrowWithInterest + borrowInterestFree)
```

So jumping back to code, just like for `supply`, we have 2 cases.

#### 3.3.1 borrowInterestFree > borrowWithInterest

```solidity
if (borrowRatio_ & 1 == 1) {
    // ratio is borrowWithInterest / borrowInterestFree (borrowInterestFree is bigger)
    borrowRatio_ = borrowRatio_ >> 1;
    borrowRatio_ = (borrowRatio_ * 1e27) / (FOUR_DECIMALS + borrowRatio_);
}
```

In this case `borrowRatio` is stored as:

```text
borrowWithInterest / borrowInterestFree
```

We need:

```text
borrowWithInterest / (borrowWithInterest + borrowInterestFree)
```

So we divide numerator and denominator by `borrowInterestFree` and get:

```text
(borrowWithInterest / borrowInterestFree)
-------------------------------------------------
1 + (borrowWithInterest / borrowInterestFree)
```

Which becomes:

```text
borrowRatio
-----------------
1 + borrowRatio
```

Which is the same thing we see in code with additional scaling.

#### 3.3.2 borrowWithInterest > borrowInterestFree

```solidity
// ratio is borrowInterestFree / borrowWithInterest (borrowWithInterest is bigger)
borrowRatio_ = borrowRatio_ >> 1;
borrowRatio_ = (1e27 - ((borrowRatio_ * 1e27) / (FOUR_DECIMALS + borrowRatio_)));
```

In this case `borrowRatio` is stored as:

```text
borrowInterestFree / borrowWithInterest
```

We need again:

```text
borrowWithInterest / (borrowWithInterest + borrowInterestFree)
```

which can be written as:

```text
1 - borrowInterestFree / (borrowWithInterest + borrowInterestFree)
```

If we take this:

```text
borrowInterestFree / (borrowWithInterest + borrowInterestFree)
```

and divide numerator and denominator by `borrowWithInterest`, we get:

```text
(borrowInterestFree / borrowWithInterest)
-------------------------------------------------
1 + (borrowInterestFree / borrowWithInterest)
```

Which becomes:

```text
borrowRatio
-----------------
1 + borrowRatio
```

Taking this back to the final formula, we get:

```text
        borrowRatio
1 - -----------------
      1 + borrowRatio
```

Which is exactly the thing we see in Solidity code.

-----

### 3.4 Calculating final supply exchange price

The final part is just gluing everything together.

This is the code we are analyzing:

```solidity
// temp_ => ratioSupplyYield. scaled down from 1e25 = 1% each to normal percent precision 1e2 = 1%.
// max nominator value is ~1.64e31 * 1e27 = 1.64e58. max result = 1.64e8
temp_ = (FOUR_DECIMALS * temp_ * borrowRatio_) / 1e54;

// 2. calculate supply rate
// temp_ => supply rate (borrow rate  - revenueFee%) * ratioSupplyYield.
// division part is done in next step to increase precision. (divided by 2x FOUR_DECIMALS, fee + borrowRate)
// Note that all calculation divisions for supplyExchangePrice are rounded down.
// Note supply rate can be bigger than the borrowRate, e.g. if there are only few lenders with interest
// but more suppliers not earning interest.
temp_ = ((exchangePricesAndConfig_ & X16) * // borrow rate
    temp_ * // ratioSupplyYield
    (FOUR_DECIMALS - ((exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_FEE) & X14))); // revenueFee
// fee can not be > 100%. max possible = 65535 * ~1.64e8 * 1e4 =~1.074774e17.

// 3. calculate increase in supply exchange price
supplyExchangePrice_ += ((supplyExchangePrice_ * temp_ * secondsSinceLastUpdate_) /
    (SECONDS_PER_YEAR * FOUR_DECIMALS * FOUR_DECIMALS * FOUR_DECIMALS));
// max possible nominator = max uint 64 * 1.074774e17 * max uint32 = ~8.52e45. Denominator can not be 0.
```

We start by multiplying `temp_` with `borrowRatio_`, which is the multiplication of steps `3.2` and `3.3`, to get the final `ratioSupplyYield`.

Note that when we say `supplyRatio`, we really mean `fractionOfSuppliersThatEarnsInterest`.

And for `borrowRatio`, we mean `fractionOfBorrowThatPaysInterest`.

```text
ratioSupplyYield =
    utilization
    * fractionOfSuppliersThatEarnsInterest
    * fractionOfBorrowThatPaysInterest
```

After that, we calculate the supply rate as:

```text
supplyRate =
    (borrowRate - revenueFee%)
    * ratioSupplyYield
```

`revenueFee` is read from the bitmap as:

```solidity
(exchangePricesAndConfig_ >> LiquiditySlotsLink.BITS_EXCHANGE_PRICES_FEE) & X14
```

so nothing new there.

And finally, once we have `supplyRate`, we perform linear scaling depending on time passed, just like we did for `borrowRate`, to get the final form of `supplyExchangePrice`:

```solidity
supplyExchangePrice_ += ((supplyExchangePrice_ * temp_ * secondsSinceLastUpdate_) /
    (SECONDS_PER_YEAR * FOUR_DECIMALS * FOUR_DECIMALS * FOUR_DECIMALS));
```

---------
---------

Congratulations! We just understood how Fluid works with exchange prices.

In the next section, we will tackle what actually happens on supply or withdrawal, and will introduce one of the nice innovations Fluid brings, which are limits.