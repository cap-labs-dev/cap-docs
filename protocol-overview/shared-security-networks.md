# Shared Security Networks

Cap is a decentralized credit marketplace that leverages Shared Security Networks (SSNs) such as Symbiotic and EigenLayer to separate yield generation from risk. Instead of relying on centralized decision making to allocate capital, Cap empowers independent Borrowers to draw funds and deploy them into yield-generating strategies. Borrowers are not trusted by default; rather, they must secure economic guarantees from Underwriters on SSNs who lock up capital as performance bonds. Hence, SSNs regulate the permissionless collective of Borrowers that generate yield for Lenders.

The result is a trustless, modular stablecoin architecture where risk is underwritten by the market, rather than being absorbed by users.

Let's dive deeper into how Cap utilizes SSNs.

## 1. Productivity-based Incentives

Cap builds on the idea of Shared Security Networks as a trust marketplace, extending the common usage as Proof of Stake (PoS) networks to a productivity based platform.&#x20;

### Higher Borrower Agency

Traditionally, SSNs have focused on validation services. In these models, operators perform uniform validation tasks such as verifying signatures, and rewards are distributed passively and evenly amongst participants. Newer models, such as network wide insurance-based services, have been introduced, where Underwriters underwrite a generic event rather than individual Borrower risk. While serving a different usecase, these models also provide passive rewards.

Cap's approach to SSNs provides a more granular and productivity-based incentive structure. Cap enables Borrowers to take on more agency, such that they can be rewarded in proportion to their individual value creation. Borrowers participate in Cap based on their borrowing competitiveness, taking the delta between their returns and the hurdle rate. Accordingly, Underwriters opt in to a subset of the Borrowers by evaluating their strategies, rates and legal agreements.

### Customizable Rewards

Cap innovates reward mechanisms for Underwriters on SSNs. Underwriters are able to charge premiums based on the productivity and risk of the specific Borrowers they underwrite. The premium is paid out in blue chip assets like ETH and USD, rather than inflationary governance tokens or offchain points programs. This stands in stark contrast to the passive, network-wide reward structures of PoS services on SSNs.

### Defined Counterparty Risk

Cap's delegation model isolates risk, ensuring that liquidation events are tied directly to the Borrower in question, rather than affecting the entire network. As a result, contagion risk is minimized, and Underwriters gain greater control and visibility over their exposure.

More specifically, because all Borrowers in Cap are regulated financial entities, Underwriters can enter into legal agreements with their counterparties. Hence, the chance of an actual liquidation event is equivalent to these regulated entities filing for bankruptcy.

## 2. Credible Financial Guarantees

The key innovation in Cap is that stablecoin holders are fully protected from yield-related losses via credible financial guarantees. If a liquidation condition is triggered on a particular Borrower, the Underwriters backing that Borrower are liquidated. Rather than burning them, the liquidated funds are redistributed via the SSNs to cover the shortfall, ensuring cUSD remains fully collateralized at all times. Redistribution enables verifiable downside protection for end users via transparent code.

### Liquidation conditions

The liquidation conditions are automated, objective and verifiable onchain. Rather than relying on human judgement to penalize malicious behavior, Cap powers a robust, decentralized trust marketplace, where the recourse provision is fully transparent.&#x20;

Liquidation conditions depend on the ratio of two variables: the amount of collateral (delegations) and debt. Liquidation is deterministically triggered when the health factor of the Borrower drops below the liquidation threshold.&#x20;

Liquidation is executed instantly and permissionlessly. There are no delay periods nor veto committees for liquidations to occur, or for the liquidated funds to be redistributed. As a buffer, grace periods are introduced, where Borrowers are given a fixed time period to regain their position's health.

## 3. Unlocking Capital Efficiencies

Crypto lending markets require borrowers to overcollateralize their own position (post 1 USDC worth of ETH as collateral to borrow 0.7 USDC). While this design ensures the most verifiable form of credit, it is capital inefficient, especially when the borrowers themselves are the ones posting the collateral.&#x20;

Cap improves on credit in DeFi via SSNs. Cap is able to achieve _insured_ private credit via credit delegations backed by restaked crypto assets and off-chain legal agreements.

This model is possible because of the inherently different cost of capital between crypto-denominated delegation assets and the dollar-denominated reserve assets. There is a lack of use case for locked crypto assets — staked ETH on the Beacon chain can only be used as limited form of _trust._ As such, the cost of capital for using ETH as delegation is much cheaper than it would be on the dollar. The delegation model is not limited to ETH; rather, the market size of delegations easily extends to all crypto asset types such as BTC.
