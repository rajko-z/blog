---
title: "Summary"
weight: 20
layout: book
hiddenInHomeList: true
---

To summarize, we understood how the main `operate` function from the liquidity layer works under the hood.

We learned that Fluid keeps liquidity in one contract, and although we gave examples with test users like Bob and Alice, we know that users of this liquidity are actually other protocols that use it through one interface and the `operate` function.

We also saw how interest can be divided into with-interest and interest-free groups, and how withdrawal and borrow limits can dictate how protocols use liquidity from this layer.

As mentioned before, in the next big chapter, we will analyze one user of the Fluid liquidity layer called the Fluid Vault protocol.

This vault protocol will be the entry point for real users who will manage their positions using vaults, and all interactions with the liquidity layer will be hidden from the end user.

So in the next chapter, when we say the vault further calls the liquidity layer `operate` function, we will know what's going on.