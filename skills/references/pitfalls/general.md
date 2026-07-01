# T2 Verification Pitfalls

Consolidated list of known traps encountered during on-chain verification. Every item below has caused a real hallucination or misclassification in a past investigation.

## 1. Etherscan Event-Log False-Positive ETH Values

Etherscan event-log aggregator shows huge ETH values (e.g., 29M ETH) for a theft address, but actual `eth_getBalance` delta is near zero. The "noise" is an internal `WETH.withdraw` / FeeCollector unwrapper sub-call.

**Detection:** Compute `eth_getBalance(addr, latest) - eth_getBalance(addr, preTheftBlock)`. If event-log sum > 10x balance delta, it's a false-positive artifact. Check outer-tx `value` — if 0, the inner call value is not a real transfer.

## 2. Sweep txs Misidentified as Exploit txs

Sweep transactions (batch token transfers from contracts) can look like exploit txs. Always check the `from` address matches the reported attacker role, not just the selector.

## 3. Permit2 Solver Swaps Producing Fake Attacker Activity

Permit2 solver swaps produce `tokentx` entries that look like attacker activity. Filter by actual attacker address, not by token contract address.

## 4. Hallucinated 0-tx Consolidation EOAs

Always verify `eth_getTransactionCount` is non-zero for any address claimed as consolidation. Past reports fabricated consolidation addresses with zero on-chain history.

## 5. Wrong accountType

Run `eth_getCode` on every address. Source articles often misclassify EIP-7702 smart accounts as EOA. Code length reference:

| code_len | Type |
|----------|------|
| 2 (`"0x"`) | EOA |
| ~48 | EIP-7702 smart account |
| 344 | Gnosis Safe v1.3.0 proxy |
| 4302 | OpenZeppelin ProxyAdmin |

## 6. attackTime UTC Drift

Always use `block.timestamp` from `eth_getBlockByNumber`, not blog-post timestamps. Blog times are often rounded or in local timezone.

## 7. Failed tx (status=0x0) Included as SUCCESS

Attacker may submit multiple attempts at the same operation; some revert (status=0x0). Always check `eth_getTransactionReceipt.status`. If status=0x0, either replace with a successful tx of the same role or annotate as "attempted but reverted".

## 8. Fabricated tx Hash with Hallucinated Suffix

LLM can generate tx hashes with correct first ~20 chars (matching a real Etherscan result) but hallucinated suffix. Always verify every tx hash via `eth_getTransactionByHash`. If "not found", re-query the original data source for the correct hash.

**Pattern of hallucinated suffix:** smooth sequential hex like `4e3a4f5e6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a`. Real tx hashes have high entropy throughout.

## 9. Literal Newline in JSON String After Patch

Patching a description field may introduce a literal `\n` (newline character) inside a JSON string value instead of escaped `\\n`. Python's `json.load()` rejects this with `JSONDecodeError: Invalid control character`.

**Fix:** After ANY patch to a JSON file, re-validate with `json.load()` independently. The patch tool's built-in lint may not catch control characters inside string values.

## 10. Fake / Zero-Activity Victim Addresses

Past reports listed victim addresses that were actually EOAs with zero transaction history. Always verify: `eth_getTransactionCount` > 0 AND `eth_getCode` confirms contract status for any address claimed as a protocol contract.

## 11. Deploy Tx ↔ Contract Address Mismatch

Report claims tx A deployed contract X, but `getcontractcreation` shows tx B. Use `getcontractcreation` API or `eth_getTransactionReceipt` to verify all deploy→contract mappings.

## 12. Missing Post-Exploit Transactions

Reports often omit consolidation txs (DEX swaps, bridges, CEX deposits) that happen after the main exploit window. Always examine the full attacker EOA txlist up to 12+ hours after the last exploit tx.

## 13. Etherscan tokentx Truncated Sender

Etherscan `tokentx` shows the ERC-20 source contract address as `from`, not the actual outer tx submitter. Always resolve the full address via `eth_getTransactionByHash` before recording.

## 14. BSC V2 API Requires Paid Plan

Etherscan V2 with `chainid=56` returns `"Free API access is not supported for this chain"` with free key. Use public BSC RPC nodes instead (see `references/onchain-verification.md`).

## 15. Arbitrum Block Timestamps vs Block Numbers

Arbitrum blocks are ~250ms apart. Don't compute attack duration from block number deltas — use `eth_getBlockByNumber` for actual timestamps.

## 16. Etherscan API Key Rate-Limiting During Extended Investigations

After ~50+ API calls across Phases 1-3, the Etherscan V2 API key can get blocked with `"Too many invalid api key attempts, please try again later"`. This is NOT an invalid key error — it's a rate-limit lockout from too many calls in a short window.

**Detection:** API returns `{"status":"0","message":"NOTOK","result":"Too many invalid api key attempts..."}`. All subsequent calls return the same error regardless of endpoint.

**Fix:** Follow the **API Key Resolution Protocol** Fallback Paths in SKILL.md. Switch to public RPC endpoints (`https://1rpc.io/eth` for Ethereum) or Blockscout API (`https://eth.blockscout.com/api/v2/`). All JSON-RPC methods (`eth_getTransactionByHash`, `eth_getTransactionReceipt`, `eth_getCode`, `eth_getBlockByNumber`) work without an API key. Blockscout provides tx lists and token transfers without a key.

**Prevention:** Cache Etherscan API responses to disk during Phase 1. Reuse cached data in Phase 4 instead of re-querying. Reserve Etherscan API for queries that public RPC cannot handle (txlist, tokentx, contract creation). If no Etherscan key is available at all, Blockscout API can serve as the primary data source for ETH — it provides tx lists, token transfers, and contract source without authentication.

## 17. EIP-7702 accountType Timing — Always Check at Attack Block

An attacker's account type can change between the attack block and `latest`. Specifically, an attacker may operate as a regular EOA during the attack and upgrade to EIP-7702 *after* the attack.

**Scenario:** `eth_getCode(attacker, "latest")` returned code_len=48 starting with `0xef0100` (EIP-7702). But `eth_getCode(attacker, hex(attackBlock))` returned empty (code_len=0) — the attacker was a **plain EOA** during the attack. The EIP-7702 delegation was set up post-attack.

**Fix:** Always run `eth_getCode` at BOTH `latest` AND `hex(attackBlock)`. The `accountType` in the report should reflect the state at the attack block, not at `latest`. If they differ, note the post-attack upgrade in the description.

**Note:** Not all public RPCs support historical `eth_getCode` at arbitrary block heights. `ethereum-rpc.publicnode.com` is documented as the best for this, but may timeout. `1rpc.io/eth` does support historical block queries.

## 18. Truncated tx Hashes from Etherscan API Display Output

When processing Etherscan `txlist` API results, the `hash` field in display output (e.g., terminal print) may be truncated. If you copy the truncated hash into the report, it will be a fabricated/invalid hash.

**Scenario:** Two funding tx hashes in the draft were reconstructed from truncated terminal display (`0xea1a30fcd6780b5020e8b7e6fda0...`). The actual on-chain hash was `0xea1a30fcd6780b502032a341212d5e1daf3c3211d08b25ffb930035a2ee70f8c` — completely different after the first 18 chars.

**Fix:** ALWAYS read the full `hash` field from the stored JSON API response file (e.g., `json.load(open('attacker_earliest.json'))['result'][0]['hash']`). Never reconstruct tx hashes from truncated display output. This is a variant of Pitfall #8 but for funding/setup txs rather than exploit txs.

## 19. Public RPC 429 Rate-Limiting

Public RPC endpoints (especially `1rpc.io/eth`) return HTTP 429 after rapid sequential calls (~8-10 calls without pause).

**Fix:** Add `time.sleep(1)` between RPC calls in verification scripts. For burst-heavy verification, use a retry wrapper with exponential backoff:

```python
def rpc_call(endpoint, method, params, retries=3):
    for attempt in range(retries):
        try:
            # ... make call ...
            return result
        except Exception as e:
            if attempt < retries - 1:
                time.sleep(2 ** attempt)
            else:
                raise
```

**Note:** `ethereum-rpc.publicnode.com` may timeout completely (not 429, but connection timeout). Have a fallback chain: `1rpc.io/eth` → `rpc.ankr.com/eth` → `eth.drpc.org`.

## 20. Silent `None` Return from RPC (Not 429, Not Timeout)

Public RPC endpoints (observed with `1rpc.io/eth`) can return `None` as the JSON-RPC `result` field for valid queries — specifically `eth_getCode` and `eth_getTransactionReceipt` — without raising an HTTP error or returning a 429. The call appears to succeed (HTTP 200, valid JSON), but `result` is `null`.

**Scenario:** During Taiko (2026-06-21) address verification, `eth_getCode(0x7506..., "latest")` returned `None` from `1rpc.io/eth`, causing `len(None)` → `0` and a misclassification as EOA or `RPC_FAILED`. A retry from `eth.drpc.org` returned the correct `"0x"` (len=2, EOA). Similarly, `eth_getTransactionReceipt` for two of four tx hashes returned `None` from `1rpc.io/eth` on first pass but succeeded on retry from the same or a different endpoint.

**Detection:** If `eth_getCode` or `eth_getTransactionReceipt` returns `None`/`null` as the result (not `"0x"`, not an error dict), treat it as a transient failure — NOT as "address is an EOA" or "tx does not exist". Retry from a different endpoint.

**Fix:** Check `result is not None` before checking `len(result)`. Use multi-endpoint rotation:

```python
def rpc_call_robust(method, params):
    endpoints = ["https://1rpc.io/eth", "https://rpc.ankr.com/eth", "https://eth.drpc.org"]
    for ep in endpoints:
        result, err = rpc_call_single(ep, method, params)
        if result is not None:
            return result
        time.sleep(1)
    return None  # truly failed after all endpoints
```

**Key distinction:** `"0x"` (len=2) is a valid EOA result. `None`/`null` (len=0 in Python) is a failed query. Always distinguish these two cases in classification logic.
