# Protocol Overview

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption><p>High-level Overview of Cap's Design</p></figcaption></figure>

Cap is a credit marketplace where the functions of underwriting, borrowing, and lending are separated and enforced by code. Underwriters escrow collateral to backstop institutional Borrowers. Those Borrowers gain access to credit lines funded by Lenders depositing stablecoins. The Underwriter's escrowed collateral is the first-loss buffer: if a Borrower defaults, the collateral is liquidated before Lenders absorb any loss. Every guarantee, every collateral position, and every Borrower exposure is visible onchain in real time.
