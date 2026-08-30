---
title: "Fluid Vault Smart Contracts - Deep Dive"
date: 2026-08-30
layout: book
description: "A technical deep dive into Fluid's liquidity layer, vault protocol, ticks, branches, and liquidation mechanics."
images:
  - "/fluid-vault/vault/fluid-vault-liquidation-branch-merge.png"
---

### Who is this guide for?

This guide is for all Solidity devs and security researchers who want to understand the technical details behind Fluid, one of the leading lending protocols in DeFi, or simply for anyone looking to level up their DeFi game.

### Who is this guide not for?

This guide is not meant for people looking for a quick Fluid introduction or a showcase of its different features from a user perspective. There are tons of good resources out there that can teach you that pretty quickly.

### Why Fluid?

Because it is one of the most innovative DeFi projects out there. It is also one of the most complex ones. You can expect reading to be quite challenging, but the ideas gathered along the way will make it worth it.

### Prerequisites

It's expected that you are already familiar with DeFi concepts like collateralization, borrow and supply interest rates, liquidations, etc. If you are unfamiliar with those, I recommend reading about some other DeFi lending protocol first, as Fluid is quite heavy for a first DeFi introduction.

It's also expected that you are pretty comfortable with EVM and Solidity. Fluid doesn't shy away from complexity, and you can expect to see heavy usage of bitwise operations and memory optimizations in almost every operation.

### Codebase & Commit

Codebase we are about to analyze can be found here: https://github.com/Instadapp/fluid-contracts-public  

Commit: `a9949b48ba1247d4f478cd0acb40896b5c8bf3f8`