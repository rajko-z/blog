---
title: "Borrow for the second time"
weight: 28
layout: book
hiddenInHomeList: true
---

At this point, we already have a borrow position with an assigned tick and we want to understand what happens when we change that position.

There is only one block of additional code that we will trigger this time because the user tick is initialized (**`tick > type(int).min`**):

{{< figure src="/fluid-vault/vault/fetch-position.png" alt="Vault protocol" width="100%" >}}

The main purpose of this code is to fetch the user position.

Why have this much logic only to fetch data? Can't we just read the tick and recalculate debt?

Well, there are 2 catches:

1. **Liquidations could have happened for the user tick**

   This means that the user's debt and collateral composition is changed alongside its tick. This means that we need to find at which tick liquidations ended and recalculate final debt and collateral.

   Why?

   Because we don't want to update every possible state of position data during liquidations, primarily because it's highly inefficient and gas intensive, and something that's not really needed.

   Fluid instead fetches the user position **on demand**. So the user does not need to interact with his position for a long time. His position can be part of multiple liquidations, but only on demand and on new interaction will we recalculate its position with the latest state.

2. **Position is changed, so we close the old one and recreate a new one**

   We don't have one simple mapping where we can update new collateral and debt. Remember that debt is not stored at all!

   So when we have existing debt, it's highly likely that after the operation we will need to have a different tick from the current one. So what Fluid does, it closes the old position and recalculates it just the way it did on first borrow.

We will analyze both cases in 2 separate pages, starting from the second one first, as it is a little easier to reason about, then move on to the liquidated position case.