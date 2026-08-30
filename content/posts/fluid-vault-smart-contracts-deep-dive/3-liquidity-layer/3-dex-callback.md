---
title: "Operate: Dex callback"
weight: 6
layout: book
hiddenInHomeList: true
---

After setup of memory variables, we are looking at this piece of code, which we are going to skip and won't analyze.

{{< figure src="/fluid-vault/liquidity-layer/3-dex-callback.png" alt="Liquidity layer" width="100%" >}}

Why?

Because we are "only" analyzing T1 vaults in this guide, and this piece of code is only used for DEX vaults (T2, T3, T4).

To prove that to you, we are going to cheat a little bit and jump to the place in the code where T1 vaults (consumers of this liquidity) use this `operate` function. Note that T1 vaults will be analyzed later in this guide in separate `Vault Protocol` section.

Snippet of code from `contracts/protocols/vault/vaultT1/coreModule/main.sol`:

{{< figure src="/fluid-vault/liquidity-layer/vault-callback.png" alt="Liquidity layer" width="70%" >}}

We can see that only these 4 places use this `LIQUIDITY.operate` function, and as callback data, they pass either empty bytes or `abi.encode(address)`.

ABI encoding of any address will produce 32 bytes length. We can double-check that with `chisel`:

{{< figure src="/fluid-vault/liquidity-layer/chisel-abi-encode-length.png" alt="Liquidity layer" width="80%" >}}

All of these usages won't satisfy the outer `if` check, which checks if the callback data size is greater than `63` bytes:

```solidity
if (callbackData_.length > 63) ...
```
