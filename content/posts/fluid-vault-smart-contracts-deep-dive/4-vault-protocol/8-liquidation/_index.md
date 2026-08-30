---
title: "Liquidate function"
weight: 34
layout: book
hiddenInHomeList: true
---

We covered what the `operate` function does for vaults, and we also covered how liquidations work in general [here]({{< ref "../7-borrow-second-time/2-fetch-liquidated-position/1-liquidations-and-factors.md" >}}), but we didn't look quite yet at the implementation and code, so this will be the focus for this section.

Here is the body of the `liquidate` function:

{{< figure src="/fluid-vault/vault/liquidate-function.png" alt="Vault protocol" width="100%" >}}

Ouch, again a 500+ lines of code function ;) and as I promised in the first liquidity layer section, the `operate` function we covered was not the biggest one. This one is certainly more complex.

However, although it seems complex on paper, we already covered quite a lot and we have pretty good base knowledge of abstraction concepts we introduced with ticks and branches.

This implementation will become quite intuitive at the end, and it will just follow the liquidation loop we already explained, of course, with a few more details along the way.

Let's analyze it step by step.