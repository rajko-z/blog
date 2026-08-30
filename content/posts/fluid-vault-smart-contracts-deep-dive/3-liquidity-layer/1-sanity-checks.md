---
title: "Operate: Sanity checks"
weight: 4
layout: book
hiddenInHomeList: true
---

The operate function starts from performing few sanity checks. We are looking at this code:

{{< figure src="/fluid-vault/liquidity-layer/1-sanity-checks.png" alt="Liquidity layer" width="100%" >}}

Remember that this function handles all operations in one place:

- `supplyAmount > 0` -> we are doing a regular supply
- `supplyAmount < 0` -> we are withdrawing liquidity
- `borrowAmount > 0` -> we are doing a regular borrow
- `borrowAmount < 0` -> we are paying back borrowed assets

So we perform straightforward sanity checks in this order:

1. Making sure we are not supplying or borrowing zero amount of tokens

2. Making sure supply and borrow amounts are in the correct range of `type(int128)`.

   Although amounts are `int256` when calling the `operate` function, we are checking for the `type(int128)` range because it gives us flexibility later, knowing that we can fit two variables in 256 bits. For example, when calculating total liquidity passed as deposit + payback.

3. Making sure we are not withdrawing or borrowing to the zero address

4. Making sure we passed `msg.value` for the native token

   Like the majority of protocols, Fluid uses a standard convention to represent the native token address as:

   ```solidity
   /// @dev address that is mapped to the chain native token
   address internal constant NATIVE_TOKEN_ADDRESS = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
   ```