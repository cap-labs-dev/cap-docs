# Oracles

The Oracle module in CAP is responsible for providing reliable, up-to-date price and rate data to the protocol. It acts as the backbone for all value calculations, including minting, burning, borrowing, and liquidation processes.&#x20;

## Oracle Data Sources

The protocol leverages multiple oracle sources for different types of data:

* [**RedStone Oracles**](https://www.redstone.finance/): Primary source for reserve asset pricing ([USDC, USDT, pyUSD & cUSD](https://app.redstone.finance/app/feeds/?page=1\&sortBy=popularity\&sortDesc=false\&perPage=32\&networks=1))
* [**Chainlink Oracles**](https://chain.link/): Used for delegation asset pricing (wstETH, wBTC) on shared security side
* [**Cap Token Adapter**](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/libraries/CapTokenAdapter.sol): Calculates weighted average of underlying basket for cUSD pricing
* [**Staked Cap Adapter**](https://github.com/cap-labs-dev/cap-contracts/blob/1064b6a969d55c822dcf0b2c4b733ceb4118737e/contracts/oracle/libraries/StakedCapAdapter.sol): Accounts for accrued yield and cUSD price for stcUSD pricing
* [**Aave Adapter**](https://app.gitbook.com/u/YhHCfPpqr7SuXN3B26Zp7K85kgr1): Fetches current borrow rates from external markets
* [**Vault Adapter**](https://app.gitbook.com/u/YhHCfPpqr7SuXN3B26Zp7K85kgr1): Calculates utilization-based interest rates

For function signatures, parameters, and data structures, see the [Oracle Contract Reference](../developers/contracts/oracle.md).
