---
title: "Borrow or payback"
weight: 15
layout: book
hiddenInHomeList: true
---

This is the complete body of the function we are analyzing.

{{< figure src="/fluid-vault/liquidity-layer/borrow-or-payback-full.png" alt="Liquidity layer" width="100%" >}}

Construction is similar to the supply or withdraw flow, but the code is simplified because we don't have decay amounts.

We first perform validation:

```solidity
uint256 userBorrowData_ = _userBorrowData[msg.sender][token_];

if (userBorrowData_ == 0) {
    revert FluidLiquidityError(ErrorTypes.UserModule__UserNotDefined);
}
if ((userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_IS_PAUSED) & 1 == 1) {
    revert FluidLiquidityError(ErrorTypes.UserModule__UserPaused);
}
```

1. User presence check. We revert if the user is not defined
2. If user borrow is paused

Then we extract the borrow amount and calculate the current borrow limit by calling the function we explained on the previous page:

```solidity
// extract user borrow amount
uint256 userBorrow_ = (userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_AMOUNT) & X64;
userBorrow_ = (userBorrow_ >> DEFAULT_EXPONENT_SIZE) << (userBorrow_ & DEFAULT_EXPONENT_MASK);

// calculate current, updated (expanded etc.) borrow limit
uint256 newBorrowLimit_ = LiquidityCalcs.calcBorrowLimitBeforeOperate(userBorrowData_, userBorrow_);
```

Then we update the user borrow amount with the new operate amount, depending on whether it is the interest or interest-free case, and whether we are doing borrow or repay. Nothing unseen here:

```solidity
if (userBorrowData_ & 1 == 1) {
    // with interest
    if (amount_ > 0) {
        // convert amount normal to raw (divide by exchange price) -> round up for borrow
        newBorrowInterestRaw_ = int256(
            FixedPointMathLib.mulDivUp(uint256(amount_), EXCHANGE_PRICES_PRECISION, borrowExchangePrice_)
        );
        userBorrow_ = userBorrow_ + uint256(newBorrowInterestRaw_);
    } else {
        // convert amount from normal to raw (divide by exchange price) -> round down for payback
        newBorrowInterestRaw_ = (amount_ * int256(EXCHANGE_PRICES_PRECISION)) / int256(borrowExchangePrice_);
        userBorrow_ = userBorrow_ - uint256(-newBorrowInterestRaw_);
    }
} else {
    // without interest
    newBorrowInterestFree_ = amount_;
    if (newBorrowInterestFree_ > 0) {
        // borrowing
        userBorrow_ = userBorrow_ + uint256(newBorrowInterestFree_);
    } else {
        // payback
        userBorrow_ = userBorrow_ - uint256(-newBorrowInterestFree_);
    }
}
```

Then we perform the borrow limit check and revert if the new borrow exceeds the limit:

```solidity
if (amount_ > 0 && userBorrow_ > newBorrowLimit_) {
    // if borrow, then check the user borrow amount after borrowing is below borrow limit
    revert FluidLiquidityError(ErrorTypes.UserModule__BorrowLimitReached);
}
```

Then we calculate the final borrow limit that will be stored:

```solidity
// calculate borrow limit to store as previous borrow limit in storage
newBorrowLimit_ = LiquidityCalcs.calcBorrowLimitAfterOperate(userBorrowData_, userBorrow_, newBorrowLimit_);
```

And finally, we convert values to BigNumbers and write them to the user borrow data bitmap:

```solidity
// Converting user's borrowings into bignumber
userBorrow_ = userBorrow_.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_UP
);

if (((userBorrowData_ >> LiquiditySlotsLink.BITS_USER_BORROW_AMOUNT) & X64) == userBorrow_) {
    // make sure that operate amount is not so small that it wouldn't affect storage update. if a difference
    // is present then rounding will be in the right direction to avoid any potential manipulation.
    revert FluidLiquidityError(ErrorTypes.UserModule__OperateAmountInsufficient);
}

// Converting borrow limit into bignumber
newBorrowLimit_ = newBorrowLimit_.toBigNumber(
    DEFAULT_COEFFICIENT_SIZE,
    DEFAULT_EXPONENT_SIZE,
    BigMathMinified.ROUND_DOWN
);

// Updating on storage
_userBorrowData[msg.sender][token_] =
    // mask to update bits 1-161 (borrow amount, borrow limit, timestamp)
    (userBorrowData_ & 0xfffffffffffffffffffffffc0000000000000000000000000000000000000001) |
    (userBorrow_ << LiquiditySlotsLink.BITS_USER_BORROW_AMOUNT) | // converted to BigNumber can not overflow
    (newBorrowLimit_ << LiquiditySlotsLink.BITS_USER_BORROW_PREVIOUS_BORROW_LIMIT) | // converted to BigNumber can not overflow
    (block.timestamp << LiquiditySlotsLink.BITS_USER_BORROW_LAST_UPDATE_TIMESTAMP);
```

Notice that there is again a check if the borrow amount actually changed because of lower amounts that can round down to the same old borrow value, so that we don't write the same value that is already present in storage.

---

This finishes explaining the borrow or repay flow, and gets us closer to the end of the main `operate` function.

The only bigger part left is understanding what happens to utilization, supply and borrow ratios, and exchange prices after the operate amount, which we are going to tackle in the following pages.