**NOTE**

#### Please enable "Allow edits by maintainers" while putting up the PR.

---

##### Name (to be shown on DefiLlama):
Acre

##### Twitter Link:
https://x.com/AcreBTC

##### List of audit links if any:
acreBTC:
- [Côme du Crest - Acre Audit (Aug 2025)](https://drive.google.com/file/d/1dbK5gCyRQURiPJloJXZOTNLaowtaeMip/view) — acreBTC vault, MidasAllocator, WithdrawalQueue, AcreAdapter (Midas), BitcoinDepositorV2, BitcoinRedeemerV2, stBTC migration. Found and fixed 2 High, 1 Medium severity issues.

stBTC (legacy):
- [Thesis Defense - Acre Smart Contracts (May 2024)](https://github.com/Thesis-Defense/Security-Audit-Reports/blob/main/PDFs/240517_Thesis_Defense-Acre_Smart_Contracts_Security_Audit_Report.pdf) — stBTC vault, MezoAllocator, BitcoinDepositor, BitcoinRedeemer
- [Thesis Defense - Mezo-Acre stBTC (Aug 2024)](https://github.com/Thesis-Defense/Security-Audit-Reports/blob/main/PDFs/240808_Thesis_Defense-Mezo-Acre_stBTC_Smart_Contracts_Security_Audit_Report.pdf) — stBTC integration with Mezo Portal
- [Immunefi Boost Audit Competition (Aug-Sep 2024)](https://immunefi.com/audit-competition/boost-acre/scope/) — Competitive security review of V1 stBTC contracts



##### Website Link:
https://acre.fi

##### Logo (High resolution, will be shown with rounded borders):
<!-- Paste Media-Kit/PNG/Acre-symbol-Red.png into this field when creating the PR on GitHub. It will auto-host as a github.com/user-attachments URL. -->

##### Current TVL:
~$5.30M (~78 tBTC)

##### Treasury Addresses (if the protocol has treasury)
N/A

##### Chain:
Ethereum

##### Coingecko ID (so your TVL can appear on Coingecko, leave empty if not listed): (https://api.coingecko.com/api/v3/coins/list)


##### Coinmarketcap ID (so your TVL can appear on Coinmarketcap, leave empty if not listed): (https://api.coinmarketcap.com/data-api/v3/map/all?listing_status=active,inactive,untracked&start=1&limit=10000)


##### Short Description (to be shown on DefiLlama):
Bitcoin yield protocol that allows BTC holders to earn yield through an ERC-4626 vault backed by tBTC.

##### Token address and ticker if any:
acreBTC — 0x19531C886339dd28b9923d903F6B235C45396ded (Ethereum)

##### Category (full list at https://defillama.com/categories) *Please choose only one:
Yield

##### Oracle Provider(s): Specify the oracle(s) used (e.g., Chainlink, Band, API3, TWAP, etc.):
Custom Midas oracle (CustomAggregatorV3CompatibleFeedGrowth) at 0xA537EF0343e83761ED42B8E017a1e495c9a189Ee

##### Implementation Details: Briefly describe how the oracle is integrated into your project:
The acreBTC vault's totalAssets() chains through MidasAllocator -> AcreAdapter -> CustomAggregatorV3CompatibleFeedGrowth oracle to reflect yield accrued in downstream Midas/Re7 strategies. Ankura Trust Company calls setRoundDataSafe() approximately weekly to update the base NAV; between updates, yield auto-interpolates via a growthApr parameter.

##### Documentation/Proof: Provide links to documentation or any other resources that verify the oracle's usage:
- Oracle contract: https://etherscan.io/address/0xA537EF0343e83761ED42B8E017a1e495c9a189Ee
- Acre docs: https://docs.acre.fi
- Contract addresses: https://docs.acre.fi/mainnet

##### forkedFrom (Does your project originate from another project):
N/A

##### methodology (what is being counted as tvl, how is tvl being calculated):
TVL is calculated by calling totalAssets() on the acreBTC ERC-4626 vault (0x19531C886339dd28b9923d903F6B235C45396ded), which returns the total tBTC backing all acreBTC shares. Marked doublecounted:true because the underlying tBTC is deployed into Aave V3 and Morpho via Re7 Labs strategies.

##### Github org/user (Optional, if your code is open source, we can track activity):
https://github.com/acre-btc

##### Does this project have a referral program?
No
