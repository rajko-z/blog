---
title: "Operate: Supply or withdraw"
weight: 11
layout: book
hiddenInHomeList: true
---

We are analyzing this piece of code:

{{< figure src="/fluid-vault/liquidity-layer/supply-or-withdraw.png" alt="Liquidity layer" width="100%" >}}

There are 2 parts. The first one calls the internal `_supplyOrWithdraw` method that contains most of the logic, and the second part just updates total supply amounts and performs sanity checks.

We can focus on the second part for now, as the first one will be analyzed on the next page.

There are 2 cases, which are labeled on the image above.

### 2.1 User supply with interest

```solidity
if (newSupplyInterestFree_ == 0) { ...
```

This case means that the user earns interest on its amount, so depending on the state of the action, whether it is a supply or withdrawal, global `supplyRawInterest` is updated. Note that the update still does not hit storage, but rather our temporary variables struct named `o_`.

```solidity
if (newSupplyInterestRaw_ > 0) {
    _checkMaxOperateAmountRatio(uint256(newSupplyInterestRaw_), o_.supplyRawInterest, true);
    o_.supplyRawInterest += uint256(newSupplyInterestRaw_);
} else {
    _checkMaxOperateAmountRatio(uint256(-newSupplyInterestRaw_), o_.supplyRawInterest, false);
    unchecked {
        o_.supplyRawInterest = o_.supplyRawInterest > uint256(-newSupplyInterestRaw_)
            ? o_.supplyRawInterest - uint256(-newSupplyInterestRaw_)
            : 0;
    }
}
```

You will also note the existence of the `_checkMaxOperateAmountRatio` method. Let's see how it looks:

{{< figure src="/fluid-vault/liquidity-layer/check-max-operate-amount-ratio.png" alt="Liquidity layer" width="100%" >}}

The main purpose of this method is to perform sanity checks on large amounts by using different constant values:

{{< figure src="/fluid-vault/liquidity-layer/check-max-operate-const.png" alt="Liquidity layer" width="100%" >}}

Both cases only work for amounts bigger than `2**80`, so this sanity check is avoided for lower amounts, even if the multiplier is greater:

1. We are doing supply or borrow  
   In this case, `newOperateAmount` can't go above `10000 * existing amount`.

2. We are doing withdraw or payback  
   In this case, `newOperateAmount` can't be above `50%` of existing amount.

This grouping is not performed based on liquidity direction, but rather on accounting direction, as supply/borrow increase amounts, while withdraw/payback decrease accounting amounts.

So this sanity check is here to prevent weird edge cases that could happen when working with larger amounts that differ too much from the existing accounting snapshot.

-----

Once we update global `supplyRawInterest`, we are arriving at this comment:

```solidity
// Note check for revert {UserModule}__ValueOverflow__TOTAL_SUPPLY is further down when we anyway
// calculate the normal amount from raw
```

We will see that branch `2.2` has an additional safety check for overflow, but here it is skipped as we are working with raw amounts (multiplied by exchange price). However, as noted, it will be performed later inside the codebase when we scale from raw amount back to normal.

We can cheat and check this right away. This is the code that will be shown later, so yes, it is checked for overflow:

```solidity
/// @dev limit any total amount to be half of type(uint128).max (~3.4e38) at type(int128).max (~1.7e38) as safety
/// measure for any potential overflows / unexpected outcomes. This is checked for total borrow / supply.
uint256 internal constant MAX_TOKEN_AMOUNT_CAP = uint256(uint128(type(int128).max));
uint256 internal constant EXCHANGE_PRICES_PRECISION = 1e12;

memVar3_ = ((o_.supplyRawInterest * o_.supplyExchangePrice) / EXCHANGE_PRICES_PRECISION);
if (memVar3_ > MAX_TOKEN_AMOUNT_CAP && supplyAmount_ > 0) {
    // only withdrawals allowed if total supply raw reaches MAX_TOKEN_AMOUNT_CAP
    revert FluidLiquidityError(ErrorTypes.UserModule__ValueOverflow__TOTAL_SUPPLY);
}
```

After this safety check comment, we convert `supplyRawInterest` to BigNumber representation and write it to the `totalAmounts` bitmap by occupying the first 64 bits:

```solidity
// Converting the updated total amount into big number for storage
memVar_ = o_.supplyRawInterest.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);
// update total supply with interest at total amounts in storage (only update changed values)
o_.totalAmounts =
    // mask to update bits 0-63
    (o_.totalAmounts & 0xffffffffffffffffffffffffffffffffffffffffffffffff0000000000000000) |
    memVar_; // converted to BigNumber can not overflow
```

### 2.2 User supply without interest

Analogously, in this case, we are working without interest, so we perform the same flow but on `supplyInterestFree`:

```solidity
// supply or withdraw interest free -> normal amount
if (newSupplyInterestFree_ > 0) {
    _checkMaxOperateAmountRatio(uint256(newSupplyInterestFree_), o_.supplyInterestFree, true);
    o_.supplyInterestFree += uint256(newSupplyInterestFree_);
} else {
    _checkMaxOperateAmountRatio(uint256(-newSupplyInterestFree_), o_.supplyInterestFree, false);
    unchecked {
        o_.supplyInterestFree = o_.supplyInterestFree > uint256(-newSupplyInterestFree_)
            ? o_.supplyInterestFree - uint256(-newSupplyInterestFree_)
            : 0;
    }
}
```

Before the bitmap mapping update, this part of the code contains an additional check which branch `2.1` skipped, as we are working with normal amounts here:

```solidity
if (o_.supplyInterestFree > MAX_TOKEN_AMOUNT_CAP) {
    // only withdrawals allowed if total supply interest free reaches MAX_TOKEN_AMOUNT_CAP
    revert FluidLiquidityError(ErrorTypes.UserModule__ValueOverflow__TOTAL_SUPPLY);
}
```

Again, after that we update the `totalAmounts` bitmap. This time we are making the update from 64 to 127 bits, as it will be the second number after `supplyInterestFree`:

```solidity
uint256 internal constant BITS_TOTAL_AMOUNTS_SUPPLY_INTEREST_FREE = 64;

// Converting the updated total amount into big number for storage
memVar_ = o_.supplyInterestFree.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);
// update total supply interest free at total amounts in storage (only update changed values)
o_.totalAmounts =
    // mask to update bits 64-127
    (o_.totalAmounts & 0xffffffffffffffffffffffffffffffff0000000000000000ffffffffffffffff) |
    (memVar_ << LiquiditySlotsLink.BITS_TOTAL_AMOUNTS_SUPPLY_INTEREST_FREE); // converted to BigNumber can not overflow
```

-------

Finally, there is another safety check:

```solidity
if (totalAmountsBefore_ == o_.totalAmounts) {
    // make sure that operate amount is not so small that it wouldn't affect storage update. if a difference
    // is present then rounding will be in the right direction to avoid any potential manipulation.
    revert FluidLiquidityError(ErrorTypes.UserModule__OperateAmountInsufficient);
}
```

which ensures that small dust operations that do not result in a `totalAmounts` change explicitly revert.

------

After this second part is analyzed, we are going to analyze the internal `_supplyOrWithdraw` method, but before that, lets introduce the concept of **`withdrawal limits`**.