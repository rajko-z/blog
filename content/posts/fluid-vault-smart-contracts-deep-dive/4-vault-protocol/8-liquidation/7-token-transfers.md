---
title: "Liquidate: Token transfers"
weight: 42
layout: book
hiddenInHomeList: true
---

Once we have net token amounts, let's perform liquidity transfers:

{{< figure src="/fluid-vault/vault/liq-liquidity-transfers.png" alt="Vault protocol" width="80%" >}}

We first check if debt token is native token, in which case we refund back to liquidator any tokens not spent on liquidation:

```solidity
SafeTransfer.safeTransferNative(msg.sender, msg.value - actualDebtAmt_);
```

After that, we perform the payback to liquidity layer, where debt tokens will be pulled from liquidator:

```solidity
// payback at liquidity
LIQUIDITY.operate{ value: temp_ }(
    BORROW_TOKEN,
    0,
    -int(actualDebtAmt_),
    address(0),
    address(0),
    abi.encode(msg.sender)
);
```

And finally, we withdraw collateral tokens back to liquidator, again using liquidity layer:

```solidity
// withdraw at liquidity
LIQUIDITY.operate(SUPPLY_TOKEN, -int(actualColAmt_), 0, to_, address(0), new bytes(0));
```