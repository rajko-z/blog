---
title: "Liquidity Layer"
weight: 3
layout: book
hiddenInHomeList: true
---

As mentioned before, all of the liquidity is stored in one contract.  
At the time of writing this, this is the state of mainnet liquidity:  
https://etherscan.io/address/0x52Aa899454998Be5b000Ad077a46Bbe360F4e497#code

{{< figure src="/fluid-vault/liquidity-layer/etherscan-liquidity.png" alt="Liquidity layer" width="60%" >}}

The contract we need to look at is placed inside `contracts/liquidity/userModule/main.sol` and it’s called `FluidLiquidityUserModule`. This will be the main point of analysis alongside `contracts/liquidity/common/variables.sol`, which stores struct and data definitions.


{{< figure src="/fluid-vault/liquidity-layer/folder-structure.png" alt="Liquidity layer" width="40%" >}}

`FluidLiquidityUserModule` contains one core function called `operate`, which is meant to be called by consumers to either deposit, withdraw, borrow, or pay back funds

{{< figure src="/fluid-vault/liquidity-layer/operate-function-signature.png" alt="Liquidity layer" width="100%" >}}

If we go directly to analyze this full function, we will see a body of 500+ lines of code, which at first can look like a complete mess. This is not an uncommon thing for the Fluid codebase, as most of the main flow functions are more like execution pipelines rather than simple “do one thing” functions.

To be able to understand these functions, we will approach them with the standard divide-and-conquer approach, where we break it into multiple parts and analyze snippet by snippet.

Here is the full body of the function:

{{< figure src="/fluid-vault/liquidity-layer/operate-function-full-overview.png" alt="Liquidity layer" width="100%" >}}

Sorry that I made you scroll this long :) but get used to it, as we will break even more complex functions later.

On a high level, this `operate` function can be broken into `11` pieces:

1. `Sanity checks`
2. `Setup memory variables`
3. `DEX callback` This one will be skipped as we are not touching Fluid DEX for this guide
4. `ETH payment check`
5. `Send funds from sender`
6. `Calculate exchange prices`
7. `Supply or withdraw`
8. `Borrow or payback`
9. `Update exchange prices, utilization, and ratios`
10. `Send funds to sender`
11. `Emit event and return`

To track progress easier, and to keep things simple, every piece will have a dedicated page. Some will be straightforward, some not, as they will call other internal functions as well.

So let's start analyzing the pieces, starting from sanity checks performed upon entering the function.