---
title: "Liquidations & Factors"
weight: 31
layout: book
hiddenInHomeList: true
---

Let's say we have 4 users opening debt positions on 4 different ticks on the `ETH / USDC` vault, and let's assume `1 ETH = 2500 USDC`.

Alice: 15k debt  
Bob: 10k debt  
Charlie: 25k debt  
Eve: 12k debt  

Important to note is that these examples will have one user per tick for simplicity in visuals, although every tick can have multiple positions in it. Fluid never works with individual positions during liquidations, but rather with tick data, which we will also do.

{{< figure src="/fluid-vault/vault/liquidations-initial-positions.png" alt="Vault protocol" width="60%" >}}

So we have an active branch with active ticks and no liquidation has ever gone through so far.

Now imagine price starts to drop and liquidations start from `topTick`. In this case, it only contains Alice's position.

**Every liquidation iteration will ask how much debt should be liquidated from the current tick, so that after debt / collateral composition moves to exactly one tick below.**

So, in our case, the first liquidation will ask how much debt should be liquidated from tick `A` so that collateral debt composition of that tick moves down to tick `B`. Let's call that `x`, so the formula would be roughly:

```text
(debt - x) / (collateral - x * collateralPerDebt) = ratioB
```

Suppose that moving from tick A to B, liquidator has to liquidate `~1500 USDC`. For this, liquidator will also get collateral with bonus, so at the end `debt / collateral` ratio sits exactly at `B`.

{{< figure src="/fluid-vault/vault/liquidations-from-a-to-b.png" alt="Vault protocol" width="100%" >}}

Let's analyze the image above as a few important steps happened here.

First, we can see that debt at tick A gets lowered by 1.5k, which is the liquidation amount, and that the rest of 13.5k gets carried on to tick B, which has final 23.5k debt.

If we are about to fetch any position from A, in this case only Alice's position, her tick would be at B, not A anymore!

On the image below, we can see one more alternative view of how this debt moving looks:

{{< figure src="/fluid-vault/vault/from-a-to-b.png" alt="Vault protocol" width="100%" >}}

Now there is one more thing left to clarify. On the first image, we also showed something called `debt factor`, which moved from `1` to `0.9`.

**Debt factors** allow us to track where liquidation starts and ends for some branch, and give us the ability to later fetch users and recalculate debt, regardless of their initial tick, how many liquidations their position underwent, and at which points.

Every branch starts with debt factor of `1`, meaning no liquidations happened yet.

As soon as some tick gets in touch with liquidation, we immediately store the current branch factor. I will give some snippet from future code just to get the idea better:

```solidity
// Updating tickData on storage with removing debt & adding connection to branch
tickData[currentData_.tick] =
    1 | // set tick as liquidated
    (temp2_ & 0x1fffffe) | // set same total tick ids
    (branch_.id << 26) | // branch id where this tick got liquidated
    (branch_.debtFactor << 56);
```

In our example, Alice's position is the first one, so she starts from branch factor of 1. This means that **`tickData[A]`** will immediately be marked as liquidated and branch factor of 1 will be stored for tick `A`.

Calculating debt factor after one liquidation iteration is rather straightforward. We just need the initial factor and which percentage of tick debt is liquidated:

```text
newFactor = oldFactor * remainingDebt / previousDebt
newFactor = 1 * 13500 / 15000
newFactor = 0.9
```

If we just stopped liquidation here, our current branch would have final debt factor of `0.9` and retrieving Alice's debt would show us that 10% of the position got liquidated:

```text
initialDebt * endBranchFactor / tickData[A].debtFactor
15000 * 0.9 / 1 = 13500
```

But let's not stop liquidation here, and assume the liquidator has more debt to liquidate, so we keep running liquidation loops.

Next in line is tick `B` with Bob's position and leftover debt from Alice's position with starting debt factor of `0.9`.

We again ask the question: what is the amount of debt needed so we move to ratio `C`?

Suppose the answer is `3916.667 USDC`, which would make remaining debt:

```text
23500 - 3916.667 = 19583.333
```

And new debt factor:

```text
0.9 * 19583.333 / 23500 = 0.75
```

Again, if we stop now, Bob's position debt would be retrieved as:

```text
10000 * 0.75 / 0.9 = 8333.333
```

We can't just multiply by the latest debt factor of `0.75`, as Bob entered the liquidation loop at `0.9`, not at `1` like Alice from the beginning.

This second iteration can be represented visually in the image below:

{{< figure src="/fluid-vault/vault/from-b-to-c.png" alt="Vault protocol" width="100%" >}}

Finally, lets run one more iteration of liquidation.

This time, we again choose the next tick. Note that we are only choosing active ticks with debt, which is in our case tick `E`.

Suppose that the liquidator is only left with `8916.667` USDC after running the previous liquidation, and that moving to `E` from `C` will require more debt. If this happens, we end up somewhere between 2 ticks. The mechanism from which we retrieve this point is called `partials`.

So suppose we have some ratio **`R`** at the perfect tick, and ratio **`R-1`**, and liquidation stops at point **`X`** between these 2 ratios.

```text
ratio(R)

     ● R
     │
     │
     X  ← exact liquidation frontier
     │
     │
     ● R-1

ratio(R-1)
```

Fluid would store minima tick as:

```text
minimaTick = (R-1) + ((R) - (R-1)) * partials
```

If partials is **`0.5`**, we would be exactly at half. If it is **`0.8`**, we would be closer to **`R`**. If it is **`0.2`**, we would be closer to **`R-1`**.

In our example, let's call this point **`D`**.

Now let's go back to debt factors and finish this iteration.

Remaining debt will be:

```text
44583.333 - 8916.667 = 35666.666
```

and debt factor:

```text
0.75 * 35666.666 / 44583.333 = 0.6
```

So final accumulated debt factor from the branch after running liquidation loops is `0.6`:

```text
1 * 0.9 * 0.83333 * 0.8 = 0.6
```

Visualized, this would look like the image below:

{{< figure src="/fluid-vault/vault/final-iteration.png" alt="Vault protocol" width="100%" >}}

We end up at point D, which would be the new `minimaTick` containing all leftover debt.

All ticks above would be marked as liquidated and will store debt factor depending on when they joined liquidation.

`A` will store debt factor of `1`.  
`B` will store debt factor of `0.9`.  
`C` will store debt factor of `0.75`.

And branch `N` will store debt factor of `0.6` with minima tick at point `D`.

If we want to see total amount liquidated and retrieve user positions after liquidation, ignoring `Eve` position which liquidation did not touch:

```text
Positions before:
-----------------
Alice:      15k debt  
Bob:        10k debt  
Charlie:    25k debt  
Total debt: 50k debt

Positions after:
----------------
Alice:      15k * 0.6 / 1    = 9k
Bob:        10k * 0.6 / 0.9  = 6.6k
Charlie:    25k * 0.6 / 0.75 = 20k
Total debt:                  = 35.666k

Liquidated = 14.33k
```

## Merging branches and connection factors

We liquidated the previous branch and finished at minima `D`, but now let's imagine the price goes up again and new users are entering above `minimaTick`.

Suppose we have Felix with 18k debt that comes above `minimaTick` at some tick `F` (highest tick so far), and Grace with 10k debt that got placed at the previously liquidated tick `C`.

For these users, a new branch `N+1` will be generated on top of the previous one (branch `N`), where `N` will be placed as the base branch. The new `topTick` will be `F`.

{{< figure src="/fluid-vault/vault/new-branch.png" alt="Vault protocol" width="100%" >}}

Let's quickly recall what happens when we assign tick `C` to Grace, which was already covered [here]({{< ref "../../6-borrow-first-time.md#assign-new-tick">}}) in case `3.2` when the existing tick already got liquidated.

Because tick `C` got liquidated from the previous branch, we don't want to put Grace's position there, but rather initialize a new active epoch and snapshot the previous liquidated state to history.

In this case, this snapshot for `C` will contain data from the previous liquidation, which will contain branch `N` for the liquidated branch and debt factor of `0.9`.

Now when we decide to run new liquidations later, Grace will have a fresh new state with debt factor of `1` for its latest tick generation.

Okay, the price went up and we got new users. Now, as before, let's assume the price suddenly goes down and we want to run a new liquidation loop.

We are starting from the new `topTick` containing Felix's position, and we again ask: what is the amount of debt `X` that needs to be liquidated so that we reach the next active tick with debt, which is in this case `C`?

Suppose the answer is `3.6k`, meaning:

```text
remainingDebt = 18k - 3.6k = 14.4k
newFactor = oldFactor * remainingDebt / previousDebt
newFactor = 1 * 14.4 / 18 = 0.8
```

This can be shown visually:

{{< figure src="/fluid-vault/vault/new-branch-liquidation-1.png" alt="Vault protocol" width="100%" >}}

Note that at this point we touched tick `C` with entry factor of `0.8`, but Charlie's tick data still lives in history with his `0.75` debt factor, so we will be able to retrieve both positions later.

So we have in storage:

```text
tick C generation G
    Branch N
    debt factor 0.75
    historical (stored in tickId struct)
    Charlie position

tick C generation G + 1
    Branch N + 1
    debt factor 0.8
    current latest (stored in tickData struct)
    Grace position
```

Now, let's say the liquidator still has debt to liquidate, so we run the next liquidation loop.

We look for the next liquidation point. We have 1 active tick sitting at E with debt 12k, but we also have the base branch minima tick sitting on a riskier point. We want to pick the riskiest point. We ask again, what debt should be liquidated so we move to tick D (note that D is not the perfect tick from before, but rather has some partials, for simplicity we will still call it a tick).

Let's say that answer is `9.15k`, meaning:

```text
remainingDebt = 24.4k - 9.15k = 15.25k
newFactor = oldFactor * remainingDebt / previousDebt
newFactor = 0.8 * 15.25 / 24.4 = 0.5
```

If we would retrieve positions from branch `N+1` now, we would get:

```text
Positions before:
-----------------
Felix:      18k debt  
Grace:      10k debt  
Total debt: 28k debt

Positions after:
----------------
Felix:      18k * 0.5 / 1    = 9k
Grace:      10k * 0.5 / 0.8  = 6.25k
Total debt:                  = 15.25k
Liquidated = 12.75k
```

That's all clear, we liquidated enough debt to move to D, but what happens when we touch D?

This moment when we touch the base branch minima tick during liquidation is called a **`merge event`**, where we merge child branch **`N+1`** with its parent (base) branch **`N`**.

The only catch here is that branch `N` has a factor of `0.6` and branch `N+1` has a factor of `0.5`.

So if branch `N+1` is gone as merged, what happens to positions from the `N+1` branch? Can we just start using the base branch debt factor of `0.6`?

Let's see:

```text
Felix:      18k * 0.6 / 1    = 10.8k
Grace:      10k * 0.6 / 0.8  = 7.5k
```

Well, this differs from above, where we know that Felix is left with `9k` and Grace with `6.25k`.

To solve this problem, we want to connect positions from the child branch with the base branch debt factor using something called **`connection factor`**.

Connection factor is calculated as:

```text
connectionFactor = baseBranchFactor / childBranchFactor
```

For our case that would be:

```text
connectionFactor = 0.6 / 0.5
connectionFactor = 1.2
```

So if we take the previous calculations:

```text
Felix:      18k * 0.5 / 1    = 9k
Grace:      10k * 0.5 / 0.8  = 6.25k
```

and multiply by `connectionFactor / connectionFactor`, we would get:

```text
Felix:
18k * 0.5 * 0.6
---------------
     0.5           18k * 0.6     
--------------- = ----------- = 9k
    1 * 0.6           1.2
  -----------
      0.5

Grace:
10k * 0.6 / (0.8 * 1.2) = 6.25k
```

So if we store the connection factor for the child branch, we can still accurately retrieve all positions from that branch. And that's what Fluid stores in branch data as well:

```solidity
// ...
/// Next 50 bits => 116-165 => Connection/adjustment debt factor of this branch with the next branch.
// ...
mapping(uint256 => uint256) internal branchData;
```

Going back to our example, we can visually represent the new state:

{{< figure src="/fluid-vault/vault/merge-event.png" alt="Vault protocol" width="100%" >}}

Now, branch `N` is again the latest liquidation frontier, with `minimaTick` being `topTick` again. So let's say the liquidator still has some debt to liquidate, so we will run one more liquidation iteration.

The beauty of this design is that everything we covered before is still relevant, the new iteration is continuing on branch `N` just like before.

We ask what is the next tick with debt, which is Eve's position in tick `E` now, that was never touched by liquidations.

Let's say to reach `E` we need `10.1833k` and the liquidator has that amount, so we calculate:

```text
remainingDebt = 50.9167k - 10.1833k = 40.7334k
newFactor = oldFactor * remainingDebt / previousDebt
newFactor = 0.6 * 40.7334k / 50.9167k = 0.48
```

After this liquidation, the liquidator will still have some debt, so we finish this iteration, we touch Eve's debt at tick `E` and move it to the liquidation branch, having around `52.7334k` of debt sitting at tick `E`.

{{< figure src="/fluid-vault/vault/fluid-vault-liquidation-branch-merge.png" alt="Vault protocol" width="100%" >}}

Now let's finish this second liquidation process, and let's say the liquidator has only `10.5467k` left to liquidate, which is not enough to reach some other active ticks. For simplification, suppose there are some active debt positions below Eve's position on branch `N` which we didn't explicitly introduce.

So we again calculate:

```text
remainingDebt = 52.7334k - 10.5467k = 42.1867k
newFactor = oldFactor * remainingDebt / previousDebt
newFactor = 0.48 * 42.1867k / 52.7334k = 0.384
```

and finish this liquidation at some point `X` that has partials (somewhere between `E` and some other active tick).

We end up with the final state:

{{< figure src="/fluid-vault/vault/final-state-liquidation.png" alt="Vault protocol" width="100%" >}}

If we want to retrieve back all positions:

```text
Positions before:
-----------------
Alice:      15k debt
Bob:        10k debt
Charlie:    25k debt
Eve:        12k debt
Felix:      18k debt
Grace:      10k debt
Total debt: 90k debt

Positions after:
----------------
Alice:      15k * 0.384 / 1             = 5.760k
Bob:        10k * 0.384 / 0.9           = 4.267k
Charlie:    25k * 0.384 / 0.75          = 12.800k
Eve:        12k * 0.384 / 0.48          = 9.600k
Felix:      18k * 0.384 / (1 * 1.2)     = 5.760k
Grace:      10k * 0.384 / (0.8 * 1.2)   = 4.000k
                                        -------
Total debt:                             = 42.187k

Total liquidated:
-----------------
90k - 42.187k = 47.8133k
```

---------

Uf, we covered a lot, so let's cool down a bit and summarize what we did above.

- So we created the initial branch with a few users
- Price went down so we ran the first liquidation, moving to the final debt factor `0.6` and stopping at minima `D`
- Then price went up, and we introduced a few new users on a new branch that was created on top of the previous branch `N`
- Then again, price went down so we ran the second liquidation call, which merged both branches and finished at `0.384` debt factor on branch `N`

During these examples, we learned how liquidations work in general, what debt factors are used for, and what connection factors are and how they help us retrieve the state after a merge event.

Knowing this, reading the implementation on the next pages will be much more intuitive and simpler to reason about.

Until then, you can visualize the whole process from above:

{{< figure src="/fluid-vault/vault/liquidation-flow.png" alt="Vault protocol" width="100%" >}}