---
title: "1st supply/withdraw no debt"
weight: 23
layout: book
hiddenInHomeList: true
---

To keep things simpler at the start, let's just completely ignore the borrow side at the moment, and only analyze how user interaction with the vault protocol works on plain supply and withdraw.

In the code below, we will hide all code that triggers on borrow behind `...` and only focus on supply/withdrawal operations.

{{< figure src="/fluid-vault/vault/operate-supply-withdraw.png" alt="Vault protocol" width="100%" >}}

We can divide this flow into 11 steps:

1. Sanity checks
2. Setup of memory vars
3. Creation of new position
4. Update of exchange prices
5. Calculation of supply or withdrawal amounts
6. Assign of new tick
7. Calculation of health factor
8. Update of user position in storage
9. Withdrawal gap check
10. Execution of operation
11. Update of vault variables in storage

## 1. Sanity checks

We first check for reentrancy.

```solidity
uint256 vaultVariables_ = vaultVariables;
// re-entrancy check
if (vaultVariables_ & 1 == 0) {
    // Updating on storage
    vaultVariables = vaultVariables_ | 1;
} else {
    revert FluidVaultError(ErrorTypes.Vault__AlreadyEntered);
}
```

Reentrancy is checked by looking at the first bit of vault variables, which follows the bitmap approach we already saw before.

```solidity
/// note: vaultVariables contains vault variables which need regular updates through transactions
/// First 1 bit => 0 => re-entrancy. If 0 then allow transaction to go, else throw.
/// Next 1 bit => 1 => Is the current active branch liquidated? If true then check the branch's minima tick before creating a new position
/// If the new tick is greater than minima tick then initialize a new branch, make that as current branch & do proper linking
/// Next 1 bit => 2 => sign of topmost tick (0 -> negative; 1 -> positive)
/// Next 19 bits => 3-21 => absolute value of topmost tick
/// Next 30 bits => 22-51 => current branch ID
/// Next 30 bits => 52-81 => total branch ID
/// Next 64 bits => 82-145 => Total supply
/// Next 64 bits => 146-209 => Total borrow
/// Next 32 bits => 210-241 => Total positions
uint256 internal vaultVariables;
```

For now, we can ignore other fields from this bitmap.

After the reentrancy check, we validate input amounts against zero and dust values:

```solidity
if (
    (newCol_ == 0 && newDebt_ == 0) ||
    // withdrawal or deposit cannot be too small
    ((newCol_ != 0) && (newCol_ > -10000 && newCol_ < 10000)) ||
    // borrow or payback cannot be too small
    ((newDebt_ != 0) && (newDebt_ > -10000 && newDebt_ < 10000))
) {
    revert FluidVaultError(ErrorTypes.Vault__InvalidOperateAmount);
}
```

And finally, if we are doing native token supply, the `newCol_` param passed to the function has to match `msg.value` sent:

```solidity
// Check msg.value aligns with input amounts if supply or borrow token is native token.
// Note that it's not possible for a vault to have both supply token and borrow token as native token.
if (SUPPLY_TOKEN == NATIVE_TOKEN && newCol_ > 0) {
    if (uint(newCol_) != msg.value) {
        revert FluidVaultError(ErrorTypes.Vault__InvalidMsgValueOperate);
    }
} else if (msg.value > 0) {
    ...
}
```

## 2. Setup memory vars

```solidity
OperateMemoryVars memory o_;
// Temporary variables used as helpers at many places
uint256 temp_;
uint256 temp2_;
int256 temp3_;

o_.vaultVariables2 = vaultVariables2;

temp_ = (vaultVariables_ >> 2) & X20;
unchecked {
    o_.topTick = (temp_ == 0)
        ? type(int).min
        : ((temp_ & 1) == 1)
            ? int((temp_ >> 1) & X19)
            : -int((temp_ >> 1) & X19);
}
```

This part creates the operate memory vars struct that will be used later. The struct looks like this:

```solidity
struct OperateMemoryVars {
    // ## User's position before update ##
    uint oldColRaw;
    uint oldNetDebtRaw; // total debt - dust debt
    int oldTick;
    // ## User's position after update ##
    uint colRaw;
    uint debtRaw;
    uint dustDebtRaw;
    int tick;
    uint tickId;
    // others
    uint256 vaultVariables2;
    uint256 branchId;
    int256 topTick;
    uint liquidityExPrice;
    uint supplyExPrice;
    uint borrowExPrice;
    uint branchData;
    // user's supply slot data in liquidity
    uint userSupplyLiquidityData;
}
```

For us, at the moment, only a few fields from this struct will be relevant.

Besides this, it also initializes something called `topTick`. Ticks are core to Fluid position management and liquidations, but for the case we are analyzing, they are not relevant as we don't have any debt.

## 3. Create new position

```solidity
{
    // Fetching user's position
    if (nftId_ == 0) {
        // creating new position.
        o_.tick = type(int).min;
        // minting new NFT vault for user.
        nftId_ = VAULT_FACTORY.mint(VAULT_ID, msg.sender);
        // Adding 1 in total positions. Total positions cannot exceed 32bits as NFT minting checks for that
        unchecked {
            vaultVariables_ = vaultVariables_ + (1 << 210);
        }
    } else {
        // Updating existing position
        ...
    }
}
```

When the user interacts with Fluid vault for the first time, they don't have any positions, so we pass the function param `nftId_` as zero for the first time.

Because Fluid vault factory inherits `ERC721`, every user position will be represented as an NFT.

We can quickly jump to the specific implementation for the factory to show this case:

```solidity
function mint(uint256 vaultId_, address user_) external returns (uint256 tokenId_) {
    if (msg.sender != getVaultAddress(vaultId_)) revert FluidVaultError(ErrorTypes.VaultFactory__InvalidVault);

    // Using _mint() instead of _safeMint() to allow any msg.sender to receive ERC721 without onERC721Received holder.
    tokenId_ = _mint(user_, vaultId_);

    emit NewPositionMinted(msg.sender, user_, tokenId_);
}
```

So we see that this function simply validates that the caller is a valid vault contract for the provided ID, and then it calls the underlying `_mint` function to give this new position NFT to the user.

After NFT creation, we update the global positions size for the vault by shifting to the right place in the bitmap:

```solidity
vaultVariables_ = vaultVariables_ + (1 << 210);
```

## 4. Update of exchange prices

```solidity
(o_.liquidityExPrice, , o_.supplyExPrice, o_.borrowExPrice) = updateExchangePrices(o_.vaultVariables2);
```

Before we start analyzing this part, let's think about what the exchange prices are now.

## 4.1 We already talked about exchange prices, why again?

We already covered in depth how liquidity layer exchange prices work in the first chapter, so what's this now?

We have to remember the Fluid architecture. There are 2 main types of accounting:

1. Liquidity layer users
2. Protocol users (in this case Vault users)

So the liquidity layer sees some vault as a regular user, although actual real users interact with the vault.

If Fluid decided to use the same exchange prices as the liquidity layer, this would force all vaults to have the same base economics.

It wouldn't be possible to have, for example, some extra reward for the supply side or lower borrow cost, as all vaults would follow the same growth.

To prevent this, Fluid decides that vaults are not strictly meant to follow the same exchange prices of the underlying liquidity.

Example
--------

Let's say some vault has 100m supplied liquidity and let's say this global supply rate on the liquidity layer is 10%.

This means that after 1 year, 100m supplied from the vault would be worth 110m.

But, let's say that the actual supply rate on the vault side is 12%, 2% bigger than the liquidity layer. In this case, we have the following:

```
Liquidity gives the Vault effectively: 10%
Vault promises its users:              12%

Vault's actual position in Liquidity: 110M
Vault users' accounting             : 112M

Deficit = 2m
```

So the vault is promising its users a better rate than the base liquidity layer. How is this possible?

Well, some vault can get **external rewards** as an example, which would cover this deficit.

The way this deficit is covered, in this case, is through a special `rebalance` function which we won't analyze in this chapter.

For now, all we need to know is that some external actor can rebalance the vault exchange prices with liquidity layer prices.

In this case, we would just pull an additional 2m from a special address which would cover the deficit by sending the deficit to the liquidity layer, making the after state:

```
Vault's actual position in Liquidity: 112M
Vault users' accounting             : 112M
```

## 4.2 Implementation

Knowing the idea behind managing exchange prices on both levels (liquidity and now vault), we can see the implementation:

{{< figure src="/fluid-vault/vault/update-exchange-prices.png" alt="Vault protocol" width="80%" >}}

Implementation can be divided into 2 main parts, calculation of liquidity layer prices and vault prices.

### 4.2.1 Calculate liquidity layer exchange prices

```solidity
// Fetching last stored rates
uint rates_ = rates;

(liqSupplyExPrice_, ) = LiquidityCalcs.calcExchangePrices(
    LIQUIDITY.readFromStorage(LIQUIDITY_SUPPLY_EXCHANGE_PRICE_SLOT)
);
(, liqBorrowExPrice_) = LiquidityCalcs.calcExchangePrices(
    LIQUIDITY.readFromStorage(LIQUIDITY_BORROW_EXCHANGE_PRICE_SLOT)
);

uint256 oldLiqSupplyExPrice_ = (rates_ & X64);
uint256 oldLiqBorrowExPrice_ = ((rates_ >> 64) & X64);
if (liqSupplyExPrice_ < oldLiqSupplyExPrice_ || liqBorrowExPrice_ < oldLiqBorrowExPrice_) {
    // new liquidity exchange price is < than the old one. liquidity exchange price should only ever increase.
    // If not, something went wrong and avoid proceeding with unknown outcome.
    revert FluidVaultError(ErrorTypes.Vault__LiquidityExchangePriceUnexpected);
}
```

This part simply calculates liquidity layer prices by calling it directly on the liquidity layer. We already covered the implementation here: [LiquidityCalcs.calcExchangePrices]({{< ref "../3-liquidity-layer/6-calculate-exchange-prices/liquiditycalcs-calc-exchange-prices.md" >}}).

After we calculate new prices, we extract previously stored prices from the `rates` bitmap, another bitmap storing all 4 exchange prices:

```solidity
/// Exchange prices are in 1e12
/// First 64 bits => 0-63 => Liquidity's collateral token supply exchange price
/// First 64 bits => 64-127 => Liquidity's debt token borrow exchange price
/// First 64 bits => 128-191 => Vault's collateral token supply exchange price
/// First 64 bits => 192-255 => Vault's debt token borrow exchange price
uint256 internal rates;

uint256 oldLiqSupplyExPrice_ = (rates_ & X64);
uint256 oldLiqBorrowExPrice_ = ((rates_ >> 64) & X64);
```

Finally, we perform a sanity check to make sure new liquidity prices are greater than previously stored prices, as supply and borrow exchange prices should only ever increase over time.

```solidity
if (liqSupplyExPrice_ < oldLiqSupplyExPrice_ || liqBorrowExPrice_ < oldLiqBorrowExPrice_) {
    // new liquidity exchange price is < than the old one. liquidity exchange price should only ever increase.
    // If not, something went wrong and avoid proceeding with unknown outcome.
    revert FluidVaultError(ErrorTypes.Vault__LiquidityExchangePriceUnexpected);
}
```

### 4.2.2 Calculate vault exchange prices

The second part is more interesting:

```solidity
// Calculating increase in supply exchange price w.r.t last stored liquidity's exchange price
// vaultSupplyExPrice_ => supplyIncreaseInPercent_
vaultSupplyExPrice_ = ((((liqSupplyExPrice_ * 1e18) / oldLiqSupplyExPrice_) - 1e18) *
    (vaultVariables2_ & X16)) / 10000; // supply rate magnifier

// Calculating increase in borrow exchange price w.r.t last stored liquidity's exchange price
// vaultBorrowExPrice_ => borrowIncreaseInPercent_
vaultBorrowExPrice_ = ((((liqBorrowExPrice_ * 1e18) / oldLiqBorrowExPrice_) - 1e18) *
    ((vaultVariables2_ >> 16) & X16)) / 10000; // borrow rate magnifier

// It's extremely hard the exchange prices to overflow even in 100 years but if it does it's not an
// issue here as we are not updating on storage
// (rates_ >> 128) & X64) -> last stored vault's supply token exchange price
vaultSupplyExPrice_ = (((rates_ >> 128) & X64) * (1e18 + vaultSupplyExPrice_)) / 1e18;
// (rates_ >> 192) -> last stored vault's borrow token exchange price (no need to mask with & X64 as it is anyway max 64 bits)
vaultBorrowExPrice_ = ((rates_ >> 192) * (1e18 + vaultBorrowExPrice_)) / 1e18;
```

It first calculates the growth factor of liquidity exchange prices, and adapts it by supply and borrow rate magnifiers.

These magnifiers are part of the `vaultVariables2` bitmap and they dictate how vault exchange prices would scale:

```solidity
/// First 16 bits => 0-15 => supply rate magnifier; 10000 = 1x (Here 16 bits should be more than enough)
/// Next 16 bits => 16-31 => borrow rate magnifier; 10000 = 1x (Here 16 bits should be more than enough)
```

E.g. having the supply rate magnifier of `12000` would mean that the vault supply rate would grow more than the base layer, creating an additional incentive.

The same way, having the magnifier below `10000` would produce a lower vault supply rate than the base layer, which would make a spread that can be later claimed by rebalancing.

After the growth factor calculation, it simply multiplies previously stored vault exchange prices to get new values.

## 5. Calculate supply or withdraw amounts

Once we calculate exchange prices from the previous steps, we need to calculate raw amounts (without the interest component):

```solidity
// supply or withdraw
if (newCol_ > 0) {
    // supply new col, rounding down
    o_.colRaw += (uint256(newCol_) * EXCHANGE_PRICES_PRECISION) / o_.supplyExPrice;
    // final user's collateral should not be above 2**128 bits
    if (o_.colRaw > X128) {
        revert FluidVaultError(ErrorTypes.Vault__UserCollateralDebtExceed);
    }
} else if (newCol_ < 0) {
    // if withdraw equals type(int).min then max withdraw
    if (newCol_ > type(int128).min) {
        // partial withdraw, rounding up removing extra wei from collateral
        temp3_ = ((newCol_ * int(EXCHANGE_PRICES_PRECISION)) / int256(o_.supplyExPrice)) - 1;
        unchecked {
            if (uint256(-temp3_) > o_.colRaw) {
                revert FluidVaultError(ErrorTypes.Vault__ExcessCollateralWithdrawal);
            }
            o_.colRaw -= uint256(-temp3_);
        }
    } else if (newCol_ == type(int).min) {
        // max withdraw, rounding up:
        // adding +1 to negative withdrawAmount newCol_ for safe rounding (reducing withdraw)
        newCol_ = -(int256((o_.colRaw * o_.supplyExPrice) / EXCHANGE_PRICES_PRECISION)) + 1;
        o_.colRaw = 0;
    } else {
        revert FluidVaultError(ErrorTypes.Vault__UserCollateralDebtExceed);
    }
}
```

If we are doing supply, we simply divide the passed param `newCol_` with the newly calculated supply exchange price.

For withdrawal, logic is more complex, as we also want to handle max withdrawal, which can be achieved by passing `type(int).min`.

In both cases, partial and max withdrawal, the rounding is performed in favor of the protocol, by explicitly withdrawing 1 unit less.

## 6. Assign new tick

As said before, we are isolating the case with no borrow, so we assign the tick to the lowest possible value (doesn't mean anything to us at the moment...)

```solidity
// Assign new tick
if (o_.debtRaw > 0) {
    ...
} else {
    // debtRaw_ remains 0 in this situation
    // This kind of position will not have any tick. Meaning it'll be a supply position.
    o_.tick = type(int).min;
}
```

## 7. Calculate health factor

This logic is only triggered on new borrow or withdrawal.

As we don't have debt, the withdrawal case will also be ignored for now.

```solidity
if (newCol_ < 0 || newDebt_ > 0) {
    // withdraw or borrow
    if (to_ == address(0)) {
        to_ = msg.sender;
    }

    unchecked {
        if (
            o_.debtRaw > 0 &&       // <---------------------- NO DEBT AT THE MOMENT
            (o_.oldTick <= o_.tick ||
                (o_.debtRaw - o_.dustDebtRaw) > (((o_.oldNetDebtRaw * 1000000001) / 1000000000) + 1))
        ) {
            ...
        }
    }
}
```

## 8. Update user position data in storage

```solidity
// Updating user's new position on storage
// temp_ => tick to insert as user position tick
if (o_.tick > type(int).min) {
    unchecked {
        temp_ = o_.tick < 0 ? (uint(-o_.tick) << 1) : ((uint(o_.tick) << 1) | 1);
    }
} else {
    // if positionTick_ = type(int).min OR positionRawDebt_ == 0 then that means it's only supply position
    // (for case of positionRawDebt_ == 0, tick is set to type(int).min further up)
    temp_ = 0;
}

positionData[nftId_] =
    ((temp_ == 0) ? 1 : 0) | // setting if supply only position (1) or not (first bit)
    (temp_ << 1) |
    (o_.tickId << 21) |
    (o_.colRaw.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 45) |
    // dust debt is rounded down because user debt = debt - dustDebt. rounding up would mean we reduce user debt
    (o_.dustDebtRaw.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 109);
```

The first part sets the `temp` field to zero, as the user does not have any debt. Also, although we don't know what tick means yet, the user gets the default `type(int).min` value.

After this, `positionData` is updated. Position data is another bitmap that has this structure:

```solidity
/// position index => position data uint
/// if the entire variable is 0 (meaning not initialized) at the start that means no position at all
/// First 1 bit => 0 => position type (0 => borrow position; 1 => supply position)
/// Next 1 bit => 1 => sign of user's tick (0 => negative; 1 => positive)
/// Next 19 bits => 2-20 => absolute value of user's tick
/// Next 24 bits => 21-44 => user's tick's id
/// Below we are storing user's collateral & not debt, because the position can also be only collateral with no tick but it can never be only debt
/// Next 64 bits => 45-108 => user's supply amount. Debt will be calculated through supply & ratio.
/// Next 64 bits => 109-172 => user's dust debt amount. User's net debt = total debt - dust amount. Total debt is calculated through supply & ratio
/// User won't pay any extra interest on dust debt & hence we will not show it as a debt on UI. For user's there's no dust.
mapping(uint256 => uint256) internal positionData;
```

Only 2 fields updated are relevant for us at the moment:

1. Flag to set position type (supply only or not)

    ```solidity
    ((temp_ == 0) ? 1 : 0) | // setting if supply only position (1) or not (first bit)
    ```

2. Storing the `colRaw` in BigNumber representation:

    ```solidity
    (o_.colRaw.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 45) |
    ```

## 9. Check withdrawal gap

```solidity
// Withdrawal gap to make sure there's always liquidity for liquidation
// For example if withdrawal allowance is 15% on liquidity then we can limit operate's withdrawal allowance to 10%
// this will allow liquidate function to get extra 5% buffer for potential liquidations.
if (newCol_ < 0) {
    // extracting withdrawal gap which is in 0.1% precision.
    temp_ = (o_.vaultVariables2 >> 62) & X10;
    if (temp_ > 0) {
        // fetching user's supply slot data
        o_.userSupplyLiquidityData = LIQUIDITY.readFromStorage(LIQUIDITY_USER_SUPPLY_SLOT);

        // converting current user's supply from big number to normal
        temp2_ = (o_.userSupplyLiquidityData >> LiquiditySlotsLink.BITS_USER_SUPPLY_AMOUNT) & X64;
        temp2_ = (temp2_ >> 8) << (temp2_ & X8);

        // fetching liquidity's withdrawal limit
        temp3_ = int(LiquidityCalcs.calcWithdrawalLimitBeforeOperate(o_.userSupplyLiquidityData, temp2_));

        // max the number could go is vault's supply * 1000. Overflowing is almost impossible.
        unchecked {
            // (liquidityUserSupply - withdrawalGap - liquidityWithdrawaLimit) should NOT be less than user's withdrawal
            if (
                (temp3_ > 0) &&
                (((int(temp2_ * (1000 - temp_)) / 1000)) - temp3_) <
                (((-newCol_) * int(EXCHANGE_PRICES_PRECISION)) / int(o_.liquidityExPrice))
            ) {
                revert FluidVaultError(ErrorTypes.Vault__WithdrawMoreThanOperateLimit);
            }
        }
    }
}
```

The first comment in the code clearly explains what's going on here. We know from before that the vault protocol is a user of the liquidity layer, having its own withdrawal limits.

From that, it would be obvious that max withdrawal possible is `liquidityUserSupply - liquidityWithdrawalLimit`, but there is one extra caveat, hence why Fluid introduces the concept of `withdrawal gap`.

Withdrawal gap additionally restricts max withdrawals from the vault, to ensure that there is always some percentage of liquidity available for potential liquidations. This means that the previously calculated withdrawal can be lowered to let's say 90% of `liquidityUserSupply - liquidityWithdrawalLimit` to leave some buffer liquidity left in the vault.

### Implementation

The code first loads the current vault supply amount to the `temp2_` variable, and then it calculates the withdrawal limit by vault data and current supply. This was already discussed here: [LiquidityCalcs.calcWithdrawalLimitBeforeOperate]({{< ref "../3-liquidity-layer/7-supply-or-withdraw/withdrawal-limits.md" >}}).

After that, it checks that the user's withdrawal is less than or equal to `(liquidityUserSupply - withdrawalGap - liquidityWithdrawalLimit)`.

## 10. Execute operation

```solidity
// execute actions at Liquidity: deposit & payback is first and then withdraw & borrow
if (newCol_ > 0) {
    // deposit
    LIQUIDITY.operate{ value: SUPPLY_TOKEN == NATIVE_TOKEN ? msg.value : 0 }(
        SUPPLY_TOKEN,
        newCol_,
        0,
        address(0),
        address(0),
        abi.encode(msg.sender)
    );
}
if (newDebt_ < 0) {
    ...
}
if (newCol_ < 0) {
    // withdraw
    LIQUIDITY.operate(SUPPLY_TOKEN, newCol_, 0, to_, address(0), new bytes(0));
}
if (newDebt_ > 0) {
    // borrow
    ...
}
```

Finally, we call `operate` from the liquidity layer, to either supply or withdraw tokens.

## 11. Update vault variables

```solidity
{
    // Updating vault variables on storage

    // Calculating new total collateral & total debt.
    temp_ = (vaultVariables_ >> 82) & X64;
    temp_ = ((temp_ >> 8) << (temp_ & X8)) + o_.colRaw - o_.oldColRaw;
    temp2_ = (vaultVariables_ >> 146) & X64;
    temp2_ = ((temp2_ >> 8) << (temp2_ & X8)) + (o_.debtRaw - o_.dustDebtRaw) - o_.oldNetDebtRaw;
    // Updating vault variables on storage. This will also reentrancy 0 back again
    // Converting total supply & total borrow in 64 bits (56 | 8) bignumber
    vaultVariables =
        (vaultVariables_ & 0xfffffffffffc00000000000000000000000000000003ffffffffffffffffffff) |
        (temp_.toBigNumber(56, 8, BigMathMinified.ROUND_DOWN) << 82) | // total supply
        (temp2_.toBigNumber(56, 8, BigMathMinified.ROUND_UP) << 146); // total borrow
}

emit LogOperate(msg.sender, nftId_, newCol_, newDebt_, to_);

return (nftId_, newCol_, newDebt_);
```

At the end, we update total vault supply or borrow amounts, emit the main `LogOperate` event, and return from the function with the newly created `nftId_` and new amount for collateral and debt.