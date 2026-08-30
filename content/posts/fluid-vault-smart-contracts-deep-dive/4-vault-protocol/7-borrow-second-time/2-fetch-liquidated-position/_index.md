---
title: "Fetch liquidated position"
weight: 30
layout: book
hiddenInHomeList: true
---

We want to load a position that went through liquidations. As a reminder, we are looking at this piece of code:

{{< figure src="/fluid-vault/vault/liquidated-position.png" alt="Vault protocol" width="100%" >}}

The main logic will be part of the first `fetchLatestPosition` function, and the rest will be just an update of internal structures.

So this is the core function:

{{< figure src="/fluid-vault/vault/fetch-latest-position.png" alt="Vault protocol" width="100%" >}}

This one is heavy, and to grasp it fully, we need some background.

On the next page, we will introduce the liquidation loop from a high level perspective and will also introduce the concepts of debt and connection factors. After that, we will return and analyze this function.