---
title: "Ticks"
weight: 25
layout: book
hiddenInHomeList: true
---

On the previous pages, we were ignoring ticks as we weren't dealing with debt, but what are the ticks?

If you are familiar with ticks from Uniswap V3, then the same thing ticks are for concentrated liquidity there, here they are for separating positions by their ratio.

```
ratio = 1.0015^tick
```

Where ratio is:

```
ratio = debt / coll
```

This means:

- higher the ratio -> higher the tick -> higher the risk of liquidation

and vice versa

- lower the ratio -> lower the tick -> lower the risk of liquidation

**The main idea here is to group users by their risk, in this tick abstraction, which will later allow Fluid to perform batched liquidations, which we will go into details later.**

You would expect most DeFi protocols to store debt principal in some separate user storage data, but because Fluid is storing ratio and collateral, this means that **storing debt is redundant as it can be recomputed**.

Let's go with an example and explain design decisions along the way.

---

## Implementation details

Let's say Bob interacts with the ETH/USDC vault, depositing 1 ETH and borrowing 1000 USDC.

His ratio would be calculated as:

```
ratio = (1000 * 1e6) / (1 * 1e18) = 0.000000001
```

We can get the `tick` from ratio:

```python
>>> import math
>>> math.log(1e-9, 1.0015)
-13825.869602414956
>>> 
```

Immediately, there are a few questions we can ask from here:

1. How to store these decimal numbers, like `1e-9` for ratio?
2. Why use `1.0015`, what is the meaning?
3. What is the minimal and maximal value for tick? `~ -13826` seems pretty random
4. If tick is a whole number, what happens with debt precision loss?

### 1. Storing decimal numbers, Q96 format

We can't store decimals in Solidity, that's why Fluid uses Q96 fixed-point arithmetic. Ratio from above would be stored as:

```
ratioX96 = 0.000000001 * 2^96
```

This also means that the actual formula on the Solidity side will be used as:

```
ratioX96 = 1.0015^tick * 2^96
```

Q formats are not Fluid-specific, so we won't analyze them here, but for those interested, check this great article from RareSkills: `https://rareskills.io/post/q-number-format`

### 2. Why use 1.0015, what is the meaning?

We have to have some meaningful distinction between user groups. So just like Uniswap V3 and price variation, Fluid uses 15 bps to represent the smallest relative change in ratio.

```
ratio[t + 1] / ratio[t] = 1.0015
```

This is a good balance of having a small enough change that is practical and already proven for implementation.

Another approach is to decrease this spacing even further, which would lower the dust amounts from accounting we will later see, but would also introduce the overhead of having more ticks and more groups, which also means more operations as there will be more iterations throughout ticks.

If we increase the spacing, however, the difference between calculated decimal tick and actual rounded tick is bigger, meaning there is more dust to handle.

Let's quickly run through examples for the formula above:

```
tick-1 -> ratio = 1.0015^(-1) = 0.9985022466300548
tick0  -> ratio = 1.0015^0    = 1
tick1  -> ratio = 1.0015^1    = 1.0015
tick2  -> ratio = 1.0015^2    = 1.0030022500000002
```

Which matches the exact spacing of `1.0015`:

```
ratio[0] / ratio[-1] = 1 / 0.9985022466300548      = 1.0015
ratio[1] / ratio[0]  = 1.0015 / 1                  = 1.0015
ratio[2] / ratio[1]  = 1.0030022500000002 / 1.0015 = 1.0015
```

### 3. What is the minimal and maximal value for tick?

Fluid uses the following range for ticks: `[-32767, 32767]`.

Why?

Well, it fits a large range of possible ratios. For example, look again at the first example where 1 ETH and 1000 USDC generated tick `-13826`.

Here the tick is going faster in one direction because one token has 18 decimals and one has 6 decimals. But let's lower the debt even more:

```
1 ETH / 1000   USDC = 1e-9  -> tick ~ -13826
1 ETH / 100    USDC = 1e-10 -> tick ~ -15362
1 ETH / 10     USDC = 1e-11 -> tick ~ -16898
1 ETH / 1      USDC = 1e-12 -> tick ~ -18434
1 ETH / 0.1    USDC = 1e-13 -> tick ~ -19971
1 ETH / 0.01   USDC = 1e-14 -> tick ~ -21507
1 ETH / 0.001  USDC = 1e-15 -> tick ~ -23043
1 ETH / 0.0001 USDC = 1e-16 -> tick ~ -24579

...

1 ETH / 0.000000001 USDC = 1e-21 -> tick ~ -32260
```

So we see that the difference between collateral and debt has to be so huge that ticks leaving the range is not expected for any practical use case.

The other reason is that these values can simplify implementation, as:

```
32767 = 2^15 - 1 = 0x7fff = 111111111111111₂
```

If we look at the factors table and combine all values, we will get:

```
1 + 2 + 4 + ... + 16384 = 32767
```

These are exactly the same factors Fluid uses in its `TickMath` library:

{{< figure src="/fluid-vault/vault/factors.png" alt="Vault protocol" width="100%" >}}

So for tick `13`, for example, instead of multiplying by 1.0015 thirteen times, we can decompose and use pre-calculated factors:

```
1.0015^13 = 1.0015^8 * 1.0015^4 * 1.0015^1
```

### 4. If tick is a whole number, what happens with debt precision loss?

There is only one main question left to answer. We already mentioned in the first example that supplying 1 ETH and borrowing 1000 USDC would give ratio of:

```
-13825.869602414956
```

We know that ticks are whole numbers, so in this case we must go in the protocol direction, meaning we want to put the user on the riskier side. We don't want to cut the debt.

Riskier side is always one tick above. In this case, it's `-13825`.

But now, debt is not `1000 USDC` anymore, it's slightly more:

```
debt = ratio * coll = 1.0015^(-13825) * 1e18 = 1001304276 ~ 1001.3 USDC
```

So Fluid does not account this as automatic debt for the user, it rather stores this as `dustDebt`:

```
dustDebt ~ 1.3 USDC
```

So when we mentioned Fluid does not store debt, we were "lying" a little bit. Yes, it does not store debt, but it stores dust debt from the imperfect ratio. The only purpose of this dust debt is to recompute the exact debt user generated. Without this info, we would only see `1001.3 USDC` as debt, but by storing an additional `1.3 USDC` as dust, we can recompute exact debt of `1000`.

---

{{< figure src="/fluid-vault/vault/fluid-ticks.png" alt="Vault protocol" width="100%" >}}