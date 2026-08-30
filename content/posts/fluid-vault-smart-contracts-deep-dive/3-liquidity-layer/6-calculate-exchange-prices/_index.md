---
title: "Operate: Calculate exchange prices"
weight: 9
layout: book
hiddenInHomeList: true
---

So far, this whole function was pretty straightforward. We performed a few sanity checks and transferred the funds from the caller.

But now we are moving to more challenging parts.

This is the snippet of code we are looking at:

{{< figure src="/fluid-vault/liquidity-layer/6-calculate-exchange-prices.png" alt="Liquidity layer" width="100%" >}}

For now let's just focus on `1` and `3` as `2` will deserve dedicated page.

### 1. Load exchange prices
We load exchange prices and config into our helper struct that fights stack too deep errors.

```solidity
o_.exchangePricesAndConfig = _exchangePricesAndConfig[token_];
```

This config is stored inside the liquidity contract storage and is represented by this mapping:

```solidity
/// @dev exchange prices and token config per token: token -> exchange prices & config
/// First 16 bits =>   0- 15 => borrow rate (in 1e2: 100% = 10_000; 1% = 100 -> max value 65535)
/// Next  14 bits =>  16- 29 => fee on interest from borrowers to lenders (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383). configurable.
/// Next  14 bits =>  30- 43 => last stored utilization (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383)
/// Next  14 bits =>  44- 57 => update on storage threshold (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383). configurable.
/// Next  33 bits =>  58- 90 => last update timestamp (enough until 16 March 2242 -> max value 8589934591)
/// Next  64 bits =>  91-154 => supply exchange price (1e12 -> max value 18_446_744,073709551615)
/// Next  64 bits => 155-218 => borrow exchange price (1e12 -> max value 18_446_744,073709551615)
/// Next   1 bit  => 219-219 => if 0 then ratio is supplyInterestFree / supplyWithInterest else ratio is supplyWithInterest / supplyInterestFree
/// Next  14 bits => 220-233 => supplyRatio: supplyInterestFree / supplyWithInterest (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383)
/// Next   1 bit  => 234-234 => if 0 then ratio is borrowInterestFree / borrowWithInterest else ratio is borrowWithInterest / borrowInterestFree
/// Next  14 bits => 235-248 => borrowRatio: borrowInterestFree / borrowWithInterest (in 1e2: 100% = 10_000; 1% = 100 -> max value 16_383)
/// Next   1 bit  => 249-249 => flag for token uses config storage slot 2. (signals SLOAD for additional config slot is needed during execution)
/// Last   6 bits => 250-255 => empty for future use
///                             if more free bits are needed in the future, update on storage threshold bits could be reduced to 7 bits
///                             (can plan to add `MAX_TOKEN_CONFIG_UPDATE_THRESHOLD` but need to adjust more bits)
///                             if more bits absolutely needed then we can convert fee, utilization, update on storage threshold,
///                             supplyRatio & borrowRatio from 14 bits to 10bits (1023 max number) where 1000 = 100% & 1 = 0.1%
mapping(address => uint256) internal _exchangePricesAndConfig;
```

At the beginning of this liquidity module section, we said we will also be analyzing this variables file alongside the liquidity contract. Well, this is where all of the structs are defined, alongside this prices config: `contracts/liquidity/common/variables.sol`.

This mapping adapts the standard bitmap pattern of efficiently storing data inside one `uint256` value.

We see a lot of data here. Hopefully for us, Fluid did a nice job by documenting exactly how the data is formatted, so we will see how this is used in a second, where we will also explain the meaning behind the data.

### 3. Extract supply/borrow amount

In this part, we first load `totalAmounts` into our helper struct:

```solidity
o_.totalAmounts = _totalAmounts[token_];
```

`_totalAmounts` is yet another storage mapping of the liquidity contract that stores `4` numbers in one `uint256`.

```solidity
/// @dev total supply / borrow amounts for with / without interest per token: token -> amounts
/// First  64 bits =>   0- 63 => total supply with interest in raw (totalSupply = totalSupplyRaw * supplyExchangePrice); BigMath: 56 | 8
/// Next   64 bits =>  64-127 => total interest free supply in normal token amount (totalSupply = totalSupply); BigMath: 56 | 8
/// Next   64 bits => 128-191 => total borrow with interest in raw (totalBorrow = totalBorrowRaw * borrowExchangePrice); BigMath: 56 | 8
/// Next   64 bits => 192-255 => total interest free borrow in normal token amount (totalBorrow = totalBorrow); BigMath: 56 | 8
mapping(address => uint256) internal _totalAmounts;
```

We already covered how Fluid works with [BigNumbers]({{< ref "../../2-big-numbers.md" >}}), and this is the first example of one such case.

The code then simply extracts these 4 values from storage and converts them from the compacted state to the final representation.

First, we take the first 64 bits, which represent `totalSupplyRaw`. We do a standard bitwise `AND` to isolate the bits:

```solidity
uint256 internal constant X64 = 0xffffffffffffffff;
memVar_ = o_.totalAmounts & X64;
```

Then we convert this `totalSupplyRaw` from `BigMath: 56 | 8` format to the final representation:

```solidity
uint256 internal constant DEFAULT_EXPONENT_SIZE = 8;
uint256 internal constant DEFAULT_EXPONENT_MASK = 0xFF;

o_.supplyRawInterest = (memVar_ >> DEFAULT_EXPONENT_SIZE) << (memVar_ & DEFAULT_EXPONENT_MASK);
```

This last expression can be explained with an example. If this was our stored `supplyRawInterest`:

```solidity
1000010101001011010000001111101111001010011010000000001100011011
<---------------------56bits---------------------------><8bits->
```

When we perform `(memVar_ >> DEFAULT_EXPONENT_SIZE)`, we are left with:

```solidity
10000101010010110100000011111011110010100110100000000011
<---------------------56bits--------------------------->
```

Then we calculate `(memVar_ & DEFAULT_EXPONENT_MASK)`, which just isolates the value of the `exponent`.

In this case, it is `00011011` = `27`.

So, combining everything together, we get the final representation that is stored in `o_.supplyRawInterest`:

```solidity
10000101010010110100000011111011110010100110100000000011000000000000000000000000000
<---------------------56bits---------------------------><----------27bits--------->
```

The same logic then applies for other values as well, where we additionally shift the value from the bitmap `uint256` value if needed:

```solidity
// TotalAmounts
uint256 internal constant BITS_TOTAL_AMOUNTS_SUPPLY_WITH_INTEREST = 0;
uint256 internal constant BITS_TOTAL_AMOUNTS_SUPPLY_INTEREST_FREE = 64;
uint256 internal constant BITS_TOTAL_AMOUNTS_BORROW_WITH_INTEREST = 128;
uint256 internal constant BITS_TOTAL_AMOUNTS_BORROW_INTEREST_FREE = 192;

memVar_ = (o_.totalAmounts >> LiquiditySlotsLink.BITS_TOTAL_AMOUNTS_SUPPLY_INTEREST_FREE) & X64;
o_.supplyInterestFree = (memVar_ >> DEFAULT_EXPONENT_SIZE) << (memVar_ & DEFAULT_EXPONENT_MASK);
memVar_ = (o_.totalAmounts >> LiquiditySlotsLink.BITS_TOTAL_AMOUNTS_BORROW_WITH_INTEREST) & X64;
o_.borrowRawInterest = (memVar_ >> DEFAULT_EXPONENT_SIZE) << (memVar_ & DEFAULT_EXPONENT_MASK);
// no & mask needed for borrow interest free as it occupies the last bits in the storage slot
memVar_ = (o_.totalAmounts >> LiquiditySlotsLink.BITS_TOTAL_AMOUNTS_BORROW_INTEREST_FREE);
o_.borrowInterestFree = (memVar_ >> DEFAULT_EXPONENT_SIZE) << (memVar_ & DEFAULT_EXPONENT_MASK);
```

So nothing new here, we just extracted the data. But we skipped part `2` and didn't talk about the semantics of these values, and why we have separate interest and interest-free values.

This is the topic we analyze on the next page.