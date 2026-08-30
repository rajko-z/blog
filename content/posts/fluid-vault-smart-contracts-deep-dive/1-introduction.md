---
title: "Introduction"
weight: 1
layout: book
hiddenInHomeList: true
---

### Overall architecture

Let's take a peek at overall Fluid architecture:

{{< figure src="/fluid-vault/architecture.jpg" alt="Fluid architecture" width="80%" >}}

We can see **liquidity** at the center, with a few other components branching from there. This is how Fluid can keep iterating faster, as they don’t have to bootstrap liquidity for new products (we can also call them consumers). Liquidity is aggregated in one contract which is handy if you have multiple versions of the protocol, as you don’t have to wait for years for users to migrate to newer versions, you just reuse the liquidity from before.

The **lending protocol** is there for users to supply liquidity without opening any borrow positions. It is conceptually implemented like an ERC4626 vault-like component.

**Periphery** is a classical design pattern, originated from Uniswap, where non-core logic, like view methods, bundlers, and other helpers, are extracted into a separate module.

**Vault protocol** is the core one and the main consumer of liquidity. This is where the borrow/lending implementation exists.

Besides these modules, there can be infinitely many other consumers of liquidity, like a DEX protocol for example.

The focus of this guide will be on Liquidity and Vault protocol. Although the DEX protocol introduces many innovations, it is a beast on its own, with concepts like smart collateral and smart debt, and will be covered in some future guides on this blog. For now, we have a lot of work to do analyzing the Liquidity and Vault protocol themselves.

### Solidity codebase

In the picture below, we can see how the Solidity codebase is organized. We will spend most of the time on selected folders that map to the Liquidity layer and Vault protocol.

Note that for the Vault protocol there are concepts of T1/T2/T3/T4 vaults. For us, the focus right now is on T1 vaults, which are standard vaults containing one collateral and one debt. Other vaults will be covered in the future, as they are meant to work with the Fluid DEX protocol.

{{< figure src="/fluid-vault/project-structure.png" alt="Fluid project structure" width="40%" >}}
