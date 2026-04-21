# cUSD Mechanics

cUSD is a US dollar-backed stablecoin redeemable against reserve assets. Users can interact with cUSD via three operations:&#x20;

• **Mint**: Deposit reserve assets to mint cUSD at oracle value\
• **Burn**: Redeem cUSD for reserve assets at the asset price of the highest deviation\
• **Redeem**: Multi-collateral redemption to ensure peg stability&#x20;

## Key highlights

### Peg Stability Module (PSM)

The cUSD contract acts as a Peg Stability Module (PSM), providing mechanisms for users to mint, burn, and redeem cUSD directly against a diversified basket of whitelisted backing assets (e.g., USDC, pyUSD, BENJI, BUIDL). The PSM’s primary objective is to ensure that cUSD’s market price remains close to $1, while managing the risk and distribution of its underlying collateral.&#x20;

Effectively, this design ensures that cUSD is always liquid and can be swapped for any of the underlying assets at transparent, market-based rates.&#x20;

### **Redeem:** Addressing Depegs

When a user redeems cUSD, the system calculates the output as a proportional basket of the underlying assets, based on their current weights and market prices. This prevents “last man standing” problem, where late redeemers could be left with only depegged assets.

The loss from a depeg of any of the underlying assets is socialized: the system disincentives users to burn once the distribution of assets deviates from the optimal level.

### **Fractional Reserves**

When deposited assets are not being actively borrowed, the idle capital earns rewards via revenue sharing of the underlying asset (MMFs) or through integration with crypto lending markets such as Aave.&#x20;

Each asset in the Fractional Reserve Vault accrues interest until they are divested when needed for withdrawing, redeeming, or borrowing.&#x20;

