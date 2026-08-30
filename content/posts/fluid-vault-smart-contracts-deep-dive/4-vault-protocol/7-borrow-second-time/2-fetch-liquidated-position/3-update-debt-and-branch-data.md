---
title: "Update debt and branch data"
weight: 33
layout: book
hiddenInHomeList: true
---

There is one small piece of code left to cover to finish this section of fetching a liquidated position.

We called the `fetchLatestPosition` function, and now the only part left is to update final debt and branch data:

{{< figure src="/fluid-vault/vault/liquidated-position.png" alt="Vault protocol" width="100%" >}}

First, we check if new raw debt is bigger than previously stored `dustDebtRaw`. This simply ensures that net debt is bigger than zero, because we will subtract dust from new raw debt. If that's not the case, we accumulate the difference and mark the user as fully liquidated:

```solidity
// Liquidated 100% or almost 100%
// absorbing dust debt
absorbedDustDebt = absorbedDustDebt + o_.dustDebtRaw - o_.debtRaw;
o_.debtRaw = 0;
o_.colRaw = 0;
```

We won't analyze this in depth, but `absorbedDustDebt` is a vault storage variable that accumulates dust and can later be pulled by admin by calling function:

```solidity
/// @notice absorbs accumulated dust debt
/// @dev in decades if a lot of positions are 100% liquidated (aka absorbed) then dust debt can mount up
/// which is basically sort of an extra revenue for the protocol.
//
// this function might never come in use that's why adding it in admin module
function absorbDustDebt(uint[] memory nftIds_) public _verifyCaller
```

If raw debt is bigger than dust, which will usually be the case, then we need to recalculate branch debt by subtracting newly calculated raw debt from old stored branch debt:

```solidity
// temp_ => branch's Debt
temp_ = (o_.branchData >> 52) & X64;
temp_ = (temp_ >> 8) << (temp_ & X8);

// o_.debtRaw should always be < branch's Debt (temp_).
// Taking margin (0.01%) in fetchLatestPosition to make sure it's always less
temp_ -= o_.debtRaw;
if (temp_ < 100) {
    // explicitly making sure that branch debt/liquidity doesn't get super low.
    temp_ = 100;
}
// Inserting updated branch's debt
branchData[temp2_] =
    (o_.branchData & 0xfffffffffffffffffffffffffffffffffff0000000000000000fffffffffffff) |
    (temp_.toBigNumber(56, 8, BigMathMinified.ROUND_UP) << 52);
```

So we are moving the user to a new tick, which means that his debt won't live in branch liquidated debt anymore, but rather in a new active tick.

Notice that we also ensure that branch debt can't go below `100`, which is an additional sanity check on top of taking `0.01%` from the user position, which we explained on the previous page. This also ensures we don't trigger this already mentioned case in the liquidation loop:

```solidity
if (currentData_.debt < 100) {
    // this can happen when someone tries to create a dust tick
    revert FluidVaultError(ErrorTypes.Vault__BranchDebtTooLow);
}
```

Once we updated branch debt, the only thing left to do is to calculate final user net debt by subtracting dust:

```solidity
unchecked {
    // Converted positionRawDebt_ in net position debt
    o_.debtRaw -= o_.dustDebtRaw;
}
```

------

This marks our section of fetching an already liquidated position. At this point, we covered how the vault `operate` function works for the most interesting parts, and the only thing left to do in this Fluid deep dive will be to analyze the main liquidation function, which will be the subject of the next pages.