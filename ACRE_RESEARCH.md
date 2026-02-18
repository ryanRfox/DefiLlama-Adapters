# Acre Protocol — Deep Research Notes

Research compiled February 2026. Updated with dashboard address review.
All on-chain data verified against Ethereum mainnet.

---

## 1. V1 stBTC Contract

- **Address**: `0xdF217EFD8f3ecb5E837aedF203C28c1f06854017`
- **Status**: Deprecated. Migration to V2 (acreBTC) completed October 15, 2025.
- **Migration mechanism**: Built-in `startMigration(address)` / `migrateDeposit(address)` in stBTC.sol. Irreversible, per-user batch process.
- **Migration event**: 47 batch transfers executed via Safe multisig "Exec Transaction" on Oct 15, 2025.
- **Remaining balance**: ~0.01 tBTC (~$767) across 461 stBTC holders. Effectively dust from unmigrated users.
- **V1 allocator**: Mezo Allocator (`0xb90fdad3dfd180458d62cc6acedc983d78e20122`) — now inactive.
- **Conclusion**: V1 does not need to be included in the DefiLlama adapter. All active TVL is in V2.

---

## 2. Contract Architecture

### Core Acre Contracts (on-chain, documented at docs.acre.fi/mainnet)

| Contract | Address | Role |
|----------|---------|------|
| acreBTC Vault (V2) | `0x19531C886339dd28b9923d903F6B235C45396ded` | ERC-4626 vault, user-facing deposit contract |
| tBTC | `0x18084fbA666a33d37592fA2633fD49a74DD93a88` | Underlying asset |
| MidasAllocator | `0xD72b0C95398058345842499975171368d49659Bb` | Pulls idle tBTC from vault, routes to Midas |
| MidasAllocator impl | `0x8d23399cc87f2f83dc46d303326c634e4d7065b2` | Implementation behind proxy |
| WithdrawalQueue | `0xe7b8c14cA8Fb4f226C0A3e45e636b84809BB5D06` | Manages withdrawal requests (14-day cooldown) |
| BitcoinDepositorV2 | `0xe5F48D3d31baf15dfF89fb394F10A5362711c777` | Accepts BTC deposits via tBTC bridge |
| BitcoinRedeemerV2 | `0x42A5f91586DDf041A6084494B0b375CdA34d55e9` | Handles BTC redemptions |

### Midas Integration Contracts

| Contract | Address | Role |
|----------|---------|------|
| AcreAdapter | `0x6A6092d9c47A7E4C085f2ED9FD4a376124587Ae0` | Wraps Midas deposit/redeem for Acre. Called "BTC-Neutral Blended Yield Vault" on dashboard |
| Midas Deposit Vault | `0x52e808bD3496c69c705028a258aEe0a6E1a5b35D` | Midas vault that mints acremBTC1 shares |
| Midas Redemption Vault | `0x319a05E260acC2490768A726Ccfd341D4b3D5106` | Handles acremBTC1 → tBTC redemptions |
| acremBTC1 token | `0xc344db27feba7f0a881a50f0f702a525a44f2368` | Dedicated Midas mToken for Acre (NOT the public mBTC). Only 2 holders, ~80.42 supply |
| acremBTC1 oracle | `0xA537EF0343e83761ED42B8E017a1e495c9a189Ee` | CustomAggregatorV3CompatibleFeedGrowth — auto-interpolates yield between weekly updates |

**Important correction**: The MidasAllocator does NOT deposit into the public mBTC token (`0x007115...`). It deposits via the AcreAdapter into a dedicated Midas vault that mints acremBTC1 — a purpose-built token for Acre's allocation.

### Operational EOA Wallets (from dashboard at bitcoin.acre.fi/dashboard)

Source: `dapp/src/constants/transparency.ts` in the Acre repo.

| Label | Address | Holdings (Feb 2026) | Role |
|-------|---------|---------------------|------|
| Assets to be Deployed | `0xbA53a278D6d68c2C8F6d600eA93581a702A6006A` | ~1.18 tBTC (~$84K) | Midas `tokensReceiver` — staging wallet where tBTC lands after allocation, before Re7 deploys it |
| Onchain Wallets | `0x7C819F438250CD9331D3E8EA540B72B3FF702f07` | ~$3.45M (15.87 mRe7BTC, 1.29M mRe7YIELD, 14.23 tBTC, Aave V3 positions) | Re7 Labs strategy execution wallet — the active DeFi positions |
| Available Liquidity Buffer | `0x910CA844Fb578f670Ca5190c1cF4ab851155Bf99` | ~0.115 tBTC (~$8K) | Liquid tBTC reserve for fast withdrawals. Pre-approved for AcreBtcRedemptionVaultWithSwapper |

These are all plain EOAs (not contracts or multisigs). Likely managed via Fordefi MPC custody.

### Other Referenced Contracts

| Contract | Address | Role |
|----------|---------|------|
| Re7 tBTC Morpho Vault | `0x43fD147d5319B8Cf39a6e57143684Efca9CF3613` | MetaMorpho ERC-4626 vault for tBTC lending |
| Morpho Blue | `0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb` | Base lending protocol |
| Public Midas mBTC | `0x007115416ab6c266329a03b09a8aa39ac2ef7d9d` | Public mBTC token — NOT used in Acre's flow |

---

## 3. Downstream tBTC Flow (Corrected)

The capital chain from user deposit to yield generation:

```
User deposits tBTC
       ↓
  acreBTC Vault (0x19531C...)
       ↓  MidasAllocator.allocate()
  MidasAllocator (0xD72b0C...)
       ↓  deposit() on AcreAdapter
  AcreAdapter (0x6A6092...)
       ↓  depositInstant() on Midas Deposit Vault
  Midas Deposit Vault (0x52e808...)
       ↓  mints acremBTC1 → MidasAllocator
       ↓  forwards tBTC to tokensReceiver
  "Assets to be Deployed" EOA (0xbA53a278...)
       ↓  manual transfer by Re7/Midas team
  "Onchain Wallets" EOA (0x7C819F43...)
       ↓  active DeFi strategy execution
  ┌─────────────────────────────────┐
  │ Strategy positions:             │
  │ • tBTC supplied to Aave V3     │
  │ • USDC borrowed against tBTC   │
  │ • USDC → mRe7YIELD (Midas)    │
  │ • tBTC → mRe7BTC (Midas)      │
  │ • Swaps via KyberSwap          │
  └─────────────────────────────────┘
```

### How `allocate()` works

```solidity
function allocate() external onlyMaintainer {
    // Pull all idle tBTC from acreBTC vault
    tbtc.safeTransferFrom(address(acreVault), address(this),
        tbtc.balanceOf(address(acreVault)));
    uint256 idleAmount = tbtc.balanceOf(address(this));
    // Deposit into AcreAdapter (labeled "midasVault" in code)
    tbtc.forceApprove(address(midasVault), idleAmount);
    uint256 shares = midasVault.deposit(idleAmount, address(this));
    emit DepositAllocated(idleAmount, shares);
}
```

Note: `midasVault` in MidasAllocator code points to the AcreAdapter (`0x6A6092...`), not a Midas vault directly.

### How `totalAssets()` propagates yield

```solidity
// MidasAllocator
function totalAssets() external view returns (uint256) {
    return tbtc.balanceOf(address(this)) +
           midasVault.convertToAssets(vaultSharesToken.balanceOf(address(this)));
}
```

`vaultSharesToken` is the acremBTC1 token. `midasVault.convertToAssets()` calls the AcreAdapter which uses the Midas oracle price feed. As the oracle reports increasing acremBTC1 value (via `CustomAggregatorV3CompatibleFeedGrowth`), MidasAllocator's totalAssets() increases, which increases acreBTC vault's totalAssets(), which increases the acreBTC redemption ratio.

### Yield mechanism: passive, not discrete payments

There are **no fortnightly tBTC payments** from Re7 to Acre. The "fortnightly" timeline in Acre docs refers to the withdrawal redemption window (14-day cooldown for withdrawals).

No tBTC ever flows back from Re7 to the acreBTC vault as a "yield payment." Instead, yield is reflected as a number update in an oracle.

### How yield gets back to depositors (the full round-trip)

**Outbound (deposit → strategy):**
```
Depositor sends tBTC → acreBTC vault → MidasAllocator → AcreAdapter
  → Midas Deposit Vault → tokensReceiver EOA → Re7 Onchain Wallets EOA
  → Aave V3 (collateral) + borrow USDC → mRe7YIELD / mRe7BTC
```

**Yield accrual (no on-chain transfer — just a number update):**
1. Re7 earns yield from Aave lending interest, Morpho markets, mRe7YIELD strategies
2. Midas/Re7 converts yield earnings to a tBTC-equivalent value off-chain
3. Ankura Trust Company (independent auditor) verifies the NAV and calls `setRoundDataSafe()` on the acremBTC1 oracle (~weekly)
4. Between updates, the oracle contract auto-interpolates growth using a `growthApr` parameter baked into the `CustomAggregatorV3CompatibleFeedGrowth` contract
5. When anyone calls `convertToAssets(acremBTC1)` on the AcreAdapter, the oracle returns a higher tBTC value per acremBTC1
6. MidasAllocator's `totalAssets()` therefore returns a larger number
7. acreBTC vault's `totalAssets()` increases accordingly
8. Each acreBTC share is now redeemable for more tBTC than before

**Inbound (withdrawal):**
```
Depositor requests withdrawal → WithdrawalQueue (14-day cooldown)
  → acreBTC vault burns shares → MidasAllocator redeems acremBTC1
  → AcreAdapter → Midas Redemption Vault → Re7 unwinds positions
  → tBTC returns via tokensReceiver → depositor receives tBTC
  (or from Liquidity Buffer EOA for small/fast withdrawals)
```

**In plain English:** A depositor's BTC grows because an oracle says each acremBTC1 token is worth progressively more tBTC. The depositor never sees a tBTC deposit into their account — instead, when they withdraw, their fixed number of acreBTC shares converts to more tBTC than they put in. The vault is non-rebasing: your acreBTC balance stays the same, but each token is worth more.

No on-chain harvest or yield transfer transaction occurs. The only on-chain evidence of yield is the `setRoundDataSafe` oracle updates.

---

## 4. Yield Analysis

- **Vault age**: ~3.5 months (as of Feb 2026)
- **TVL**: ~78.5 tBTC (~$5.14M at time of test)
- **Share price**: ~0.9999 (slightly below 1.0)
- **Realized APY**: ~0%
- **acremBTC1 price**: ~1.0314 BTC per acremBTC1 (3.14% yield accrued in Midas layer)

### Why acreBTC share price is still ~0.9999 despite acremBTC1 showing 3.14% gain

1. **20% protocol fee on yield** — cuts net yield significantly
2. **Midas deposit vault was paused Dec 18, 2025** — potential pipeline disruption
3. **Stream Finance / xUSD crisis (Nov 2025)** — Re7 disclosed $14.65M exposure; claimed no mBTC impact but may have reduced yield generation
4. **ERC-4626 rounding** — standard integer math rounding in favor of the vault
5. **Operational overhead** — gas costs for allocate/deallocate transactions
6. **Vault is young** — insufficient time for yield to compound meaningfully

### Advertised vs realized yield

- Acre's advertised rate was ~14% APY at launch, currently shows ~11% on homepage
- This was an estimated rate dependent on DeFi incentive programs
- CEO's longer-term target was "above 5%"
- The actual strategy is leveraged (Aave collateral → USDC borrow → yield farming), which amplifies both returns and risk

---

## 5. Fee Structure

| Fee | Amount | When |
|-----|--------|------|
| Exit fee (on-chain) | 0.25% | `exitFeeBasisPoints` in vault contract |
| tBTC bridge fee | 0.20% | Charged by Threshold tBTC bridge on redemption |
| **Total withdrawal cost** | **0.45%** | Combined |
| Protocol yield fee | 20% | Taken from earned yield before share price update |

No entry fee. No deposit fee.

---

## 6. Risk Observations

### Custody model

The tBTC leaves on-chain smart contracts at the Midas `tokensReceiver` step. From there, three plain EOAs manage the funds:
- `0xbA53a278...` — staging (receives tBTC from Midas deposit vault)
- `0x7C819F43...` — active positions (Aave, mRe7BTC, mRe7YIELD)
- `0x910CA844...` — liquidity buffer

Each EOA is controlled by a single private key (likely Fordefi MPC, but not on-chain multisig). A key compromise at any of these would expose the tBTC.

### Leverage risk

The "Onchain Wallets" EOA runs a leveraged strategy:
- ~$3.45M total portfolio
- ~$1.4M USDC borrowed from Aave V3 against tBTC collateral
- Liquidation risk if tBTC/USDC price drops sharply

### Oracle dependency

Yield propagation depends entirely on the acremBTC1 oracle (`CustomAggregatorV3CompatibleFeedGrowth`). If Ankura Trust stops updating, or if the `growthApr` parameter is set incorrectly, yield reporting could be inaccurate.

---

## 7. Dune Dashboard Feasibility

The discovery of the three operational EOA wallets and the acremBTC1 oracle significantly expands what's trackable on Dune compared to our initial assessment.

### What IS possible on Dune

**Vault activity (was already known):**
- Deposit/withdrawal event tracking — all `Deposit`, `Withdraw`, `Transfer` events on the acreBTC vault
- User-level position tracking — map depositors, amounts, timestamps
- `DepositAllocated` events from MidasAllocator — when tBTC is pushed downstream
- Token holder distribution — acreBTC balance snapshots over time
- tBTC bridge activity — cross-reference with tBTC mint/burn events

**Operational wallet flows (NEW — from dashboard addresses):**
- **Full tBTC flow tracing**: Every tBTC transfer between the vault, MidasAllocator, AcreAdapter, tokensReceiver (`0xbA53a278...`), Onchain Wallets (`0x7C819F43...`), and Liquidity Buffer (`0x910CA844...`) is a standard ERC-20 transfer, fully indexable
- **Strategy deployment timing**: Track when tBTC moves from "Assets to be Deployed" to "Onchain Wallets" — shows how quickly Re7 puts capital to work
- **Aave V3 positions**: The Onchain Wallets EOA's Aave supply/borrow events are fully on-chain. Track tBTC collateral supplied, USDC borrowed, health factor changes, and any liquidation events
- **mRe7BTC and mRe7YIELD holdings**: Token transfer events show when the strategy wallet buys/sells these Midas Re7 products
- **Liquidity buffer adequacy**: Track the buffer EOA's tBTC balance over time vs. Acre's stated 2% reserve policy
- **Idle capital detection**: Compare tokensReceiver balance to total vault TVL — shows what % of deposits are sitting undeployed

**Yield oracle tracking (NEW — from acremBTC1 discovery):**
- **`setRoundDataSafe` events on the acremBTC1 oracle** (`0xA537EF03...`): Each call updates the base price and growth rate. These are on-chain transactions with indexable calldata. You can reconstruct the full yield history from these events
- **Yield rate over time**: Extract the `growthApr` parameter from each oracle update to chart the annualized rate Re7/Ankura is reporting
- **Oracle staleness monitoring**: Track time between `setRoundDataSafe` calls — flag if updates stop or become irregular

### What is still NOT possible on Dune

- **Historical `totalAssets()` calls**: Dune cannot call view functions at arbitrary historical blocks. You can't chart the exact vault TVL by querying the contract. However, the oracle events above provide a close proxy.
- **Exact share price history**: Same limitation for `convertToAssets(1e18)`. But you can reconstruct it from oracle data + deposit/withdrawal events.
- **Off-chain yield conversion details**: The step where Re7 converts Aave/Morpho yield into the tBTC-equivalent NAV update happens off-chain. You can see the inputs (Aave positions) and the output (oracle update) but not the conversion itself.
- **Fordefi MPC signing decisions**: The EOA key management is entirely off-chain.

### Workarounds

**TVL history** (two approaches, combinable):
1. Sum `Deposit` minus `Withdraw` events over time → net deposits (accurate for a vault with ~0% yield so far)
2. Reconstruct from oracle: take deposit events + apply the `growthApr` from each oracle update period → approximate TVL including yield

**Share price history**: Reconstruct from `setRoundDataSafe` calldata. Each update encodes the new base price and growth rate. Replay the `CustomAggregatorV3CompatibleFeedGrowth` formula off-chain to get share price at any historical timestamp.

### Assessment: ~80% coverage now achievable

With the dashboard addresses, a Dune dashboard can cover most of what the community wants. The main remaining gap is the exact `totalAssets()` at historical blocks, but the oracle events provide a very close approximation. The Aave V3 positions on the Onchain Wallets EOA are the most valuable new data source — they show leverage ratio and risk exposure in real time, which was previously assumed to be invisible.

---

## 8. Community Tracking Ideas

### Quick wins (days)

1. **DefiLlama listing** — The adapter is ready. Once merged, Acre appears on DefiLlama with automatic TVL tracking.
2. **DeBank portfolio bookmarks** — The three operational EOAs are already viewable on DeBank. Community members can bookmark them today for real-time portfolio monitoring with no development effort.
3. **Etherscan watchlists** — Set up Etherscan alerts on the three EOAs + the acremBTC1 oracle. Get notified on every allocation, strategy trade, and yield oracle update.

### Medium effort (weeks)

4. **Dune dashboard — vault activity** — Track deposits, withdrawals, unique depositors, net flow over time, and holder distribution using acreBTC vault events.
5. **Dune dashboard — capital deployment** — Track tBTC flows across all 6 addresses in the pipeline (vault → MidasAllocator → AcreAdapter → tokensReceiver → Onchain Wallets → Liquidity Buffer). Show time-to-deploy (how long tBTC sits idle at each stage).
6. **Dune dashboard — yield tracking** — Index `setRoundDataSafe` calls on the acremBTC1 oracle. Chart the reported yield rate over time. Flag oracle staleness.
7. **Dune dashboard — risk monitoring** — Track the Onchain Wallets EOA's Aave V3 positions: collateral value, borrow amount, health factor, liquidation distance. Alert on leverage increases.
8. **Custom bot/script** — Call `totalAssets()`, `totalSupply()`, compute share price daily, store results. Feed to Telegram/Discord bot or simple web page. This fills the one gap Dune can't cover (historical view function calls).

### Aspirational (months)

9. **Full yield attribution** — Decompose returns: how much comes from Aave lending interest, mRe7BTC appreciation, mRe7YIELD, and KyberSwap trades. Requires parsing all txns from the Onchain Wallets EOA.
10. **Liquidity reserve compliance** — Compare the Liquidity Buffer EOA balance to Acre's stated 2% reserve policy. Track whether the buffer is being maintained as TVL grows.
11. **Withdrawal queue analytics** — Track WithdrawalQueue contract events: average wait time, queue depth, fulfillment rate, largest pending withdrawals.

---

## 9. DefiLlama Adapter Assessment

The adapter at `projects/acre/index.js` is **correct and ready to submit**.

```javascript
const { sumERC4626VaultsExport } = require("../helper/erc4626");

module.exports = {
  doublecounted: true,
  methodology:
    "TVL is calculated by calling totalAssets() on the acreBTC ERC-4626 vault, which returns the total tBTC backing all acreBTC shares.",
  ethereum: {
    tvl: sumERC4626VaultsExport({
      vaults: ["0x19531C886339dd28b9923d903F6B235C45396ded"],
      isOG4626: true,
    }),
  },
};
```

Why it works despite the complex downstream architecture:
- `totalAssets()` on the acreBTC vault calls MidasAllocator's `totalAssets()`
- MidasAllocator calls AcreAdapter's `convertToAssets()` on its acremBTC1 holdings
- The oracle-based pricing captures all yield accrued downstream
- The entire chain resolves to a single tBTC amount — which is what DefiLlama reports

The `doublecounted: true` flag is essential because the same tBTC is counted by:
- Acre (this adapter)
- Potentially Aave V3 (if Re7's collateral positions are tracked)
- Potentially Morpho (if Re7 strategies are tracked)

---

## 10. Key Takeaways

1. **The adapter is correct.** `sumERC4626VaultsExport` with `isOG4626: true` captures TVL through the full oracle chain.
2. **V1 stBTC is fully deprecated.** No need to track it.
3. **tBTC leaves on-chain custody** at the Midas `tokensReceiver` step and is managed by Re7 via EOA wallets, not smart contracts.
4. **Yield is passive, not periodic.** No fortnightly payments — acremBTC1 oracle auto-interpolates growth.
5. **The strategy is leveraged.** tBTC collateral → USDC borrow → yield farming. This amplifies returns and risk.
6. **Three operational EOAs** (Assets to Deploy, Onchain Wallets, Liquidity Buffer) hold the protocol's working capital. These are visible on the Acre dashboard for transparency.
7. **acremBTC1 ≠ public mBTC.** Acre has its own dedicated Midas mToken, separate from the public mBTC market.

---

## References

- Acre docs: https://docs.acre.fi
- Acre contracts: https://docs.acre.fi/mainnet
- Acre dApp source: https://github.com/acre-btc/acre
- Acre dashboard: https://bitcoin.acre.fi/dashboard
- acreBTC vault: https://etherscan.io/address/0x19531C886339dd28b9923d903F6B235C45396ded
- V1 stBTC: https://etherscan.io/address/0xdF217EFD8f3ecb5E837aedF203C28c1f06854017
- MidasAllocator: https://etherscan.io/address/0xD72b0C95398058345842499975171368d49659Bb
- AcreAdapter: https://etherscan.io/address/0x6A6092d9c47A7E4C085f2ED9FD4a376124587Ae0
- acremBTC1 token: https://etherscan.io/token/0xc344db27feba7f0a881a50f0f702a525a44f2368
- acremBTC1 oracle: https://etherscan.io/address/0xA537EF0343e83761ED42B8E017a1e495c9a189Ee
- Midas Deposit Vault: https://etherscan.io/address/0x52e808bD3496c69c705028a258aEe0a6E1a5b35D
- Midas Redemption Vault: https://etherscan.io/address/0x319a05E260acC2490768A726Ccfd341D4b3D5106
- Re7 tBTC Morpho Vault: https://etherscan.io/address/0x43fD147d5319B8Cf39a6e57143684Efca9CF3613
- Morpho Blue: https://etherscan.io/address/0xBBBBBbbBBb9cC5e90e3b3Af64bdAF62C37EEFFCb
- Midas docs: https://docs.midas.app
- Fordefi x Midas custody: https://www.fordefi.com/customer-stories/how-midas-brings-tokenized-investment-opportunities-on-chain-with-fordefis-defi-native-custody
- Blockworks coverage: https://blockworks.co/news/acre-btc-yield-ethereum
