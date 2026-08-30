---
title: "Liquidate: Vault variables storage update"
weight: 43
layout: book
hiddenInHomeList: true
---

Finally, all that's left to do for this `liquidate` function is to update total vault supply and debt by lowering the liquidated amounts.

Note that `vaultVariables` will already contain updated branch data info after liquidation.

{{< figure src="/fluid-vault/vault/liq-final-update.png" alt="Vault protocol" width="80%" >}}

After we perform the update, we emit the `LogLiquidate` event with liquidated amounts:

```solidity
emit LogLiquidate(msg.sender, actualColAmt_, actualDebtAmt_, to_);
```