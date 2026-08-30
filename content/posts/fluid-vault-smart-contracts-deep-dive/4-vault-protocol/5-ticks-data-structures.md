---
title: "Ticks data structures"
weight: 26
layout: book
hiddenInHomeList: true
---

Before we dive into borrow flow, let's quickly introduce some tick data structures that will help us later to understand the code better.

Note that we will analyze liquidations in depth later, so some of these concepts will only make sense then, but I want to introduce them now so we conceptually know what is going on.

## TickData

```solidity
/// mapping tickId => tickData
/// Tick related data. Total debt & other things
/// First bit => 0 => If 1 then liquidated else not liquidated
/// Next 24 bits => 1-24 => Total IDs. ID should start from 1.
/// If not liquidated:
/// Next 64 bits => 25-88 => raw debt
/// If liquidated
/// The below 3 things are of last ID. This is to be updated when user creates a new position
/// Next 1 bit => 25 => Is 100% liquidated? If this is 1 meaning it was above max tick when it got liquidated (100% liquidated)
/// Next 30 bits => 26-55 => branch ID where this tick got liquidated
/// Next 50 bits => 56-105 => debt factor 50 bits (35 bits coefficient | 15 bits expansion)
mapping(int256 => uint256) internal tickData;
```

`TickData` describes the latest info for some tick. We will call that info the latest epoch.

What does this mean?

Every tick goes through phases. A tick can be active, then liquidated, then active again, etc. Those phases can be called epochs.

`TickData` only keeps the latest epoch for some tick.

Let's say from our previous example:

1. a few users were supplying to the `ETH/USDC` vault, and all positions were placed into tick `-13825` -> **epoch 0**
2. price went down suddenly, and all those users in tick `-13825` got liquidated -> **epoch 2**
3. price went up and new users got placed into tick `-13825` -> **epoch 3**

Now in this case, **tickData** would only store info for **epoch 3**.

By looking at fields, besides storing latest total debt, it will also store `Total IDs`. These total IDs are actually epochs, or the number of phases this tick has passed so far.

We can also see that if the tick is liquidated, we have 3 additional fields. We will talk about liquidations in depth later, but for now it is enough to understand that liquidations will move through bigger ticks (highest risks) to lower ticks (less risky ticks) and liquidate enough debt to recover health factor. Some ticks will be fully liquidated, and some will be partially liquidated.

These 3 additional fields will be:

- info if the tick is fully liquidated or not
- branch ID
- debt factor

Conceptually, think of a branch as one liquidation cycle. It will start at the highest risk tick called **topTick** and it will finish at **minimaTick**. Liquidation can be finished right at the tick or somewhere between 2 ticks.

Just like tick epochs, there can be many branches, but only 1 latest active branch.

{{< figure src="/fluid-vault/vault/liquidation-zone.png" alt="Vault protocol" width="100%" >}}

Debt factor, on the other hand, will be used as a connection between different branches, which we will use to recreate a position after liquidation events.

## TickId

```solidity
/// tick id => previous tick id liquidation data. ID starts from 1
/// One tick ID contains 3 IDs of 80 bits in it, holding liquidation data of previously active but liquidated ticks
/// 81 bits data below
/// #### First 85 bits ####
/// 1st bit => 0 => Is 100% liquidated? If this is 1 meaning it was above max tick when it got liquidated
/// Next 30 bits => 1-30 => branch ID where this tick got liquidated
/// Next 50 bits => 31-80 => debt factor 50 bits (35 bits coefficient | 15 bits expansion)
/// #### Second 85 bits ####
/// 85th bit => 85 => Is 100% liquidated? If this is 1 meaning it was above max tick when it got liquidated
/// Next 30 bits => 86-115 => branch ID where this tick got liquidated
/// Next 50 bits => 116-165 => debt factor 50 bits (35 bits coefficient | 15 bits expansion)
/// #### Third 85 bits ####
/// 170th bit => 170 => Is 100% liquidated? If this is 1 meaning it was above max tick when it got liquidated
/// Next 30 bits => 171-200 => branch ID where this tick got liquidated
/// Next 50 bits => 201-250 => debt factor 50 bits (35 bits coefficient | 15 bits expansion)
mapping(int256 => mapping(uint256 => uint256)) internal tickId;
```

`TickId` is just the mapping for liquidation history of tick epochs. So just like `tickData` stored latest info for liquidation in terms of 3 fields if the latest epoch got liquidated, `tickId` stores the same 3 fields but for the 3 latest epochs.

The 3 is picked as an implementation detail, as it allows us to put 3 epoch infos in one `uint256`, so querying later can be performed more gas efficiently.

## TickHasDebt

```solidity
/// Tick has debt only keeps data of non liquidated positions. liquidated tick's data stays in branch itself
/// tick parent => uint (represents bool for 256 children)
/// parent of (i)th tick:-
/// if (i>=0) (i / 256);
/// else ((i + 1) / 256) - 1
/// first bit of the variable is the smallest tick & last bit is the biggest tick of that slot
mapping(int256 => uint256) internal tickHasDebt;
```

`TickHasDebt` is yet another bitmap storing info about current active ticks having non-zero total debt.

The above 2 structs, `TickData` and `TickId`, were telling us info for epochs of one specific tick like `-13825`, while this `TickHasDebt` struct will give us total active debt of ticks (e.g. `-13825`, `-1`, `7405`...), so it does not have anything to do with epochs.

Note one important thing mentioned by the comment:

```solidity
/// Tick has debt only keeps data of non liquidated positions. liquidated tick's data stays in branch itself
```

Which, if we think about it, matches what we described so far, as we said that `TickData` will only store debt of the latest epoch, and `TickId` will store history info of liquidations, but none of those liquidated epochs in history will have debt stored in the `tickId` mapping, which means it will be stored on the branch data during liquidation.

---

Whoa, `tick`, `tickId`, `tickData`, `tickHasDebt`, `branch`, `branch factor`, we introduced all of these concepts at once, so you might be confused at this point. It will all make sense once we start digging deeper into the code.

On the next page, we will go through the flow when the user performs borrow for the first time.