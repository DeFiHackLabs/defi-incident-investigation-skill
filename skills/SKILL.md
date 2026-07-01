---
name: defi-incident-investigation
description: Investigate DeFi security incidents with multi-agent on-chain verification. Gathers intel, drafts schema-compliant JSON reports, verifies all claims against on-chain data, and runs adversarial validation before finalizing.
---

# DeFi Incident Investigation

## When to Use

- User asks to investigate a DeFi hack and produce a detailed report JSON
- User provides reference URLs (X posts, blog analyses, post-mortems) for an incident
- Need to verify an existing JSON report against on-chain data

## Output

A single JSON file conforming to `templates/schema.json`. File naming: `YYYYMMDD-ProtocolName.json`. See `templates/schema-guide.md` for field documentation.

## Subagent Roles

This skill uses platform-agnostic delegated-agent terminology:

- **research agent**: A delegated agent optimized for focused data extraction and factual verification. Cannot delegate further.
- **general agent**: A delegated agent that investigates broadly and may spawn focused research agents.
- **adversarial validator**: A delegated agent that acts as a skeptical reviewer, assuming all report claims are false until proven against on-chain data.

### Harness Mapping

The table below maps the platform-agnostic roles to concrete tools in each supported harness. If your harness is not listed, use whatever subagent/delegation mechanism it provides. If it has no delegation capability, run all phases sequentially in the main agent.

| Role | Hermes | Claude Code | OpenCode | Codex |
|------|--------|-------------|----------|-------|
| research agent | `delegate_task` with `role="leaf"` | `Task` tool with a focused prompt | Run in main agent (no built-in subagent) | `Task` tool with a focused prompt |
| general agent | `delegate_task` with `role="orchestrator"` | `Task` tool, then spawn more `Task` calls | Run in main agent | `Task` tool, then spawn more `Task` calls |
| adversarial validator | `delegate_task` with `role="leaf"` in a separate session | `Task` tool with a "prove this wrong" prompt | Run in main agent with explicit "assume all claims are false" instruction | `Task` tool with a "prove this wrong" prompt |

**When delegation is unavailable:** The main agent runs all phases sequentially itself. Phase 5 (Adversarial Validation) must still be performed — the agent re-reads the final JSON from disk and re-verifies all claims with a skeptical mindset, as if it were a different agent.

## Checkpoint System

Every phase writes a checkpoint JSON to `/tmp/defi-incident-<incidentId>/`. This enables resume after timeout, token limit, or interruption.

Checkpoint file format:
```json
{
  "phase": "intel|draft|verify|report|adversarial|finalize",
  "status": "in_progress|completed|failed",
  "input": "<what was provided to this phase>",
  "result": "<what this phase produced>",
  "output": "<artifacts written (file paths, tx hashes verified, etc.)>",
  "timestamp": "ISO8601"
}
```

Before starting any phase, check if a checkpoint exists. If found and status="completed", skip to next phase. If "in_progress" or "failed", review the partial result and resume from where it left off.

## Workflow

### Phase 1: Intel Gathering (general agent)

**Goal:** Extract all available information from user-provided reference URLs.

1. Read all provided URLs. For X/Twitter URLs, use the vxtwitter API (`https://api.vxtwitter.com/Twitter/status/<TWEET_ID>`) — it requires no auth and returns full tweet text, media URLs, threaded tweets, and quoted tweets. Use `vision_analyze` on `mediaURLs` to extract on-chain data from tweet images. For other URLs, use `browser_navigate` + `browser_console` for JavaScript-rendered pages. Extract full article content including code blocks, addresses, tx hashes.
2. If anti-crawler / Cloudflare blocks access: ask the user to manually visit the page and paste content. Do not attempt to bypass.
3. If API keys are needed (Etherscan, etc.): follow the **API Key Resolution Protocol** (see section below). Do not guess or fabricate keys.
4. Extract entities: protocol name, blockchain(s), attacker addresses, victim contracts, tx hashes, loss figures, attack timeline, attack method.
5. Cross-reference with external databases. See `references/data-sources.md`.
6. Write checkpoint: `phase=intel, status=completed, result=<extracted entities JSON>`.

### Phase 2: Schema Draft (main agent)

**Goal:** Produce a draft JSON report from intel.

1. Read `templates/schema.json` for field requirements and valid enum values.
2. Read `templates/schema-guide.md` for field constraints, extensible fields workflow, common compliance errors, and the validation snippet.
3. Map extracted entities to schema fields. For any vocabulary key that doesn't fit, add a new key to `schema.json` `_metadata` (see Extensible Fields in schema-guide.md).
4. Set `metadata.human_verified = false`.
5. Write draft JSON to `/tmp/defi-incident-<incidentId>/draft.json`.
6. Write checkpoint: `phase=draft, status=completed, result=<draft.json path>`.

### Phase 3: On-Chain Verification (research agent)

**Goal:** Verify every tx hash, address, amount, and timestamp against on-chain data.

1. Read `references/onchain-verification.md` for multi-chain RPC patterns and Etherscan V2 API usage.
2. Read `references/pitfalls/general.md` for known verification traps.
3. **Important:** Cache all API responses to disk during Phase 1. Reuse cached data here instead of re-querying. If the API key gets rate-limited (see Pitfall #16), switch to public RPC (e.g., `https://1rpc.io/eth` for Ethereum). Add `time.sleep(1)` between RPC calls to avoid 429 (see Pitfall #19).
4. For every `txHash` in the draft:
   - Call `eth_getTransactionByHash` (or equivalent for the chain) — verify tx exists, `from`/`to` match, `status == 0x1` (SUCCESS)
   - Call `eth_getBlockByNumber` — verify block timestamp matches `attackTime`
   - Decode `input` selector — verify it matches the claimed function
   - For mint/transfer: decode amount and verify against report
5. For every `address` in `attackers[]` and `victims[]`:
   - Call `eth_getCode` at BOTH `latest` AND `hex(attackBlock)` — verify `accountType` (EOA vs Contract vs EIP-7702). The accountType should reflect the state at the attack block, not at `latest` (see Pitfall #17).
6. For `estimatedLoss`: sum on-chain amounts by decoding Transfer events from the drain tx receipt, verify total matches report
7. Record all discrepancies. Fix the draft JSON accordingly.
8. Write checkpoint: `phase=verify, status=completed, result=<verified draft.json path, discrepancies list>`.

### Phase 4: Report Generation (main agent)

**Goal:** Produce the final JSON report file.

1. Read the verified draft from Phase 3 output.
2. Write the final JSON to the user's requested output path (or `/tmp/defi-incident-<incidentId>/report.json` if no path specified).
3. Validate against schema: run `python3 -c "import json; json.load(open('<path>'))"` to confirm valid JSON. Check all required fields present. Check all enum values valid.
4. Write checkpoint: `phase=report, status=completed, result=<final report path>`.

### Phase 5: Adversarial Validation (adversarial validator agent)

**Goal:** Independently re-verify the FINAL JSON against on-chain data, assuming all claims are false until proven.

This is a SECOND verification pass, independent of Phase 3. The adversarial validator must:

1. **Re-read the final JSON from disk** (not from memory).
2. **For every `txHash`**: re-verify on-chain. If tx NOT FOUND → CRITICAL (fabricated hash). If `status == 0x0` → CRITICAL (failed tx included as success). If selector doesn't match description → CRITICAL.
3. **For every `address`**: re-verify `accountType` via `eth_getCode`. Mismatch → CRITICAL.
4. **For aggregate claims**: re-sum on-chain amounts. Off by >1% → CRITICAL.
5. **For `attackTime`**: re-verify from block timestamps. Off by >1 min → HIGH.
6. **Logical consistency**: check that `description`, `rootCause`, and `attackVector` do not contradict each other or the on-chain evidence. Inconsistency → HIGH.
7. **Source figures**: if a source figure was adopted without on-chain cross-check → HIGH.
8. Produce a verdict: PASS / FAIL with discrepancy list (severity: Critical / High / Medium).

Discrepancy severity:
- **Critical**: fabricated address, wrong accountType, wrong tx hash, wrong selector, amount off by >1% → must fix
- **High**: timestamp off by >1 min, source figure adopted without cross-check, logical inconsistency → fix or annotate
- **Medium**: minor rounding → fix if easy, otherwise annotate

If verdict is FAIL: return to Phase 4, fix all Critical and High discrepancies, re-run Phase 5.

Write checkpoint: `phase=adversarial, status=completed, result=<verdict + discrepancy list>`.

### Phase 6: Finalize (main agent)

1. Present the final report path and adversarial validation verdict to the user.
2. Wait for user confirmation ("apply it" / "flip it" / "looks good").
3. Only after user confirmation: set `metadata.human_verified = true` and update `metadata.lastUpdated`.
4. Write checkpoint: `phase=finalize, status=completed`.

## API Key Resolution Protocol (Cross-Harness)

This protocol ensures consistent API key handling across all agent harnesses (Hermes, OpenCode, Claude Code, etc.). It is written as a behavioral specification — the agent follows these steps using whatever user-interaction mechanism its harness provides.

### Service Registry

| Service | Key Variable | Where to Look | Required For |
|---------|-------------|---------------|-------------|
| Etherscan V2 | `ETHERSCAN_API_KEY` | `~/.zshenv`, `~/.zshrc`, `~/.env`, env vars | ETH/BSC txlist, tokentx, contract source |
| BSCScan | Same as Etherscan | Same | BSC-specific queries |
| Blockscout | None | N/A | Fallback for Etherscan — no key needed |
| Public RPC | None | N/A | eth_getTransactionByHash, getCode, getBlockByNumber |
| X/Twitter API | `XURL_*` in `~/.xurl` | `~/.xurl` (YAML) | xurl CLI (optional — vxtwitter is keyless) |
| CoinGecko | None | N/A | Historical ETH/token prices |

### Resolution Procedure (for each service the investigation needs)

Execute these steps IN ORDER. Stop at the first step that succeeds:

1. **Check environment variables.** Run `env | grep -i <KEY_VAR>` or read `~/.zshenv` / `~/.zshrc` / `~/.env` / `~/.bashrc` for the key. If found, use it.

2. **Ask the user (Human-in-the-Loop).** If the key is not found, stop and ask the user:
   - State which service needs the key and why (e.g., "I need an Etherscan API key to fetch the attacker's transaction list — this data is not available via public RPC.")
   - State what the key will be used for specifically.
   - Ask the user to provide the key or to set it in a file.
   - WAIT for the user's response before proceeding. Do not skip this step or guess a key.

3. **If the user provides the key:** Use it immediately. Do not write it to any file unless the user explicitly asks.

4. **If the user cannot provide the key, or does not respond:** Fall back to the alternative path for that service (see Fallback Paths below). Continue the investigation. Do not block the entire investigation on a single missing key.

5. **If the key is found but rate-limited / blocked:** Switch to the fallback path immediately. Do not retry the blocked key. Note the issue in the checkpoint.

### Fallback Paths (when a key is missing or blocked)

| Service | Primary Path (with key) | Fallback Path (no key) |
|---------|------------------------|----------------------|
| Etherscan (ETH) | `api.etherscan.io/v2/api` | Public RPC: `https://1rpc.io/eth` for all JSON-RPC methods. Blockscout: `https://eth.blockscout.com/api/v2/` for tx lists, token transfers, contract source. |
| Etherscan (BSC) | Same key, `chainid=56` (paid) | Public BSC RPC: `https://bsc-dataseed.binance.org/` for JSON-RPC. Blockscout: `https://bsc.blockscout.com/api/v2/`. |
| X/Twitter | xurl CLI with OAuth | vxtwitter API: `https://api.vxtwitter.com/Twitter/status/<id>` (keyless, returns full tweet text + media). Browser navigation for threads. |
| CoinGecko | Free tier (10 calls/min) | Hardcode approximate price from source tweets, note as "estimated" in report. |

### What NOT to Do

- Do NOT fabricate or guess API keys.
- Do NOT silently skip a data source because the key is missing — ask first, fall back second.
- Do NOT write API keys into report files, checkpoints, or any output that may be shared.
- Do NOT block the investigation entirely if a key is missing — always have a fallback path.
- Do NOT store user-provided keys in memory longer than needed for the current investigation.

## Reference Files

| File | Covers |
|------|--------|
| `templates/schema-guide.md` | Field constraints, extensible fields workflow, common compliance errors, validation snippet |
| `references/onchain-verification.md` | Multi-chain RPC endpoints, Etherscan V2 API, EOA vs Contract detection, input decoding |
| `references/pitfalls/general.md` | All known T2 verification traps (19 pitfalls from real investigations) |
| `references/attack-patterns.md` | Attack pattern quick reference (key compromise, bridge, non-EVM, EIP-7702, MEV phishing, flash loan) |
| `references/data-sources.md` | External databases (DeFiLlama, SlowMist, Etherscan, X/Twitter) + cross-source reconciliation table |
| `references/pitfalls/README.md` | Template for adding case-specific pitfalls after an investigation |

## Case-Specific Pitfalls

Per-case pitfalls extracted from past investigations. Each file documents what went wrong, how it was detected, and the fix. After completing an investigation, add a pitfalls file to `references/pitfalls/` following the template in `references/pitfalls/README.md`.

Read relevant case files before investigating a similar attack pattern.
