# Contract Reference

Technical reference for all Cap protocol smart contracts. Each page covers the contract's mechanics, full function signatures, data structures, and usage examples.

| Module | Description |
|--------|-------------|
| [Vault](vault.md) | cUSD backing asset vault — mint, burn, redeem, utilization tracking |
| [Minter](minter.md) | Dynamic mint/burn fee model, deposit caps, and asset whitelisting |
| [Fractional Reserve](fractional-reserve.md) | ERC4626 yield strategy integration — invest, divest, harvest |
| [Lender](lender.md) | Operator borrowing, two-stream interest accrual, and liquidations |
| [Delegation](delegation.md) | Slashable collateral tracking across EigenLayer and Symbiotic |
| [Fee Auction](fee-auction.md) | Dutch auction distributing protocol yield to stcUSD holders |
| [Oracle](oracle.md) | Price and rate oracle with pluggable adapter architecture |
| [Access Controls](access-controls.md) | Per-function role-based permissioning |
| [Tokens](tokens.md) | cUSD (CapToken), stcUSD (StakedCap), and L2Token (OFT) |
