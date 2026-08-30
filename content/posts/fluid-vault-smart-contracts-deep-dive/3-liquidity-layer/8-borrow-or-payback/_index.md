---
title: "Operate: Borrow or payback"
weight: 14
layout: book
hiddenInHomeList: true
---

We are analyzing this piece of code:

{{< figure src="/fluid-vault/liquidity-layer/borrow-or-payback.png" alt="Liquidity layer" width="100%" >}}

Just like with the supply or withdraw flow, there are 2 parts. The first one calls the internal `_borrowOrPayback` method that contains most of the logic, and the second part just updates total borrow amounts and performs sanity checks we already saw when we analyzed the supply or withdraw flow.

We will first analyze the second part, although faster this time, as the logic is similar to the one with the supply or withdraw flow, but we only work with borrow amounts instead. Then, on the next page, we will go over the first part and the `_borrowOrPayback` function.

### 2.1 User borrows with interest

```solidity
if (newBorrowInterestFree_ == 0) {
```

Similarly to supply, borrow can be with or without interest. This branch handles the case when the user borrows and pays interest.

Depending on whether it is a borrow or payback operation, it updates the local struct and calls the same `_checkMaxOperateAmountRatio` sanity check validation which we already covered in the supply or withdraw flow:

```solidity
if (newBorrowInterestRaw_ > 0) {
    _checkMaxOperateAmountRatio(uint256(newBorrowInterestRaw_), o_.borrowRawInterest, true);
    o_.borrowRawInterest += uint256(newBorrowInterestRaw_);
} else {
    _checkMaxOperateAmountRatio(uint256(-newBorrowInterestRaw_), o_.borrowRawInterest, false);
    unchecked {
        o_.borrowRawInterest = o_.borrowRawInterest > uint256(-newBorrowInterestRaw_)
            ? o_.borrowRawInterest - uint256(-newBorrowInterestRaw_)
            : 0; // payback amount is > total borrow -> payback total borrow down to 0
    }
}
```

After this, the new total borrow amount with interest is converted to BigNumber and written to the `totalAmounts` bitmap:

```solidity
// Converting the updated total amount into big number for storage
memVar_ = o_.borrowRawInterest.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_UP
);
// update total borrow with interest at total amounts in storage (only update changed values)
o_.totalAmounts =
    // mask to update bits 128-191
    (o_.totalAmounts & 0xffffffffffffffff0000000000000000ffffffffffffffffffffffffffffffff) |
    (memVar_ << LiquiditySlotsLink.BITS_TOTAL_AMOUNTS_BORROW_WITH_INTEREST); // converted to BigNumber can not overflow
```

### 2.2 User borrows without interest

Analogously, in this case, we are working without interest, so we perform the same flow but on `borrowInterestFree`:

```solidity
// borrow or payback interest free -> normal amount
if (newBorrowInterestFree_ > 0) {
    _checkMaxOperateAmountRatio(uint256(newBorrowInterestFree_), o_.borrowInterestFree, true);
    o_.borrowInterestFree += uint256(newBorrowInterestFree_);
} else {
    _checkMaxOperateAmountRatio(uint256(-newBorrowInterestFree_), o_.borrowInterestFree, false);
    unchecked {
        o_.borrowInterestFree = o_.borrowInterestFree > uint256(-newBorrowInterestFree_)
            ? o_.borrowInterestFree - uint256(-newBorrowInterestFree_)
            : 0; // payback amount is > total borrow -> payback total borrow down to 0
    }
}
```

Then we perform a sanity check on the max token amount cap:

```solidity
/// @dev limit any total amount to be half of type(uint128).max (~3.4e38) at type(int128).max (~1.7e38) as safety
/// measure for any potential overflows / unexpected outcomes. This is checked for total borrow / supply.
uint256 internal constant MAX_TOKEN_AMOUNT_CAP = uint256(uint128(type(int128).max));
if (o_.borrowInterestFree > MAX_TOKEN_AMOUNT_CAP) {
    // only payback allowed if total borrow interest free reaches MAX_TOKEN_AMOUNT_CAP
    revert FluidLiquidityError(ErrorTypes.UserModule__ValueOverflow__TOTAL_BORROW);
}
```

And then we write to the `totalAmounts` bitmap the same way we did before:

```solidity
// Converting the updated total amount into big number for storage
memVar_ = o_.borrowInterestFree.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_UP
);
// update total borrow interest free at total amounts in storage (only update changed values)
o_.totalAmounts =
    // mask to update bits 192-255
    (o_.totalAmounts & 0x0000000000000000ffffffffffffffffffffffffffffffffffffffffffffffff) |
    (memVar_ << LiquiditySlotsLink.BITS_TOTAL_AMOUNTS_BORROW_INTEREST_FREE); // converted to BigNumber can not overflow
```

-----

Finally, there is the same check as before with the supply or withdraw flow, which ensures that small dust operations that do not result in a `totalAmounts` change explicitly revert.

```solidity
if (totalAmountsBefore_ == o_.totalAmounts) {
    // make sure that operate amount is not so small that it wouldn't affect storage update. if a difference
    // is present then rounding will be in the right direction to avoid any potential manipulation.
    revert FluidLiquidityError(ErrorTypes.UserModule__OperateAmountInsufficient);
}
```

------

After this second part is analyzed (or better said, recalled from the supply/withdraw flow, as it was quite similar), we are going to analyze the internal `_borrowOrPayback_` method. But before that, we will work with limits again, this time for borrow.