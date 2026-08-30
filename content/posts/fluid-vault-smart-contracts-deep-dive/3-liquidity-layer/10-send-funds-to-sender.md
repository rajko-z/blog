---
title: "Operate: Send funds to sender"
weight: 18
layout: book
hiddenInHomeList: true
---

We are almost finished with this main `operate` function. As we finished all calculations, now we are only left to send funds to the user if he is using withdraw or borrow operation.

This is the code we are analyzing.

{{< figure src="/fluid-vault/liquidity-layer/send-funds-to-sender.png" alt="Liquidity layer" width="100%" >}}

First, this whole branch of using `o_.netTransfersOut` can be skipped, as it is only used for DEXes, leaving us only the second branch to analyze.

The code is straightforward. We first store borrow and withdraw amounts to memory variables:

```solidity
memVar2_ = borrowAmount_ > 0 ? uint256(borrowAmount_) : 0;
if (supplyAmount_ < 0) {
    unchecked {
        memVar_ = uint256(-supplyAmount_);
    }
} else {
    memVar_ = 0;
}
```

After this, we perform a small optimization if the user is using the same address to receive funds for withdraw and borrow, which means we can aggregate it in one transfer:

```solidity
if (memVar_ > 0 && memVar2_ > 0 && withdrawTo_ == borrowTo_) {
    // if user is doing borrow & withdraw together and address for both is the same
    // then transfer tokens of borrow & withdraw together to save on gas
    if (token_ == NATIVE_TOKEN_ADDRESS) {
        SafeTransfer.safeTransferNative(withdrawTo_, memVar_ + memVar2_);
    } else {
        memVar_ = memVar_ + memVar2_;
        _preTransferOut(token_, memVar_);
        SafeTransfer.safeTransfer(token_, withdrawTo_, memVar_);
    }
}
```

Otherwise, if the amount is non-zero, we transfer tokens separately to different addresses.

If the token is ETH:

```solidity
if (token_ == NATIVE_TOKEN_ADDRESS) {
    // if withdraw
    if (memVar_ > 0) {
        SafeTransfer.safeTransferNative(withdrawTo_, memVar_);
    }
    // if borrow
    if (memVar2_ > 0) {
        SafeTransfer.safeTransferNative(borrowTo_, memVar2_);
    }
}
```

Or if the token is a regular ERC20 token:

```solidity
// if withdraw
if (memVar_ > 0) {
    _preTransferOut(token_, memVar_);
    SafeTransfer.safeTransfer(token_, withdrawTo_, memVar_);
}
// if borrow
if (memVar2_ > 0) {
    _preTransferOut(token_, memVar2_);
    SafeTransfer.safeTransfer(token_, borrowTo_, memVar2_);
}
```

Note that `_preTransferOut` is only used for specific chain implementations, which is not the focus of this analysis.