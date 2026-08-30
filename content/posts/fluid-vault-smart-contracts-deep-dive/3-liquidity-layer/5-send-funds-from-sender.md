---
title: "Operate: Send funds from sender"
weight: 8
layout: book
hiddenInHomeList: true
---

After the ETH payment check, we want to send funds from the caller to this liquidity layer.

This is the snippet of code we are looking at:

{{< figure src="/fluid-vault/liquidity-layer/5-send-funds-from-sender.png" alt="Liquidity layer" width="100%" >}}

At first, we see that we want to transfer tokens only if `memVar2_` is greater than zero, meaning `deposit + payback != 0`.

For ETH transfers, this is already handled in the previous section by setting it to zero, which is also noted by the comment, so no new transfer is needed.

Stepping into the function, callback length will again be skipped, as we proved in the DEX callback section that callback data will have at most 32 bytes for T1 vaults.

After that, we:

- Take a snapshot of current liquidity
- Call back the caller to transfer the funds
- Calculate the funds transferred by taking the post-callback snapshot
- Check if enough funds were sent and revert if needed

For this line:

```solidity
IProtocol(msg.sender).liquidityCallback(token_, memVar2_, callbackData_);
```

we will cheat again and look at the vault implementation:

{{< figure src="/fluid-vault/liquidity-layer/liquidity-callback.png" alt="Liquidity layer" width="100%" >}}

The function performs a few security checks: whether the liquidity contract called us and whether we are in a reentrancy state. This part of bitwise operations on vault variables will be explained later: `vaultVariables & 1 == 0`, but intuitively, this just checks whether someone already entered this vault before, in which case we want to revert.

After the checks, we decode the callback data which we previously encoded while calling the liquidity contract, and send the amount (`deposit + payback`) back to the liquidity contract.

Once we are back at the liquidity layer, we simply perform the same sanity check like we did for ETH transfer: whether we got enough tokens and whether the user sent us more than 1% extra tokens, in which case we want to revert. As stated in the comment, we are allowing 1% tolerance for rounding issues.

```solidity
if (memVar_ < memVar2_ || memVar_ > (memVar2_ * (FOUR_DECIMALS + memVar3_)) / FOUR_DECIMALS) {
    // revert if protocol did not send enough to cover supply / payback
    // or if protocol sent more than expected, with 1% tolerance for any potential rounding issues (and for DEX revenue cut)
    revert FluidLiquidityError(ErrorTypes.UserModule__TransferAmountOutOfBounds);
}
```

At the end, we call:

```solidity
_afterTransferIn(token_, memVar_);
```

which is only used for custom EVM chains as additional hook logic, with the default empty implementation:

```solidity
/// @notice Hook invoked after assets are transferred into the contract. Must be implemented for chain specific implementation version.
/// @dev ATTENTION: THIS IS NOT CALLED FOR NATIVE TOKEN (e.g. ETH).
/// @param token_ The address of the transferred-in token.
/// @param amount_ The amount of tokens transferred in.
function _afterTransferIn(address token_, uint256 amount_) internal virtual;
```

For curious readers, here is how one such hook looks for Zircuit, an EVM-compatible zero-knowledge (ZK) rollup built on the Optimism Bedrock stack:

{{< figure src="/fluid-vault/liquidity-layer/zircuit.png" alt="Liquidity layer" width="100%" >}}