# Data Sources & Cross-Source Reconciliation

## External Databases

### DeFiLlama Hacks API
**Endpoint:** `https://api.llama.fi/hacks`
- Returns JSON array. Fields: `date` (Unix timestamp), `name`, `classification`, `technique`, `amount` (integer USD), `chain` (array), `bridgeHack`, `targetType`, `source`, `returnedFunds`, `language`.
- Filter by protocol name: `curl -s 'https://api.llama.fi/hacks' | python3 -c "import sys,json; [print(json.dumps(i,indent=2)) for i in json.load(sys.stdin) if 'protocol' in json.dumps(i).lower()]"`
- `date` is Unix seconds: `datetime.fromtimestamp(ts, tz=timezone.utc)`
- `amount` may be rounded

### SlowMist Hacked Database
**URL:** `https://hacked.slowmist.io/`
- JS-rendered, use `browser_navigate` to access
- Search by protocol name
- Click "View Reference Sources" for SlowMist X/Twitter thread links
- X threads often contain images with loss breakdown tables, token lists, tx details

### Etherscan V2 API
**Key:** `ETHERSCAN_API_KEY` in env files — see **API Key Resolution Protocol** in SKILL.md
**Endpoint:** `https://api.etherscan.io/v2/api?chainid=1&module=...&apikey=KEY`
- Same key works for Etherscan + BSCScan, but BSC V2 requires paid plan
- Free key: 5 calls/second rate limit
- If key is missing or rate-limited, use Blockscout or public RPC (see Fallback Paths in SKILL.md)
- See `references/onchain-verification.md` for full query reference

### Blockscout API (No Key Required)
**URL:** `https://<chain>.blockscout.com/api/v2/addresses/{addr}/transactions`
- Rich JSON: hash, block_number, timestamp, from, to, value, method, decoded_input, token_transfers
- No API key, no rate limit issues
- Primary fallback when Etherscan API key fails
- Also: `/api/v2/addresses/{addr}/token-transfers` for ERC-20 transfers
- Also: `/api/v2/smart-contracts/{addr}` for source code + ABI

### X/Twitter Data Extraction

**vxtwitter API (no auth required, reliable):**
```
curl -sL "https://api.vxtwitter.com/Twitter/status/<TWEET_ID>" -o tweet.json
```
Returns JSON with: `text`, `user_name`, `user_screen_name`, `date`, `date_epoch`, `likes`, `retweets`, `mediaURLs` (image URLs), `qrt` (quoted tweet object), `replies` (count). Works for tweets that are public. Does NOT require X API credentials.

**Image extraction from tweets:** `mediaURLs` contains direct `pbs.twimg.com` URLs. Use `vision_analyze` with these URLs to extract on-chain data (addresses, tx hashes, amounts) from screenshot images in security firm tweets.

- Use `browser_navigate` + `browser_console(expression="document.body.innerText")` for full thread content when vxtwitter text is truncated or when reading reply threads
- `cdn.syndication.twimg.com/tweet-result` truncates long threads at ~280 chars — use browser for full text
- EN and ZH security firm threads often differ in detail level — cite both as separate sources
- ZH versions frequently contain more root-cause detail
- GoPlus ZH (`@GoPlusZH`) threads often link Etherscan card previews with attacker contract addresses embedded in the URL — extract these from the browser snapshot

## Source Accessibility Tiers

### Tier 1 — Always Accessible
- DeFiLlama API, SlowMist (browser), public RPC, Etherscan API, Blockscout API, Wikipedia, Telegram public preview (`https://t.me/s/<channel>`), Mirror.xyz blogs

### Tier 2 — Intermittently Accessible
- Wayback Machine (`web.archive.org/cdx/search/cdx?url=...`), DOJ/FBI press releases, CNBC

### Tier 3 — Blocked (CAPTCHA/Cloudflare)
- BleepingComputer, Reuters (DataDome), archive.ph, Google Search

### When Blocked
1. Check Wayback Machine: `curl -s 'https://web.archive.org/cdx/search/cdx?url=<URL>&output=json&limit=3'`
2. Search for quoted paragraphs elsewhere
3. Verify via on-chain data instead
4. Mark field as `[uncertain]` — `human_verified: false` exists for this

### When Anti-Crawler/Cloudflare Blocks Access
Ask the user to manually visit the page and paste content. Do not attempt to bypass without permission. If an API key would unblock access (e.g., a paid news API), follow the **API Key Resolution Protocol** in SKILL.md.

## Cross-Source Reconciliation

Multiple sources often disagree on numbers. Always verify on-chain and note discrepancies.

| Discrepancy Type | Example | Resolution |
|-----------------|---------|------------|
| Loss figure | GoPlus "$20M" vs QuillAudits "$36M" | Early estimates are partial; use most comprehensive figure and note discrepancy |
| Mint/burn count | QuillAudits "122B" vs on-chain 132.5B | Count all mint txs on-chain; sources may only report initial burst |
| Attack scope | Notion "300M H (3x100M)" vs actual 12 mints | Verify ALL transactions, not just ones in source article |
| Address classification | QuillAudits calls admin wallet "EOA" but it's EIP-7702 Contract | Run eth_getCode on every address |

**Rule:** When sources diverge, on-chain data is ground truth. Document the discrepancy explicitly in the report's `description` or `estimatedLoss.note`.

## Key Security Firm X Accounts

These accounts frequently post early alerts, loss figures, and detailed analysis threads during DeFi incidents. EN and ZH versions from the same firm often differ in detail level — cite both as separate sources when available.

- `@PeckShieldAlert` — early alerts, loss figures, tx hashes
- `@SlowMist_Team` — detailed analysis with images, often includes root cause diagrams
- `@BlockSecTeam` — BlockSec alerts, tx tracing
- `@Phalcon_xyz` - BlockSec analysis
- `@CertiKAlert` — CertiK security alerts
- `@CyversAlerts` — Cyvers real-time exploit detection
- `@BeosinAlert` — Beosin security alerts
- `@exvulsec` — ExVul security research
- `@GoPlusSecurity` - GoPlus English alerts
- `@GoPlusZH` — GoPlus Chinese alerts (often more root-cause detail than EN version)
- `@TenArmorAlert` — TenArmor security alerts
- `@DefimonAlerts` — DeFi monitor alerts
- `@QuillAudits_AI` — QuillAudits AI-powered post-mortem analyses
- `@RasterSec_com` — Raster Security alerts
- `@huntorlabs` — Huntor Labs security research
- `@Defi_Nerd_sec` — DeFi Nerd security analysis
- `@blockaid_` — Blockaid exploit detection system
- `@defillama_alerts` — DeFiLlama alerts


## GitHub Code Search

Rate-limited without auth. Repository file listing (`/repos/{owner}/{repo}/contents`) works better than code search. Useful for:
- Finding patch commits after an exploit
- Verifying protocol source code
- Protocol's audit reports
- Community PoC repos
