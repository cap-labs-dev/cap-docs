# Overview

Cap consists of the six main modules:

1. [**Vault**](vault/): Stores reserve assets and issues cUSD tokens, providing liquidity to the Lender
2. [**Lender**](lender/): Handles borrowing, repayment, liquidation, and interest calculations
3. [**Fee Auction**](fee-auction.md): Converts generated yield and fees to cUSD via a Dutch auction
4. [**Delegation**](delegation/): Connects to Shared Security Networks for delegated collateral, rewards, and liquidation
5. [**Oracles**](oracles.md): Price oracles for asset valuation and rate oracles for interest calculations
6. [**Access Controls**](access-controls.md): Function-level granular access controls across all protocol operations
