# DeFi Incident Report Schema

Output JSON must conform to `schema.json` in this directory.

## File Naming

`YYYYMMDD-ProtocolName.json` — date is the incident date. ProtocolName is UpperCamelCase.

## Required Fields

All 19 fields are required. `additionalProperties: false` at top level — no extra fields allowed.

| Field | Type | Constraint |
|-------|------|-----------|
| id | string | `^[a-zA-Z0-9]+(-[a-zA-Z0-9]+)*-[0-9]{8}$` — e.g., `HumanityProtocol-20260608` |
| date | string | `^\d{4}-\d{2}-\d{2}$` (YYYY-MM-DD) |
| title | string | Human-readable event title |
| protocol | string | Affected protocol name |
| blockchain | string[] | Valid IDs from `_metadata.blockchains` |
| category | string[] | Valid IDs from `_metadata.categories` |
| ecosystem | string | Single valid ID from `_metadata.ecosystems` |
| language | string[] | Valid IDs from `_metadata.languages` |
| estimatedLoss | object | `{ totalUSD: string, breakdown: [{ asset, amount, convertedUSD, note? }], note? }` |
| attackTime | object | `{ startTime: ISO8601, endTime: ISO8601, date: YYYY-MM-DD, isRange?: bool, note? }` |
| description | string | Detailed incident description |
| transactions | array | `[{ txHash, blockchain, role, description? }]` |
| attackers | array | `[{ address, blockchain, role, accountType?, description? }]` |
| victims | array | `[{ address, blockchain, role, accountType?, description? }]` |
| rootCause | string | Root cause of the exploit |
| attackVector | string | How the attack was executed |
| lessons | string[] | Lessons learned |
| references | string[] | Accessible URLs (not relative paths) |
| metadata | object | `{ human_verified: bool, dateAdded?, lastUpdated?, models? }` |

## Vocabularies

All vocabularies are defined in `schema.json` under `_metadata`. Values must match an `id` from the corresponding list. All enum IDs are lowercase with underscores. Case-sensitive.

- **categories**: access_control, bridge, business_logic, flashloan, forged_message, governance_attack, key_compromise, mev_front_running, mev_honeypot, offchain_backend_exploit, price_manipulation, reentrancy, rug_pull, signature_abuse, social_engineering
- **blockchains**: arbitrum, avalanche, base, bitcoin, bitcoin_cash, bsc, cosmos, dogecoin, ethereum, fantom, hyperevm, litecoin, megaeth, monad, monero, optimism, polygon, solana, sui, tron, xrp
- **ecosystems**: bitcoin, cosmos, evm, move, polkadot, solana, substrate, ton
- **languages**: cairo, go, huff, move, rust, solidity, vyper, yul, N/A
- **attackerRoles**: consolidation, exploit_contract, gas_funder, primary_attacker
- **victimRoles**: liquidity_pool, protocol, protocol_contract, treasury, user, vulnerable_contract
- **transactionRoles**: consolidation, exploit, funding, laundering, setup
- **accountTypes**: Contract, EOA, Exchange, Multisig, Unknown

### Enum Validation Mapping

- `blockchain[]` → `_metadata.blockchains[].id`
- `category[]` → `_metadata.categories[].id`
- `ecosystem` → `_metadata.ecosystems[].id`
- `language[]` → `_metadata.languages[].id`
- `attackers[].role` → `_metadata.attackerRoles[].id`
- `victims[].role` → `_metadata.victimRoles[].id`
- `transactions[].role` → `_metadata.transactionRoles[].id`
- `accountType` → `_metadata.accountTypes[].id` (enum: EOA, Contract, Multisig, Exchange, Unknown)

## Extensible Fields

When no existing vocabulary key matches the event, you MAY add a new key to: `languages`, `ecosystems`, `categories`, `blockchains`, `attackerRoles`, `victimRoles`, `transactionRoles`. Add the new entry to `schema.json` `_metadata.<vocabulary>`:

```json
{ "id": "new_id", "name": "Human Readable Name" }
```

`accountTypes` is NOT extensible — use `Unknown` if none fits.

### Workflow for Adding a New Category/Tag

When you discover that no existing category accurately describes the attack pattern:

1. **Identify the gap:** Articulate WHY no existing tag fits (e.g., "mev_front_running implies the attack USES front-running, but here the MEV bot was the VICTIM").
2. **Choose a tag name:** Use snake_case. Prefer terms from authoritative sources (academic papers, security firm reports) over invented names.
3. **Add to schema.json:** Add `{"id": "new_id", "name": "Human Name", "description": "..."}` to `_metadata.categories`.
4. **Sync this file:** Update the Vocabularies section above to list the new tag.
5. **Update attack-patterns.md:** If the new category has an attack pattern entry, update the Schema mapping section to use the new tag.
6. **Ask the user:** Present the proposed tag and rationale. Do not silently add categories without user awareness.

**Example:** `mev_honeypot` was added when no existing category described "attacker deploys fake tokens/pools to lure automated MEV bots." `mev_front_running` was incorrect because it describes attacks that USE front-running, not attacks ON front-running bots. `mev_honeypot` matches GoPlus's "MEV蜜罐攻击" terminology and the Sen Yang et al. (CCS 2026) academic framework.

## metadata Object

`additionalProperties: false` — only `human_verified`, `dateAdded`, `lastUpdated`, `models` allowed.

**Cannot add custom tracking fields** (e.g., `t2_findings`, `verified_by`, `source_notes`) to `metadata` or top level without extending the schema. Instead:
1. Inline T2 findings into `description` as a final paragraph
2. Use `references[]` for URL-only sources
3. Extend schema if the field will be reused across many reports

### metadata.human_verified

Set to `false` until adversarial validation passes AND the user explicitly confirms. Never auto-flip to `true`.

## Common Compliance Errors

| Error | Example | Fix |
|-------|---------|-----|
| Invalid enum | `"access"` instead of `"access_control"` | Use exact ID from `_metadata` |
| Invalid blockchain | `"Multi-chain"` | List individual chain IDs |
| ID pattern violation | Hyphens in wrong place | `^[a-zA-Z0-9]+(-[a-zA-Z0-9]+)*-[0-9]{8}$` |
| Missing comma after long string | `JSONDecodeError: Expecting ','` | Run `python3 -m json.tool` after writing |
| Trailing comma | `JSONDecodeError` before `}` or `]` | Remove — JSON doesn't allow trailing commas |
| Unescaped newline in string | `JSONDecodeError: Invalid control character` | Use `\\n` not literal newline |
| Placeholder tx hash | `txHash` starting with `TBD-` | Remove or replace with verified hash |
| Relative URL in references | `/?event=...` | Use absolute accessible URL |
| `date` != `attackTime.date` | Inconsistent dates | Must match |
| `sum(breakdown[].convertedUSD)` != `totalUSD` | Rounding mismatch | Adjust or document in `estimatedLoss.note` |

## Validation Snippet

```python
import json, re

with open('templates/schema.json') as f: schema = json.load(f)
with open('<report>.json') as f: data = json.load(f)

errors = []
for field in schema['required']:
    if field not in data: errors.append(f'Missing: {field}')
if not re.match(schema['properties']['id']['pattern'], data['id']):
    errors.append(f'ID pattern: {data["id"]}')
if data.get('date') != data.get('attackTime', {}).get('date'):
    errors.append('date != attackTime.date')
for field, key in [('blockchain','blockchains'),('category','categories'),
                    ('ecosystem','ecosystems'),('language','languages')]:
    valid = {x['id'] for x in schema['_metadata'][key]}
    vals = data.get(field, [])
    vals = vals if isinstance(vals, list) else [vals]
    for v in vals:
        if v not in valid: errors.append(f'Invalid {field}: {v}')
for coll in ['attackers', 'victims']:
    for i, e in enumerate(data.get(coll, [])):
        at = e.get('accountType', '')
        if at and at not in ['EOA','Contract','Multisig','Exchange','Unknown']:
            errors.append(f'{coll}[{i}] bad accountType: {at}')
if errors:
    for e in errors: print(f'  ERROR: {e}')
    exit(1)
print('VALIDATION PASSED')
```
