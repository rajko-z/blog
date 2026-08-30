---
title: "BigMath Library"
weight: 2
layout: book
hiddenInHomeList: true
---

Before we dive into main flows, it's worth exploring how Fluid stores and works with numbers.  
In order to compress numbers, Fluid uses a custom-made BigMath library that stores numbers with a coefficient and exponent.

```sol
number = coefficient << exponent
```

Let's take an example from the Fluid codebase for the decimal number `5035703444687813576399599`.
Storing that in binary we get:
```sol
10000101010010110100000011111011110010100110100000000011100101001101001101011101111
```

Now if we take a coefficient of `32` bits and an exponent of `8` bits, the same number gets represented like:
```sol
10000101010010110100000011111011000000000000000000000000000000000000000000000000000
<-----------32bits-------------><---------------------51bits---------------------->
```

So we keep the first `32` bits of precision and sacrifice the rest.  
To have the final compact form, the exponent which represents the number of filled zeros will be written in binary form (`51 = 00110011`):

```sol
1000010101001011010000001111101100110011
<------------32bits------------><8bits->
```

So we just went from storing this initial `83`-bit number to just a `40`-bit number. If you would have stored this number initially in a separate `bytes32` slot, you would get more than `6x` lower storage usage `(256/40)`.

Now, as with anything in life, there are tradeoffs. You lose precision. Let's go back to the initial example and see how much precision we lost. We started with `5035703444687813576399599` and ended up with:

```sol
>>> int("10000101010010110100000011111011000000000000000000000000000000000000000000000000000", 2)
5035703442907428892442624
```

If we look at these values as standard `18`-decimal token amounts, the difference is around `0.00178` tokens. For `ETH` priced at `$4k`, this is around `$7`, which is not negligible for a financial protocol.

```sol
>>> 5035703444687813576399599 - 5035703442907428892442624
1780384683956975
>>> 1780384683956975 / 10**18
0.001780384683956975
>>> 0.001780384683956975 * 4000
7.1215387358279
```

This is the reason that we need to find good enough precision so that this difference becomes negligible while still being able to compress data.  

That's why Fluid chooses **`56`** bits for the coefficient and **`8`** bits for the exponent.
You may be wondering why `8` bits are used for the exponent. Well, if you take a large enough number, let's say `type(uint256).max`, you will have `56` bits for the coefficient and `200` zeroes for the exponent, so storing `200` would require `8` bits (`11001000`).

Rerunning the same example from above with new values:

```sol
10000101010010110100000011111011110010100110100000000011000000000000000000000000000
<---------------------56bits---------------------------><----------27bits--------->
```

Writing the exponent in binary we get the final form:

```sol
1000010101001011010000001111101111001010011010000000001100011011
<---------------------56bits---------------------------><8bits->
```

Checking the difference from the initial value:

```sol
>>> int("10000101010010110100000011111011110010100110100000000011000000000000000000000000000", 2)
5035703444687813498372096
>>> 5035703444687813576399599 - 5035703444687813498372096
78027503
>>> 78027503 / 10**18
7.8027503e-11
>>> 7.8027503e-11 * 4000
3.12110012e-07
```

We clearly see now that the difference is negligible this time, and we also used `4x` less storage than storing a regular `uint256` number.

It is also important to note that Fluid performs rounding to be on the more conservative side regarding supply and borrow, which will be seen across the codebase to additionally mitigate any potential edge cases with dust values.  

This representation gives Fluid a few good properties:

- Ability to store `4` numbers in only `1` slot
- Ability to represent numbers up to `2**56` (`72057594037927936`) with no precision loss. This basically means that storing amounts for lower token decimals like USDC would hardly exceed this limit
- Ability to store large token amounts with negligible precision sacrifice