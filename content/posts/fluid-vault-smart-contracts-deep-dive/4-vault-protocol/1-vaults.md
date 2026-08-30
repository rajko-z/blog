---
title: "Vaults"
weight: 22
layout: book
hiddenInHomeList: true
---

Fluid has 4 types of vaults:

- T1 vaults -> 1 collateral token / 1 debt token
- T2 vaults -> 2 collateral tokens (smart collateral) / 1 debt token
- T3 vaults -> 1 collateral token / 2 debt tokens (smart debt)
- T4 vaults -> 2 collateral tokens (smart collateral) / 2 debt tokens (smart debt)

We are going to analyze only T1 vaults for this module, as that's enough to understand the core idea behind managing positions and the liquidation engine.

Every vault is a separate smart contract which has its own state and users.

Here we can see a subset of different T1 vaults in a table with different parameters. We can also see that every vault has a different unique ID:

{{< figure src="/fluid-vault/vault/vaults.png" alt="Vault protocol" width="80%" >}}

## Vaults creation

All of these vaults are created through the Fluid vault factory.

Here is the mainnet address of the factory: https://etherscan.io/address/0x324c5Dc1fC42c7a4D43d92df1eBA58a54d13Bf2d#code

Where we can see contract creation transactions for different types of vaults:

{{< figure src="/fluid-vault/vault/vault-creation.png" alt="Vault protocol" width="100%" >}}

Creation flow goes from the factory `deployVault` function:

{{< figure src="/fluid-vault/vault/deploy-vault-code.png" alt="Vault protocol" width="100%" >}}

We won't analyze every line, but we can see a few main parts with different colours.

First, in yellow, we see that vaults are tracked internally with the `totalVaults` variable. A new vault will simply start from the latest deployed one.

Next, in pink, we can see that the vault address can be determined from its id by calling the `getVaultAddress` function:

{{< figure src="/fluid-vault/vault/get-vault-address.png" alt="Vault protocol" width="100%" >}}

Next, in green, we see a delegate call to the deployment logic contract that has to give us deployment data.

If we peek further, we can see this function from the `vaultDeploymentLogic` contract, which is responsible for returning creation bytecode:

{{< figure src="/fluid-vault/vault/vault-t1-init-logic.png" alt="Vault protocol" width="100%" >}}

This function primarily sets default implementations, constant data like token data, and returns already prestored T1 vault creation bytecode.

After we have creation bytecode, we can perform the deploy call which is labeled red on the initial code snippet:

```solidity
if (!(success_ && vault_ == _deploy(abi.decode(data_, (bytes))) && isVault(vault_))) {
    revert FluidVaultError(ErrorTypes.VaultFactory__InvalidVaultAddress);
}
```

This internal `_deploy` call then simply deploys a new T1 vault:

{{< figure src="/fluid-vault/vault/deploy-internal.png" alt="Vault protocol" width="100%" >}}

## Vault methods

Vault has 2 main methods:

- operate
- liquidate

{{< figure src="/fluid-vault/vault/vault-main-methods-1.png" alt="Vault protocol" width="100%" >}}

These methods will be our focus, as there lies the liquidation engine logic, although we will also cover other important methods like `absorb`.

----

Just like the liquidity layer, the main function used for interaction with the vault is called `operate`.

Users can supply/borrow/withdraw and payback by calling `operate` with the right arguments.

Also, besides `operate`, liquidators, as the second actor type, can call `liquidate` to perform liquidations.

{{< figure src="/fluid-vault/vault/vault-methods.png" alt="Vault protocol" width="80%" >}}