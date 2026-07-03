# Parallel Dispatch Strategy

Cross-harness reference for parallelizing subagent work in the investigation workflow.

## When to Parallelize

Parallelize when you have N independent work items with no ordering dependency:

| Phase | Work Item Granularity | Example |
|-------|----------------------|---------|
| Phase 1: Intel Gathering | One subagent per reference URL | 15 URLs → up to 15 subagents |
| Phase 2: Autonomous Enrichment | One subagent per search query or discovered URL | 5 search queries + 3 discovered URLs |
| Phase 4: On-Chain Verification | One subagent per tx hash, per address, or per group | 4 txs + 5 addresses → up to 9 subagents |

## When NOT to Parallelize

- **Phase 3 (Schema Draft):** requires holistic view of all intel — must be done by the main agent
- **Phase 5 (Report Generation):** sequential file write
- **Phase 6 (Adversarial Validation):** requires full-report context — single agent re-reads entire JSON
- **Phase 7 (Finalize):** human interaction, inherently sequential
- Any step where work items have ordering dependencies or shared mutable state

## Harness Mapping

### Parallel Batch Dispatch

| Harness | Mechanism | Concurrency Limit | Notes |
|---------|-----------|-------------------|-------|
| Hermes | `delegate_task(tasks=[{goal, context, ...}, ...])` | `max_concurrent_children` (default 3) | Batch mode runs children in parallel; excess items queued automatically |
| Claude Code | Multiple `Task()` calls in a single response | No hard limit (practical: 3-5) | All tasks in one response run concurrently |
| Codex | Multiple `Task()` calls in a single response | No hard limit (practical: 3-5) | Same as Claude Code |
| OpenCode | No built-in subagent | N/A | Run all items sequentially in the main agent |

### No-Delegation Fallback

When the harness has no delegation capability (e.g., OpenCode), the main agent runs all work items sequentially itself. The structured return format and aggregation logic still apply — the agent processes each item one by one and collects results into the same aggregation structure.

## Concurrency Management

When the number of work items exceeds the harness concurrency limit:

1. **Group items into batches** of size = concurrency limit
2. **Dispatch batch 1**, wait for all to complete
3. **Dispatch batch 2**, wait for all to complete
4. **Repeat** until all items processed
5. **Aggregate** all results across batches

Example: 15 URLs with Hermes (max 3 concurrent):
- Batch 1: URLs 1-3 → 3 parallel subagents
- Batch 2: URLs 4-6 → 3 parallel subagents
- Batch 3: URLs 7-9 → 3 parallel subagents
- Batch 4: URLs 10-12 → 3 parallel subagents
- Batch 5: URLs 13-15 → 3 parallel subagents

For harnesses without a hard limit (Claude Code, Codex), dispatch all items at once but keep batches of 3-5 as a practical guideline to avoid overwhelming the model or triggering API rate limits.

## Structured Return Format

Each parallel subagent MUST return a structured result so the main agent can aggregate without re-reading raw output.

### Intel Gathering (Phase 1) — per-URL subagent return

```json
{
  "source_url": "<URL processed>",
  "source_type": "x_post | blog_post | post_mortem | alert | other",
  "entities": {
    "protocol_name": "<string or null>",
    "blockchains": ["<chain_id>"],
    "attacker_addresses": [{"address": "0x...", "blockchain": "<chain>"}],
    "victim_addresses": [{"address": "0x...", "blockchain": "<chain>"}],
    "tx_hashes": [{"txHash": "0x...", "blockchain": "<chain>", "role_guess": "<string>"}],
    "loss_figures": [{"amount": "<string>", "asset": "<string>", "usd_value": "<string>", "source_claim": "<string>"}],
    "attack_timeline": {"start_time": "<ISO8601 or null>", "end_time": "<ISO8601 or null>"},
    "root_cause": "<string or null>",
    "attack_vector": "<string or null>",
    "post_mortem_actions": ["<string>"]
  },
  "raw_content_excerpt": "<key paragraphs or tweet text>",
  "media_urls": ["<image URLs if any>"],
  "access_issues": "none | blocked | 404 | partial"
}
```

### Autonomous Enrichment (Phase 2) — per-search subagent return

```json
{
  "search_query": "<query used>",
  "discovered_urls": ["<URL>"],
  "source_type_per_url": {"<URL>": "<type>"},
  "entities": { "/* same structure as Phase 1 */": "..." },
  "raw_content_excerpt": "<key findings>"
}
```

### On-Chain Verification (Phase 4) — per-item subagent return

Per tx hash:
```json
{
  "tx_hash": "0x...",
  "blockchain": "<chain>",
  "found": true,
  "from": "0x...",
  "to": "0x...",
  "status": "0x1",
  "block_number": "0x...",
  "block_timestamp": "<ISO8601>",
  "input_selector": "0x...",
  "decoded_function": "<string or null>",
  "value": "0x...",
  "discrepancies": ["<description of any mismatch>"],
  "verification_status": "verified | discrepancy | not_found"
}
```

Per address:
```json
{
  "address": "0x...",
  "blockchain": "<chain>",
  "code_len_latest": 0,
  "code_len_attack_block": 0,
  "account_type": "EOA | Contract | Multisig | Exchange | Unknown",
  "is_eip7702": false,
  "delegated_address": "<0x... or null>",
  "tx_count": 0,
  "discrepancies": ["<description>"],
  "verification_status": "verified | discrepancy | not_found"
}
```

## Result Aggregation

The main agent collects all subagent returns and performs:

1. **Entity deduplication**: Merge identical addresses/tx hashes reported by multiple subagents
2. **Conflict resolution**: When subagents disagree (e.g., different loss figures), apply Cross-Source Reconciliation rules from `references/data-sources.md` — on-chain data is ground truth
3. **Completeness check**: Verify no critical entity is missing after aggregation
4. **Provenance tracking**: Record which subagent (and thus which source URL) contributed each entity

## RPC Endpoint Distribution (Phase 4)

When multiple verification subagents run in parallel and all need RPC access:

1. **Assign each subagent a different primary endpoint** from the rotation list:
   - Ethereum: `1rpc.io/eth`, `rpc.ankr.com/eth`, `eth.drpc.org`, `ethereum-rpc.publicnode.com`
   - BSC: `bsc-dataseed.binance.org`, `bsc-dataseed1.binance.org`, `bsc.publicnode.com`
2. **Each subagent falls back** to other endpoints if its assigned endpoint fails
3. **Add `time.sleep(1)`** between RPC calls within each subagent (see Pitfall #19 in `pitfalls/general.md`)
4. **Monitor for `None` returns** (see Pitfall #20) — retry from a different endpoint

This prevents N parallel agents from hammering the same endpoint and triggering 429 rate limits.

### Endpoint Assignment Example

```python
# Distribute endpoints across parallel subagents
eth_endpoints = [
    "https://1rpc.io/eth",
    "https://rpc.ankr.com/eth",
    "https://eth.drpc.org",
    "https://ethereum-rpc.publicnode.com",
]

def assign_endpoint(subagent_index, chain="eth"):
    endpoints = eth_endpoints if chain == "eth" else bsc_endpoints
    return endpoints[subagent_index % len(endpoints)]
```

## Error Handling

- **Single subagent failure does not block the batch**: If one subagent returns an error or empty result, the main agent records the failure and continues with the other results
- **Retry failed items**: After the batch completes, the main agent can re-dispatch failed items as a new single-item batch
- **Partial results are valid**: If 14 of 15 URLs were successfully processed, proceed with 14 results — the 15th can be retried or noted as inaccessible

## Dispatch Prompt Template

When dispatching a parallel subagent, the goal/context should include:

1. **The specific item to process** (one URL, one tx hash, one address, one search query)
2. **The required return format** (reference the structured return format above)
3. **Relevant constraints** from the skill (e.g., "use vxtwitter API for X URLs", "check eth_getCode at both latest and attack block")
4. **The RPC endpoint to use** (for Phase 4 verification subagents)
5. **Instruction to return ONLY the structured JSON** — no prose

### Example: Phase 1 URL Extraction Subagent Goal

```
Extract all on-chain entities from this URL: <URL>

Rules:
- For X/Twitter URLs, use vxtwitter API (https://api.vxtwitter.com/Twitter/status/<TWEET_ID>)
- For long threads, fall back to browser_navigate + browser_console(expression="document.body.innerText")
- Use vision_analyze on any mediaURLs to extract on-chain data from images
- If URL returns 404, report access_issues: "404"

Return ONLY this JSON structure:
{ "source_url": "...", "source_type": "...", "entities": { ... }, "raw_content_excerpt": "...", "media_urls": [...], "access_issues": "..." }
```

### Example: Phase 4 Tx Verification Subagent Goal

```
Verify this transaction on-chain: txHash=0x..., blockchain=ethereum

Use this RPC endpoint as primary: https://1rpc.io/eth
Fall back to: https://rpc.ankr.com/eth, https://eth.drpc.org

Steps:
1. eth_getTransactionByHash — verify from, to, status (0x1=SUCCESS), input selector
2. eth_getBlockByNumber — get block timestamp
3. Compare against the expected values from the draft

Add time.sleep(1) between RPC calls. If result is None/null, retry from a different endpoint.

Return ONLY this JSON structure:
{ "tx_hash": "0x...", "found": true, "from": "...", "to": "...", "status": "0x1", "block_number": "...", "block_timestamp": "...", "input_selector": "...", "decoded_function": "...", "discrepancies": [...], "verification_status": "verified" }
```
