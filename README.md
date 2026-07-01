<div align="center">

# DeFi Security Incident Investigation Skill

### A coding-agent skill for multi-phase DeFi incident investigations with adversarially verified, on-chain-grounded reports

</div>

---

## Why This Skill Exists

LLM-generated incident reports share one fatal flaw: **hallucinated on-chain data**. Fake transaction hashes, wrong loss figures, fabricated addresses — all wrapped in confident prose that looks credible. This skill solves that by verifying every claim against blockchain RPC nodes twice — once during investigation, once by an independent adversarial agent that assumes the report is lying.

---

## Workflow Architecture

<!-- TODO: Insert workflow diagram here -->

---

## Key Features

- **Parallel agent delegation** — Phases 1 (intel gathering), 3 (on-chain verification), and 5 (adversarial validation) run as independent subagents with isolated context, preventing bias bleed-through from the main conversation. Works across Hermes, Claude Code, OpenCode, and Codex; falls back to sequential execution when delegation is unavailable.

- **Adversarial validation** — A second, independent agent re-reads the final JSON from disk (not from memory), assumes every claim is false, re-issues RPC calls for every tx hash / address / amount, and flags discrepancies by severity (Critical / High / Medium). If the verdict is FAIL, the pipeline loops back to fix and re-run.

- **Structured, schema-compliant output** — Reports are not free-text prose. They are strict JSON conforming to a formal schema with 19 required fields, extensible vocabularies (15 attack categories, 21 blockchains), and enum validation. Every tx hash, address, and amount is on-chain verified before the report is finalized.

- **Checkpoint & resume** — Long-running investigations can hit token limits, timeouts, or interruptions. Every phase writes a checkpoint JSON to `/tmp/defi-incident-<id>/`, capturing inputs, outputs, and status. Resume from the last completed phase — no lost work, no re-verification of already-confirmed data.

- **Self-improving pitfalls library** — Each completed investigation contributes case-specific pitfalls to `references/pitfalls/`. Every file records what went wrong, how it was detected, and the fix applied. 19 general pitfalls from real investigations are already documented, covering hallucinated tx hashes, EIP-7702 account type confusion, truncated terminal output, API rate-limiting, and more. Future investigations read these before starting, avoiding repeated mistakes.

- **Human-in-the-loop API key resolution** — When the skill needs an API key (Etherscan, BSCScan, etc.), it first checks environment variables, then asks the user to provide one. If the user cannot or does not respond, it automatically falls back to keyless alternatives (public RPC, Blockscout, vxtwitter). The investigation never blocks entirely on a missing key.

---

## Installation

```bash
# Install globally for all agents
npx skills add <your-org>/defi-incident-investigation --all -g

# Install for a specific agent
npx skills add <your-org>/defi-incident-investigation --agent claude-code -g
npx skills add <your-org>/defi-incident-investigation --agent opencode -g
npx skills add <your-org>/defi-incident-investigation --agent codex -g
npx skills add <your-org>/defi-incident-investigation --agent hermes -g

# List available skills without installing
npx skills add <your-org>/defi-incident-investigation -l
```

The skill is pure markdown — no build step, no dependencies, no compilation.

---

## Usage (Example)

<!-- TODO: Insert usage examples here -->
