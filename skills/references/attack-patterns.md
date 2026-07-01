# Attack Pattern Quick Reference

Concise summaries of common DeFi attack patterns. Each section lists the detection signal and verification approach.

## Key Compromise (ProxyAdmin Seizure)

Target: upgradeable proxy contracts with Gnosis Safe multisig governance.

**Pattern:**
1. Key theft (off-chain) → attacker steals enough Safe owner keys to cross threshold
2. ProxyAdmin transfer → Safe `execTransaction` calls `transferOwnership` on ProxyAdmin
3. Implementation upgrade → `upgradeAndCall(proxy, newImpl, data)` swaps to malicious impl
4. Drain/mint → malicious impl exposes drain function or unrestricted `mint()`

**Detection signals:**
- Safe `execTransaction` selector `0x6a761202`
- All threshold confirmations collapse to a single block = offline key theft fingerprint
- ProxyAdmin `owner()` changed (verify via `eth_call` with `0x8da5cb5b`)
- Implementation slot changed: `eth_getStorageAt(proxy, 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc, "latest")`
- Malicious impl has minimal code vs legitimate impl

## TEE / SGX Key Compromise

Target: TEE-based bridge verifiers (e.g., Taiko SGX).

**Pattern:**
1. Key leak → enclave signing key (RSA private key) found in public GitHub repo
2. Enclave signing → attacker signs own malicious SGX enclave with leaked key
3. Instance registration → `registerInstance()` passes DCAP verification
4. Forged proof → malicious SGX instance signs fake L2 block state proofs
5. `proveBlocks()` → bridge accepts forged proofs, marks fake blocks as "proven"
6. `processMessage()` / `retryMessage()` → vault/bridge releases L1 assets

**Detection signals:**
- `registerInstance` selector `0x09c5eabe` on SgxVerifier contracts
- `proveBlocks` selector `0x0432873c` on TaikoL1
- `InstanceAdded` events from SgxVerifier
- `BlockVerified` events from TaikoL1
- Verify leaked key exists: `openssl rsa -in enclave-key.pem -pubout`

## EIP-7702 Exploit Helper (Post-Pectra)

Active since Ethereum Pectra upgrade (2025-05-07). EOA temporarily inherits code from a delegated contract for one transaction.

**Detection recipe (run on every post-Pectra exploit):**
1. `eth_getCode(eoa, "latest")` → `"0x"` (looks like normal EOA after attack)
2. `eth_getCode(eoa, hex(attackBlock))` → if starts with `0xef0100`, next 40 hex = delegated contract address
3. `tx.to == tx.from` (EOA self-call with calldata) = EIP-7702 signature

**Three signals confirming EIP-7702:**
- `tx.to == tx.from` (EOA-to-EOA with calldata)
- Attacker EOA nonce increments by exactly 1 per attack tx
- `eth_getCode(eoa, attackBlock)` returns `0xef0100<delegated_addr>`

**Sybil helper pattern:** EIP-7702 delegated helper may spawn >100 sub-helper contracts per tx to act as fresh `msg.sender` on vulnerable protocol's reward path.

**Best RPC for EIP-7702 historical queries:** `https://1rpc.io/eth` — supports historical `eth_getCode` at arbitrary block heights. `ethereum-rpc.publicnode.com` is documented as supporting this but has been observed to timeout in practice; use as secondary fallback.

## Cross-Chain Bridge Funding

When attacker EOA receives first inbound ETH from a non-CEX, non-Tornado contract, it's likely a cross-chain bridge proxy.

**Known bridge proxies:**
- Bridgers (`0xB685760EBD368a891F27ae547391F4E2A289895b`) — Chinese-language attacker infrastructure
- Across Protocol SpokePool — legitimate, but common L2→L1 exit hop (selector `0x3ce33bff`)
- Stargate / LayerZero — `swap()` on Stargate Router

**Investigation protocol:**
1. Fetch first 5 inbound txs to attacker EOA
2. For each, trace `from` field. If contract, check if known bridge via selector decoding
3. Tag in `transactions[]` with `role: "funding"`
4. Add to `description`: "Attack funded via [BridgeName] proxy [0x...], [amount] ETH on [date]"

**Attribution signals:**
- Lazarus/DPRK: Tornado Cash, YoMix
- Chinese-language clusters: Bridgers bridge proxy
- Vietnamese/SEA: changeNOW, FixedFloat, Binance P2P

## Non-EVM P2P Attacks

For attacks on Monero/Zcash/Bisq/Haveno-style P2P DEXes with no on-chain transaction layer.

**Schema treatment:**
- `blockchain`: `["monero"]` or relevant chain
- `ecosystem`: `"other"` (not in vocab — add if needed)
- `language`: `["N/A"]` (bug is in P2P message handling, not smart contract)
- `category`: `["access_control", "business_logic"]` or `["forged_message"]`
- `transactions[]`: empty `[]` or synthetic P2P-event entries (e.g., `P2P-Tor-onion-event-YYYY-MM-DDTHH:MM:00Z`)
- `attackers[]`: use synthetic identifier (`onion-v3-address-banned-by-<protocol>-YYYYMMDD`), `accountType: "Unknown"`
- `victims[]`: aggregate identifier (`<protocol>-multisig-trade-deposits-YYYY-MM-DD`), `accountType: "Unknown"`

**T2 verification:** If structurally impossible (e.g., Monero), document explicitly in `description`.

**Root cause verification:** Fetch upstream protocol's GitHub patch commit. Read the diff to get the dev-facing root cause. This substitutes for on-chain verification.

**Price lookup:** CoinGecko historical API:
```bash
curl -sL "https://api.coingecko.com/api/v3/coins/monero/history?date=20-05-2026"
# Format: date=DD-MM-YYYY (NOT YYYY-MM-DD — wrong format returns 404)
```

## Flash Loan Attack

**Pattern:** Borrow large amount via flash loan → manipulate price oracle → exploit dependent protocol → repay loan in same tx.

**Detection:** `trace_transaction` shows Aave/Uniswap flash loan in first internal calls, followed by price manipulation calls, ending with repayment. Look for `tx.to` = flash loan provider, then series of DEX swaps.

## MEV Phishing / Honeypot Attack

Target: automated MEV sandwich/arbitrage bots that execute trades against arbitrary pools without trust verification.

**Pattern (two-tx attack):**
1. **Lure tx** — Attacker pre-deploys fake ERC-20 tokens (e.g., fWETH, fUSDC, fUSDT) and fake liquidity pools that appear profitable. The MEV bot's automated strategy detects the fake opportunity and executes a sandwich/arbitrage trade through the fake pools. During this trade, the bot's swap routing logic grants ERC-20 `approve()` calls to attacker-controlled helper contracts as part of normal DEX interaction. These approvals are **not consumed or revoked** within the same tx — they remain as open allowances.
2. **Drain tx** (separate block, 6-72 seconds later) — An EOA calls the attacker's trap contract, which uses `transferFrom()` with the open allowances to move the bot's legitimate token holdings (real WETH, USDC, USDT) to the attacker.

**Detection signals:**
- `approve()` events (signature `0x8c5be1e5ebec7d5bd14f71427d1e84f3dd0314c0f7b2291e5b200ac8c7c3b925`) in the lure tx, with the victim contract as `owner` and unknown addresses as `spender`
- `transferFrom()` events in the drain tx: `Transfer` events (signature `0xddf252ad...`) from victim to attacker, but the drain tx `from` is NOT the victim — it's a third-party EOA calling the attacker's trap contract
- Large number of identical-amount Transfer events in the drain tx (e.g., 16 x 92.16 WETH, 20 x 143,528 USDC) — the attacker splits the drain into per-allowance chunks
- Attacker setup begins days/weeks before (deploying fake tokens, fake pools, trap contracts)
- Fake token symbols typically prefix with "f" (fWETH, fUSDC, fUSDT, fCAP)

**Key verification steps:**
- Decode `Approval` events in the lure tx receipt to identify which spender addresses received allowances
- Verify the drain tx receipt: all `Transfer` events should have `from` = victim, `to` = attacker
- Count the exact number of transfers and sum amounts per token — these must match the report's `estimatedLoss.breakdown`
- Check attacker's tx history for setup txs (contract deployments, fake token creation) starting days/weeks before the attack

**Academic reference:** Sen Yang et al., "SKANF: Detecting MEV Bot Vulnerabilities via Symbolic Execution" (CCS 2026, https://arxiv.org/abs/2504.13398). The paper identified 1,030 vulnerable MEV bot contracts, 394 auto-exploitable, with $10.6M potential losses and 104 confirmed historical attacks totaling $2.76M.

**Schema mapping:**
- `category`: `["mev_honeypot", "business_logic"]` — `mev_honeypot` is the category for attacks where fake tokens/pools lure automated MEV bots; `business_logic` covers the bot's flawed allowance management. Do NOT use `mev_front_running` — that category is for attacks that USE front-running, not for attacks ON front-running bots.
- `victims[].role`: `"vulnerable_contract"` for the MEV bot contract, `"protocol"` for the operator EOA
- `attackers[].role`: `"primary_attacker"` for the EOA/EIP-7702 account, `"exploit_contract"` for trap/drain and helper contracts
- `transactions[].role`: `"exploit"` for both lure and drain txs, `"setup"` for fake token/pool deployment, `"funding"` for gas funding, `"consolidation"` for post-exploit WETH withdrawal, `"laundering"` for ETH distribution
