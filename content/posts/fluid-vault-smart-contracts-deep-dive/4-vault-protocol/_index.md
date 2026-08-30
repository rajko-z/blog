---
title: "Vault Protocol"
weight: 21
layout: book
hiddenInHomeList: true
---

We are here:

{{< figure src="/fluid-vault/vault/architecture-update.jpg" alt="Vault protocol" width="80%" >}}

We just finished the liquidity module, and now we are tackling the core of Fluid: its vault protocol.

Vault protocol is the main consumer of the liquidity layer and also the main entry point for users to manage their collateralized debt positions.

Expect this module to be quite challenging reading, as it has way more logic and abstractions than the liquidity layer.

It's advised to not read everything at once, and ideally to try to analyze the code before reading explanations.

We will first start slowly, by explaining some higher-level concepts, which will give us momentum to understand more complex topics later, like internals of the Fluid liquidation engine.