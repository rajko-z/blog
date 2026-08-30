---
title: "2nd supply/withdraw no debt"
weight: 24
layout: book
hiddenInHomeList: true
---

Everything stays the same, the only difference is that now we are working with existing NFT, meaning we are not passing `0` to main `operate` function.  

If we look at the implementation, we see that we only unlock this new code below in second if case during position fetching:

{{< figure src="/fluid-vault/vault/nft-exists.png" alt="Vault protocol" width="100%" >}}


Code:

1. Checks nft ownership if we are doing withdraw or borrow as those operations affect health factor
2. Checks if there exists valid non empty position for passed NFT. This ensures that NFT is from this vault we are interacting with
3. Fetch and convert supply amount and store it in temp struct for memory variables
4. Fetch and convert debt amount and store it in temp struct for memory variables
5. Assign new tick. Note because this is supply only position, tick get's default value of `type(int).min` just like on first supply
